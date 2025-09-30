# io_uring Internals: How It Achieves Low Overhead

## Overview

This document explains the technical mechanisms behind io_uring's exceptional performance: how it minimizes syscalls, reduces CPU usage, and how polling modes work at a low level.

## Table of Contents

1. [Why Syscalls Are Expensive](#why-syscalls-are-expensive)
2. [How io_uring Reduces Syscalls](#how-io_uring-reduces-syscalls)
3. [Shared Ring Buffer Architecture](#shared-ring-buffer-architecture)
4. [Why CPU Usage Is Lower](#why-cpu-usage-is-lower)
5. [Polling Modes Deep Dive](#polling-modes-deep-dive)
6. [Performance Analysis](#performance-analysis)

---

## Why Syscalls Are Expensive

### The Cost of a Syscall

Every syscall involves significant overhead:

```
Userspace:
    Application running
    ↓
1. Save registers to stack
2. Load syscall number into register
3. Execute syscall instruction (int 0x80 or syscall)
    ↓
Kernel Space:
4. Context switch (~200-500 CPU cycles)
    - Save user context (registers, stack pointer, instruction pointer)
    - Switch to kernel stack
    - Switch page tables (TLB flush!)
    - Check permissions
5. Execute syscall handler
6. Prepare return value
    ↓
Back to Userspace:
7. Context switch back (~200-500 cycles)
    - Restore user context
    - Switch page tables again (TLB flush!)
    - Return to userspace
8. Restore registers
```

**Typical cost**: 
- Fast path: ~300-500ns (modern CPU, warm cache)
- Slow path: ~1-2μs (cold cache, TLB miss)

**Why it's expensive**:
- **Context switches** (2× per syscall)
- **TLB flushes** - Translation Lookaside Buffer invalidation
- **Cache pollution** - Kernel code evicts your data from CPU cache
- **Speculation barriers** - CPU must clear speculative execution (Spectre/Meltdown mitigations)
- **Mode transition overhead** - CPU pipeline flush

### Traditional I/O: Syscall Per Operation

```c
// Read 100 files (traditional approach)
for (int i = 0; i < 100; i++) {
    read(fds[i], buffers[i], sizes[i]);  // 100 syscalls
}

// Cost: 100 × 500ns = 50μs just in syscall overhead
// Plus actual I/O time
```

### Linux AIO: Still High Syscall Count

```c
// Submit 100 I/Os with AIO
for (int i = 0; i < 100; i++) {
    io_submit(ctx, 1, &iocbs[i]);  // 100 syscalls
}

// Poll for completions
for (int i = 0; i < 100; i++) {
    io_getevents(ctx, 1, 1, &events[i], NULL);  // 100 syscalls
}

// Total: 200 syscalls!
// Cost: 200 × 500ns = 100μs in syscall overhead alone
```

---

## How io_uring Reduces Syscalls

### Key Innovation: Shared Memory Ring Buffers

io_uring uses **memory-mapped ring buffers** shared between userspace and kernel:

```
Userspace Process          Kernel
┌──────────────┐          ┌──────────────┐
│              │          │              │
│  Your Code   │          │   Kernel     │
│              │          │   Worker     │
└──────┬───────┘          └──────▲───────┘
       │                         │
       │    Shared Memory        │
       │    (mmap'd)            │
       ▼                         ▼
┌────────────────────────────────────┐
│  Submission Queue (SQ)             │
│  [Entry 0][Entry 1][Entry 2]...   │
└────────────────────────────────────┘
       │
       │
       ▼
┌────────────────────────────────────┐
│  Completion Queue (CQ)             │
│  [Entry 0][Entry 1][Entry 2]...   │
└────────────────────────────────────┘
```

**Critical insight**: Both userspace and kernel can read/write these buffers **without syscalls**!

### The Setup Phase (One-Time Cost)

```c
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);
```

**What happens internally**:

```c
// Kernel side (simplified)
io_uring_setup() {
    // 1. Allocate ring buffers
    sq_ring = alloc_pages(SQ_SIZE);
    cq_ring = alloc_pages(CQ_SIZE);
    sqes = alloc_pages(SQES_SIZE);
    
    // 2. Map into userspace
    mmap(sq_ring, PROT_READ | PROT_WRITE);
    mmap(cq_ring, PROT_READ | PROT_WRITE);
    mmap(sqes, PROT_READ | PROT_WRITE);
    
    // 3. Return file descriptor for io_uring_enter()
    return ring_fd;
}
```

**Result**: Userspace and kernel now share memory. No syscalls needed to communicate!

### Submitting Operations (Often Zero Syscalls)

#### Step 1: Write to Submission Queue (No Syscall)

```c
// Get next available SQ entry
sqe = io_uring_get_sqe(&ring);
```

**Under the hood**:
```c
struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring) {
    struct io_uring_sq *sq = &ring->sq;
    unsigned head = *sq->khead;  // Read from shared memory
    unsigned next = sq->sqe_tail + 1;
    
    if (next - head <= *sq->kring_entries) {
        sqe = &sq->sqes[sq->sqe_tail & sq->ring_mask];
        sq->sqe_tail = next;
        return sqe;  // No syscall!
    }
    return NULL;  // Queue full
}
```

**Key point**: Just manipulating **local memory**. No kernel interaction!

#### Step 2: Fill in Operation Details (No Syscall)

```c
io_uring_prep_read(sqe, fd, buffer, size, offset);
```

**Under the hood**:
```c
void io_uring_prep_read(struct io_uring_sqe *sqe, int fd,
                        void *buf, unsigned nbytes, off_t offset) {
    sqe->opcode = IORING_OP_READ;
    sqe->fd = fd;
    sqe->addr = (unsigned long) buf;
    sqe->len = nbytes;
    sqe->off = offset;
    // All just memory writes - no syscall!
}
```

#### Step 3: Submit (Sometimes Zero Syscalls!)

```c
io_uring_submit(&ring);
```

**Three possible paths**:

##### Path A: SQPOLL Mode (Zero Syscalls)

```c
// If SQPOLL enabled, kernel thread is polling the SQ ring
// Just update the tail pointer in shared memory
io_uring_submit() {
    io_uring_smp_store_release(sq->ktail, sq->sqe_tail);
    // Kernel thread sees new tail, processes entries
    // No syscall needed!
    return 0;
}
```

**Timeline**:
```
T=0: Userspace writes sqe_tail (memory write, ~5ns)
T=1: Kernel polling thread sees new tail
T=2: Kernel starts processing I/O
Total userspace overhead: ~5ns!
```

##### Path B: Normal Mode (One Syscall for Batch)

```c
io_uring_submit() {
    // Update tail pointer
    io_uring_smp_store_release(sq->ktail, sq->sqe_tail);
    
    // Wake up kernel with syscall
    io_uring_enter(ring_fd, to_submit, 0, 0);
    // ↑ One syscall, but for ALL queued operations
}
```

**Cost**: 1 syscall for N operations (amortized to ~5ns per operation if N=100)

##### Path C: Kernel Already Awake (Zero Syscalls)

If kernel is still processing previous batch:
```c
io_uring_submit() {
    io_uring_smp_store_release(sq->ktail, sq->sqe_tail);
    
    // Check if kernel needs waking
    if (!kernel_needs_wakeup(ring)) {
        return 0;  // Kernel already working, no syscall!
    }
    
    io_uring_enter(ring_fd, to_submit, 0, 0);
}
```

### Getting Completions (Often Zero Syscalls)

#### Peek at Completion Queue (No Syscall)

```c
io_uring_peek_cqe(&ring, &cqe);
```

**Under the hood**:
```c
int io_uring_peek_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr) {
    struct io_uring_cq *cq = &ring->cq;
    unsigned head = *cq->khead;  // Read from shared memory
    unsigned tail = io_uring_smp_load_acquire(cq->ktail);
    
    if (head != tail) {
        *cqe_ptr = &cq->cqes[head & cq->ring_mask];
        return 0;  // Found completion, no syscall!
    }
    
    return -EAGAIN;  // Nothing ready, no syscall!
}
```

#### Wait for Completion (One Syscall Only If Needed)

```c
io_uring_wait_cqe(&ring, &cqe);
```

**Under the hood**:
```c
int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr) {
    // First try to peek (no syscall)
    if (io_uring_peek_cqe(ring, cqe_ptr) == 0) {
        return 0;  // Already ready!
    }
    
    // Nothing ready, need to wait (syscall)
    io_uring_enter(ring_fd, 0, 1, IORING_ENTER_GETEVENTS);
    
    // Try again after waking up
    return io_uring_peek_cqe(ring, cqe_ptr);
}
```

### Syscall Comparison

**Traditional I/O (100 operations)**:
```
Operation: read() × 100
Syscalls: 100
Cost: 100 × 500ns = 50μs
```

**Linux AIO (100 operations)**:
```
Operation: io_submit() × 100 + io_getevents() × 100
Syscalls: 200
Cost: 200 × 500ns = 100μs
```

**io_uring Normal Mode (100 operations)**:
```
Operation: Queue 100 operations + submit() + wait()
Syscalls: 2 (one submit, one wait)
Cost: 2 × 500ns = 1μs (50× better than traditional!)
```

**io_uring SQPOLL Mode (100 operations)**:
```
Operation: Queue 100 operations (kernel polls)
Syscalls: 0
Cost: ~500ns total (memory writes only, 100× better!)
```

---

## Shared Ring Buffer Architecture

### Memory Layout

```
io_uring File Descriptor
        ↓
┌──────────────────────────────────────┐
│ Submission Queue Ring                │
├──────────────────────────────────────┤
│ • khead (kernel read position)       │ ← Kernel writes, user reads
│ • ktail (kernel write position)      │ ← Kernel writes, user reads
│ • ring_mask (size - 1)               │
│ • ring_entries (queue size)          │
│ • flags                              │
│ • dropped (overflow counter)         │
│ • array (indices into sqes[])        │ ← User writes indices
└──────────────────────────────────────┘
        ↓
┌──────────────────────────────────────┐
│ Submission Queue Entries (SQEs)      │
├──────────────────────────────────────┤
│ SQE[0]: opcode=READ, fd=3, ...      │
│ SQE[1]: opcode=WRITE, fd=4, ...     │
│ SQE[2]: opcode=FSYNC, fd=5, ...     │
│ ...                                  │
└──────────────────────────────────────┘
        ↓
┌──────────────────────────────────────┐
│ Completion Queue Ring                │
├──────────────────────────────────────┤
│ • khead (kernel write position)      │ ← Kernel writes, user reads
│ • ktail (kernel read position)       │ ← User writes, kernel reads
│ • ring_mask                          │
│ • ring_entries                       │
│ • overflow                           │
│ • cqes (completion entries)          │
│   CQE[0]: res=4096, user_data=...   │ ← Kernel writes
│   CQE[1]: res=2048, user_data=...   │
│   ...                                │
└──────────────────────────────────────┘
```

### Producer-Consumer Pattern

#### Submission Queue (User → Kernel)

**User is producer**:
```c
// Add entry
sqe = &sq->sqes[sq->sqe_tail & sq->ring_mask];
fill_sqe(sqe, ...);  // Write operation details

// Update tail (makes visible to kernel)
sq->array[sq->sqe_tail & sq->ring_mask] = sq->sqe_tail;
io_uring_smp_store_release(&sq->ktail, sq->sqe_tail + 1);
```

**Kernel is consumer**:
```c
// Kernel worker thread
while (true) {
    head = sq->khead;
    tail = io_uring_smp_load_acquire(&sq->ktail);
    
    while (head != tail) {
        // Read entry
        sqe_idx = sq->array[head & sq->ring_mask];
        sqe = &sq->sqes[sqe_idx];
        
        // Process I/O
        process_sqe(sqe);
        
        head++;
    }
    
    // Update head
    io_uring_smp_store_release(&sq->khead, head);
}
```

#### Completion Queue (Kernel → User)

**Kernel is producer**:
```c
// After I/O completes
cqe = &cq->cqes[cq->head & cq->ring_mask];
cqe->user_data = sqe->user_data;
cqe->res = result;
cqe->flags = flags;

// Update head (makes visible to user)
io_uring_smp_store_release(&cq->khead, cq->head + 1);
```

**User is consumer**:
```c
// Check for completions
head = cq->khead;  // User's position
tail = io_uring_smp_load_acquire(&cq->ktail);  // Kernel's position

if (head != tail) {
    cqe = &cq->cqes[head & cq->ring_mask];
    process_completion(cqe);
    
    // Update tail (tells kernel we're done with this entry)
    cq->ktail = head + 1;
    io_uring_smp_store_release(&cq->ktail, cq->ktail);
}
```

### Lock-Free Synchronization

**Memory barriers** ensure correctness:

```c
// Store-release: All writes before this are visible to other threads
// before the store itself
io_uring_smp_store_release(ptr, value);

// Implementation (x86):
asm volatile("" ::: "memory");  // Compiler barrier
*ptr = value;
// x86 has strong memory model, no CPU barrier needed for release

// Load-acquire: All reads after this see writes that happened-before
// the store we're loading
value = io_uring_smp_load_acquire(ptr);

// Implementation (x86):
value = *ptr;
asm volatile("" ::: "memory");  // Compiler barrier
```

**Why this works without locks**:
- Single producer, single consumer per queue
- Memory barriers ensure ordering
- No contention → no need for atomic operations or locks

---

## Why CPU Usage Is Lower

### 1. Fewer Context Switches

**Traditional I/O**:
```
For each operation:
  User mode → Kernel mode → User mode
  = 2 context switches × 100 ops = 200 switches
```

**io_uring**:
```
For batch of 100 operations:
  User mode → Kernel mode → User mode
  = 2 context switches total (100× reduction!)
```

### 2. No TLB Thrashing

**TLB (Translation Lookaside Buffer)**: CPU cache for virtual→physical address translations

**Traditional I/O**:
```
Each syscall:
1. Switch to kernel page tables → TLB flush
2. Kernel accesses its memory
3. Switch back to user page tables → TLB flush
Result: Cold TLB, lots of page table walks (expensive!)
```

**io_uring**:
```
One syscall for batch:
1. Switch to kernel (TLB flush)
2. Kernel processes all 100 operations
3. Switch back (TLB flush)
Result: Only 2 TLB flushes for 100 operations
```

**TLB miss cost**: ~100-200 CPU cycles
**TLB flushes saved**: 198 (for 100 operations)
**CPU cycles saved**: ~20,000-40,000 cycles

### 3. Better CPU Cache Utilization

**Traditional I/O**:
```
read(fd1, buf1, size);  // Load kernel code/data into cache
                        // Evict user data
compute();              // User code/data back in cache
                        // Evict kernel data
read(fd2, buf2, size);  // Load kernel again...
                        // Cache pollution on every syscall!
```

**io_uring**:
```
// Queue all operations (just memory writes)
for (i = 0; i < 100; i++) {
    queue_read(fd[i], buf[i], size);  // User cache stays hot
}

// One syscall
io_uring_submit();  // Kernel code loaded once
                    // Processes all operations
                    
// Back to user
compute();  // User cache not polluted 100 times
```

**Result**: L1/L2/L3 caches stay hot for user code

### 4. Kernel Can Optimize Batches

When kernel sees multiple operations at once:

```c
// Kernel sees these all at once:
read(file1, offset=0,    size=4K);
read(file1, offset=4K,   size=4K);
read(file1, offset=8K,   size=4K);
read(file1, offset=12K,  size=4K);

// Kernel can optimize:
read(file1, offset=0, size=16K);  // Single large I/O!
// Or reorder for elevator scheduling
```

**Benefits**:
- Fewer I/O operations to disk
- Better disk scheduling
- More efficient DMA transfers

### 5. Kernel Worker Threads (Buffered I/O)

**Traditional blocking I/O**:
```
Your thread:
  Issue read() → Block → Wake up
  ↓ Wasted time waiting
```

**io_uring with buffered I/O**:
```
Your thread:                Kernel worker thread:
  Queue operations            (none)
  io_uring_submit()        →  Process operations
  Continue working!           (may block on I/O)
  Compute...                  Still working...
  Compute...                  Completes
  Check completions           
  
↑ Your thread never blocks!
```

Kernel uses internal thread pool for blocking operations, so your application threads stay runnable.

### CPU Usage Example

**Benchmark**: Process 1M random 4KB reads

**Traditional blocking I/O**:
```
Time: 10 seconds
CPU: 100% (blocked waiting, but counted as CPU time)
Actual work: ~60% (40% wasted in syscalls/context switches)
```

**io_uring**:
```
Time: 4 seconds
CPU: 60% (doing actual work)
Actual work: ~90% (10% in batched I/O handling)
```

**Result**: 
- 2.5× faster
- 40% less CPU
- More CPU available for other work

---

## Polling Modes Deep Dive

io_uring offers three polling modes, each with different trade-offs.

### Mode 1: Interrupt-Driven (Default)

**How it works**:

```
1. Submit I/O operation
2. Kernel submits to device driver
3. Application continues or waits
4. Device completes I/O
5. Device raises interrupt → CPU interrupt handler
6. Interrupt handler wakes up kernel thread
7. Kernel thread writes completion to CQ
8. Application woken up (if waiting)
```

**Timeline**:
```
T=0:    Submit I/O
T=1:    Application continues
        ...
T=100:  Disk completes
T=101:  Interrupt raised (hardware → CPU)
T=102:  Interrupt handler runs (~1μs overhead)
T=103:  Kernel processes completion
T=104:  Completion visible in CQ
```

**Pros**:
- Low CPU usage (application can sleep)
- Good for mixed workloads
- Application can do other work

**Cons**:
- Interrupt overhead (~1-2μs)
- Interrupt latency jitter
- Context switch if application was waiting

### Mode 2: IOPOLL (I/O Completion Polling)

**Enable**:
```c
struct io_uring_params params = {0};
params.flags = IORING_SETUP_IOPOLL;
io_uring_queue_init_params(256, &ring, &params);
```

**How it works**:

```
1. Submit I/O with O_DIRECT flag
2. Instead of waiting for interrupt, kernel polls device
3. Kernel continuously checks device status register
4. When complete, immediately processes completion
```

**Kernel code (simplified)**:
```c
// Interrupt-driven (normal)
submit_bio(bio);
bio->bi_end_io = completion_callback;  // Called when interrupt fires
return;

// IOPOLL mode
submit_bio(bio);
while (!bio_complete(bio)) {
    cpu_relax();  // Busy wait, checking device
}
process_completion(bio);
```

**Timeline**:
```
T=0:    Submit I/O
T=1:    Kernel starts polling
T=2:    Check device... not ready
T=3:    Check device... not ready
        ...
T=100:  Check device... READY!
T=100:  Process completion immediately (no interrupt)
T=101:  Completion visible in CQ
```

**Device polling loop**:
```c
// NVMe driver (simplified)
int nvme_poll(struct blk_mq_hw_ctx *hctx) {
    struct nvme_queue *nvmeq = hctx->driver_data;
    
    // Read completion queue entry from device
    u16 status = nvmeq->cqes[nvmeq->cq_head].status;
    
    if (!(status & 1)) {  // Phase bit check
        return 0;  // Not ready
    }
    
    // Process completion
    handle_completion(nvmeq);
    return 1;  // Found completion
}
```

**CPU usage**:
```
# Dedicated polling thread burns 100% of one CPU core
# But latency drops from ~100μs to ~30μs
```

**Pros**:
- Lower latency (no interrupt overhead)
- More deterministic (no interrupt jitter)
- Better for real-time applications

**Cons**:
- Burns CPU (100% on polling thread)
- Only works with O_DIRECT
- Wastes power

**When to use**:
- Ultra-low latency requirements (<50μs)
- Dedicated I/O threads
- Consistent high load (polling is wasteful when idle)

### Mode 3: SQPOLL (Submission Queue Polling)

**Enable**:
```c
struct io_uring_params params = {0};
params.flags = IORING_SETUP_SQPOLL;
params.sq_thread_idle = 2000;  // Idle timeout in ms
io_uring_queue_init_params(256, &ring, &params);
```

**How it works**:

```
1. Kernel creates dedicated thread
2. Thread continuously polls SQ ring for new entries
3. When found, processes immediately
4. No io_uring_enter() syscall needed!
```

**Kernel thread (simplified)**:
```c
// Kernel SQPOLL thread
void io_sq_thread(void *data) {
    struct io_ring_ctx *ctx = data;
    
    while (!kthread_should_stop()) {
        // Check if new submissions
        if (sq_tail_changed(ctx)) {
            // Process all new entries
            while (has_work(ctx)) {
                sqe = get_next_sqe(ctx);
                process_sqe(ctx, sqe);
            }
        } else {
            // Nothing to do
            if (idle_time > ctx->sq_thread_idle) {
                // Go to sleep to save CPU
                schedule_timeout(HZ);
            }
            cpu_relax();  // Brief pause
        }
    }
}
```

**Application code (zero syscalls!)**:
```c
// Queue operations (just memory writes)
for (int i = 0; i < 100; i++) {
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd[i], buf[i], size, offset[i]);
}

// Update tail pointer (memory write, kernel thread sees it)
io_uring_smp_store_release(&ring.sq.ktail, ring.sq.sqe_tail);

// NO SYSCALL! Kernel thread processes operations.
// Continue doing other work...
```

**Timeline**:
```
T=0:    Application writes 100 SQEs
T=1:    Application updates tail pointer
T=2:    Kernel thread sees new tail (was polling every ~100ns)
T=3:    Kernel starts processing operations
        Application continues working in parallel!
T=50:   Kernel completes all operations
T=51:   Application checks CQ, finds completions
```

**CPU usage**:
```
# Kernel thread: ~10-50% (depends on load)
# Application thread: 0% I/O overhead (no syscalls!)
```

**Idle behavior**:
```c
// After sq_thread_idle milliseconds of no work:
if (no_work_for(sq_thread_idle)) {
    // Stop polling, go to sleep
    set_current_state(TASK_INTERRUPTIBLE);
    schedule();
    // Application's next io_uring_enter() will wake us up
}
```

**Pros**:
- Zero syscalls for submission (fastest!)
- Application never blocks on I/O
- Good for sustained throughput

**Cons**:
- Kernel thread uses CPU even when idle (until timeout)
- Need to tune sq_thread_idle
- More complex setup

**When to use**:
- High sustained I/O throughput
- Minimize application thread overhead
- Backend services with consistent load

### Combining SQPOLL + IOPOLL

**Ultimate low-latency mode**:
```c
params.flags = IORING_SETUP_SQPOLL | IORING_SETUP_IOPOLL;
params.sq_thread_idle = 2000;
```

**Result**:
- Zero syscalls for submission (SQPOLL)
- No interrupt overhead for completion (IOPOLL)
- Lowest possible latency (~20-30μs)
- **Cost**: Burns 2 CPU cores (SQPOLL thread + IOPOLL thread)

**Use case**: Trading systems, real-time analytics where latency >> cost

### Polling Mode Comparison

| Mode | Syscalls | Latency | CPU Usage | Use Case |
|------|----------|---------|-----------|----------|
| Interrupt (default) | Low | ~100μs | Low (~10%) | General purpose |
| IOPOLL | Low | ~30μs | High (100%) | Ultra-low latency |
| SQPOLL | None! | ~50μs | Medium (~30%) | High throughput |
| SQPOLL+IOPOLL | None! | ~20μs | Very High (200%) | Trading, real-time |

---

## Performance Analysis

### Microbenchmark: Single 4KB Read

**Setup**: Random 4KB read from NVMe SSD

**Traditional blocking**:
```
Total: 150μs
  - Syscall overhead: 1μs
  - Context switch: 2μs  
  - I/O submission: 2μs
  - Disk latency: 100μs
  - Interrupt: 2μs
  - Context switch back: 2μs
  - Memory copy: 1μs
```

**io_uring (interrupt mode)**:
```
Total: 105μs
  - Queue operation: 0.01μs (memory write)
  - io_uring_submit: 0.5μs (syscall)
  - I/O submission: 1μs
  - Disk latency: 100μs
  - Interrupt: 2μs
  - io_uring_wait: 0.5μs (syscall)
  - Memory copy: 1μs
  
Savings: 45μs (30% faster)
```

**io_uring (IOPOLL mode)**:
```
Total: 32μs
  - Queue operation: 0.01μs
  - io_uring_submit: 0.5μs
  - I/O submission: 1μs
  - Disk latency: 30μs (NVMe parallelism)
  - Polling overhead: 0μs (no interrupt)
  - io_uring_wait: 0.5μs
  
Savings: 118μs (82% faster!)
```

### Throughput Benchmark: 10,000 Random Reads

**Traditional blocking I/O**:
```
Time: 1.5 seconds
Throughput: 6,666 IOPS
CPU: 1 core at 100%
Syscalls: 10,000
```

**io_uring (batch=100)**:
```
Time: 0.6 seconds
Throughput: 16,666 IOPS
CPU: 1 core at 60%
Syscalls: 200 (100× batches of 100)
```

**io_uring (SQPOLL, batch=100)**:
```
Time: 0.4 seconds
Throughput: 25,000 IOPS
CPU: 1 core at 50% (app) + 1 core at 30% (kernel)
Syscalls: 0
```

### Real-World: Database Query Workload

**Workload**: TPC-C like, random reads + some writes

**Traditional approach (thread pool)**:
```
Threads: 32
CPU: 32
