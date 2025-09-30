# io_uring: Modern Async I/O for Linux

## Overview

io_uring is a Linux kernel interface for asynchronous I/O operations, introduced in kernel 5.1 (May 2019). It was designed by Jens Axboe to address the fundamental limitations of Linux AIO and provide a unified, efficient async I/O interface for all operation types.

**Key Innovation**: Shared ring buffers between userspace and kernel eliminate most syscall overhead while providing true async behavior for all I/O types.

## Why io_uring Was Created

Linux AIO had critical flaws:
- Only worked async with O_DIRECT (buffered I/O would block)
- High syscall overhead
- Limited operation support
- Broken cancellation
- Poor batching

io_uring provides:
- ✅ True async for buffered I/O
- ✅ Minimal syscall overhead (often zero!)
- ✅ Support for 100+ operation types
- ✅ Efficient batching
- ✅ Operation chaining
- ✅ Zero-copy data transfers

## Architecture

### The Ring Buffers

io_uring uses two shared ring buffers (mapped into both kernel and userspace):

```
Userspace                    Kernel Space
    │                             │
    ├──────────────┬──────────────┤
    │              │              │
┌───▼────┐     ┌───▼────┐     ┌──▼─────┐
│  SQ    │     │  CQ    │     │ Kernel │
│ Ring   │────>│ Ring   │<────│Workers │
│(Submit)│     │(Complete)     │        │
└────────┘     └────────┘     └────────┘
    ↑              ↓              ↑
    └──────────────┴──────────────┘
        Shared memory (mmap)
```

**SQ (Submission Queue)**:
- Userspace writes I/O requests here
- Kernel reads and processes them
- No syscall needed to add entries!

**CQ (Completion Queue)**:
- Kernel writes completion events here
- Userspace reads results
- No syscall needed to check completions!

### How It Works

**Setup** (once):
```c
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);  // 1 syscall
```

**Submit operations** (often 0 syscalls):
```c
// Get submission queue entry (just memory write)
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);

// Prepare operation (just filling struct)
io_uring_prep_read(sqe, fd, buffer, size, offset);

// Submit (1 syscall for batch, or 0 if kernel polling)
io_uring_submit(&ring);
```

**Get completions** (often 0 syscalls):
```c
// Peek at completion (just memory read)
struct io_uring_cqe *cqe;
io_uring_peek_cqe(&ring, &cqe);  // No syscall!

// Or wait if nothing ready (1 syscall)
io_uring_wait_cqe(&ring, &cqe);

// Mark as processed
io_uring_cqe_seen(&ring, cqe);
```

## Basic Example

### Simple File Read

```c
#include <liburing.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    
    // Initialize ring with 256 entries
    io_uring_queue_init(256, &ring, 0);
    
    // Open file (works with buffered I/O!)
    int fd = open("data.txt", O_RDONLY);
    
    // Allocate buffer
    char buffer[4096];
    
    // Get submission entry
    sqe = io_uring_get_sqe(&ring);
    
    // Prepare read operation
    io_uring_prep_read(sqe, fd, buffer, sizeof(buffer), 0);
    
    // Optional: attach user data for identification
    io_uring_sqe_set_data(sqe, (void*)0x1234);
    
    // Submit the operation
    io_uring_submit(&ring);
    
    // Wait for completion
    io_uring_wait_cqe(&ring, &cqe);
    
    // Check result
    if (cqe->res < 0) {
        fprintf(stderr, "Read failed: %s\n", strerror(-cqe->res));
    } else {
        printf("Read %d bytes\n", cqe->res);
    }
    
    // Mark completion as seen
    io_uring_cqe_seen(&ring, cqe);
    
    // Cleanup
    close(fd);
    io_uring_queue_exit(&ring);
    
    return 0;
}
```

**Compile**: `gcc -o example example.c -luring`

## Advanced Features

### 1. Batching Multiple Operations

Submit many operations with minimal syscalls:

```c
#define BATCH_SIZE 64

// Queue multiple reads
for (int i = 0; i < BATCH_SIZE; i++) {
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buffers[i], 4096, offsets[i]);
    io_uring_sqe_set_data(sqe, (void*)(long)i);
}

// Submit all at once (1 syscall for 64 operations!)
io_uring_submit(&ring);

// Process completions
for (int i = 0; i < BATCH_SIZE; i++) {
    io_uring_wait_cqe(&ring, &cqe);
    int idx = (int)(long)io_uring_cqe_get_data(cqe);
    printf("Operation %d completed with %d bytes\n", idx, cqe->res);
    io_uring_cqe_seen(&ring, cqe);
}
```

**Performance**: Amortizes syscall overhead across many operations.

### 2. Polling Mode (Ultra Low Latency)

Skip interrupts entirely for lowest latency:

```c
struct io_uring_params params = {0};
params.flags = IORING_SETUP_IOPOLL;

io_uring_queue_init_params(256, &ring, &params);

// Now kernel continuously polls for completions
// No interrupt overhead (~1-2μs saved per operation)
// Trade: Burns CPU, but latency drops from ~100μs to ~30μs
```

**Use case**: Ultra-low latency applications (trading systems, real-time analytics)

### 3. Kernel-Side Polling (SQPOLL)

Kernel thread monitors submission queue:

```c
struct io_uring_params params = {0};
params.flags = IORING_SETUP_SQPOLL;
params.sq_thread_idle = 2000;  // Idle after 2 seconds

io_uring_queue_init_params(256, &ring, &params);

// Now submissions don't even need io_uring_submit()!
// Just write to SQ ring, kernel thread picks it up
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buffer, size, offset);
// No submit call needed - kernel polls the ring!
```

**Performance**: Zero syscalls for submission!

### 4. Operation Chaining (Linked Operations)

Express dependencies between operations:

```c
// Read file, then write result to another file
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd_in, buffer, size, 0);
sqe->flags |= IOSQE_IO_LINK;  // Link to next operation

sqe = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe, fd_out, buffer, size, 0);
// This write only executes if read succeeds

io_uring_submit(&ring);
```

**Use case**: Complex workflows without manual orchestration.

### 5. Fixed Files (Pre-registered File Descriptors)

Register files once to avoid repeated fd lookups:

```c
int fds[10] = {fd1, fd2, fd3, ...};
io_uring_register_files(&ring, fds, 10);

// Use registered files (faster than regular fd)
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, 0, buffer, size, offset);  // 0 = first registered file
sqe->flags |= IOSQE_FIXED_FILE;
```

**Performance**: Skips fd table lookup on each operation.

### 6. Fixed Buffers (Pre-registered Memory)

Register buffers to enable zero-copy:

```c
struct iovec iovecs[BUFFER_COUNT];
// Setup iovecs pointing to your buffers...

io_uring_register_buffers(&ring, iovecs, BUFFER_COUNT);

// Use registered buffer (zero-copy from kernel to buffer)
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, NULL, size, offset, 0);  // 0 = buffer index
```

**Performance**: Eliminates memory copy from kernel to userspace.

### 7. Multishot Operations

Single submission, multiple completions:

```c
// Accept multiple connections with one submission
sqe = io_uring_get_sqe(&ring);
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);

// Each new connection generates a completion event
// No need to resubmit accept() after each connection
while (running) {
    io_uring_wait_cqe(&ring, &cqe);
    int client_fd = cqe->res;
    handle_client(client_fd);
    io_uring_cqe_seen(&ring, cqe);
}
```

**Use case**: Network servers, event-driven applications.

## Supported Operations

io_uring supports 100+ operation types. Key ones:

### File I/O
- `IORING_OP_READ` / `IORING_OP_WRITE`
- `IORING_OP_READV` / `IORING_OP_WRITEV` (vectored I/O)
- `IORING_OP_FSYNC` / `IORING_OP_FDATASYNC`
- `IORING_OP_FALLOCATE`
- `IORING_OP_FADVISE`

### File Management
- `IORING_OP_OPENAT` / `IORING_OP_OPENAT2`
- `IORING_OP_CLOSE`
- `IORING_OP_STATX`
- `IORING_OP_RENAME`
- `IORING_OP_UNLINK`

### Network I/O
- `IORING_OP_ACCEPT`
- `IORING_OP_CONNECT`
- `IORING_OP_SEND` / `IORING_OP_RECV`
- `IORING_OP_SENDMSG` / `IORING_OP_RECVMSG`
- `IORING_OP_SHUTDOWN`

### Advanced
- `IORING_OP_TIMEOUT` (timeouts for operations)
- `IORING_OP_POLL_ADD` (poll for readiness)
- `IORING_OP_PROVIDE_BUFFERS` (buffer management)
- `IORING_OP_NOP` (no-op for testing)

[Full list](https://kernel.dk/io_uring.pdf)

## Performance Comparison

### Benchmark: Random 4KB Reads on NVMe SSD

| Method                    | IOPS    | Latency (avg) | CPU Usage | Syscalls/op |
|---------------------------|---------|---------------|-----------|-------------|
| Blocking read()           | 100K    | 150μs         | 100%      | 1.0         |
| Linux AIO (O_DIRECT)      | 300K    | 100μs         | 100%      | 2.0         |
| io_uring (buffered)       | 800K    | 50μs          | 60%       | 0.02        |
| io_uring + IOPOLL         | 1.2M    | 30μs          | 100%      | 0.0         |
| io_uring + SQPOLL         | 900K    | 45μs          | 70%       | 0.0         |

**Key takeaways**:
- 3-8× better IOPS than traditional methods
- Much lower CPU usage (40% reduction)
- Near-zero syscall overhead with polling modes

### Syscall Overhead Reduction

**Traditional approach (1000 operations)**:
```
Blocking:   1000 read() = 1000 syscalls
Linux AIO:  1000 submit + 1000 getevents = 2000 syscalls
io_uring:   1 submit + 1 wait = 2 syscalls (500× reduction!)
```

## Use Cases

### Perfect For:
- **Databases** - High-throughput I/O with minimal overhead
- **Storage systems** - Object storage, distributed filesystems
- **Network servers** - Web servers, proxies, load balancers
- **Disk-based search** - ANN search, full-text search with large indexes
- **Data processing** - ETL pipelines, log aggregation
- **Game engines** - Asset streaming, texture loading

### Real-World Adoptions:
- **RocksDB** - Facebook's key-value store
- **ScyllaDB** - High-performance NoSQL database
- **QEMU** - Virtualization
- **Ceph** - Distributed storage
- **Rust async runtime** - Tokio-uring

## Best Practices

### 1. Choose Queue Depth Wisely

```c
// Too small: Not enough parallelism
io_uring_queue_init(8, &ring, 0);

// Too large: Memory waste, cache pollution
io_uring_queue_init(65536, &ring, 0);

// Good: Match to workload (typically 256-4096)
io_uring_queue_init(1024, &ring, 0);
```

### 2. Use Polling Modes Selectively

**IOPOLL**: Only for dedicated I/O threads with consistent load
**SQPOLL**: Good for sustained high throughput, wastes CPU when idle

### 3. Batch Operations

```c
// Bad: Submit each operation individually
for (int i = 0; i < 1000; i++) {
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, ...);
    io_uring_submit(&ring);  // 1000 syscalls!
}

// Good: Batch submissions
for (int i = 0; i < 1000; i++) {
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, ...);
}
io_uring_submit(&ring);  // 1 syscall!
```

### 4. Handle Errors Properly

```c
io_uring_wait_cqe(&ring, &cqe);

if (cqe->res < 0) {
    // res contains negative errno
    int error = -cqe->res;
    if (error == EAGAIN) {
        // Retry
    } else if (error == EIO) {
        // I/O error
    }
}
```

### 5. Track Operations with User Data

```c
struct my_request {
    int id;
    void *buffer;
    // ... other fields
};

struct my_request *req = malloc(sizeof(*req));
req->id = 123;

sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, req->buffer, size, offset);
io_uring_sqe_set_data(sqe, req);  // Attach your data

// Later in completion:
io_uring_wait_cqe(&ring, &cqe);
struct my_request *completed = io_uring_cqe_get_data(cqe);
printf("Request %d completed\n", completed->id);
free(completed);
```

## Integration with Disk-Based ANN Search

io_uring is ideal for disk-based approximate nearest neighbor search:

```c
// Prefetch graph nodes for ANN search
void prefetch_nodes(struct io_uring *ring, int fd, 
                    NodeID *nodes, int count) {
    char *buffers[count];
    
    // Queue all reads
    for (int i = 0; i < count; i++) {
        buffers[i] = malloc(NODE_SIZE);
        
        struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
        off_t offset = nodes[i] * NODE_SIZE;
        io_uring_prep_read(sqe, fd, buffers[i], NODE_SIZE, offset);
        io_uring_sqe_set_data(sqe, (void*)(long)i);
    }
    
    // Submit batch (1 syscall)
    io_uring_submit(ring);
    
    // Process completions
    for (int i = 0; i < count; i++) {
        struct io_uring_cqe *cqe;
        io_uring_wait_cqe(ring, &cqe);
        
        int idx = (int)(long)io_uring_cqe_get_data(cqe);
        if (cqe->res == NODE_SIZE) {
            cache_insert(nodes[idx], buffers[idx]);
        }
        
        io_uring_cqe_seen(ring, cqe);
    }
}

// Use with spatial locality caching
void search_graph_with_prefetch(Query query) {
    // Predict which nodes we'll need
    NodeID predicted_nodes[64];
    predict_hot_nodes(query, predicted_nodes, 64);
    
    // Async prefetch while computing distances
    prefetch_nodes(&ring, graph_fd, predicted_nodes, 64);
    
    // Start graph traversal (will find prefetched nodes in cache)
    traverse_graph(query);
}
```

**Benefits**:
- Prefetch predicted nodes with minimal overhead
- Overlaps I/O with computation
- Buffers I/O benefits from page cache (hot nodes stay cached)
- Coordinates with spatial locality caching strategy

## Common Pitfalls

### 1. Queue Full

```c
sqe = io_uring_get_sqe(&ring);
if (!sqe) {
    // Queue is full! Submit pending operations first
    io_uring_submit(&ring);
    sqe = io_uring_get_sqe(&ring);
}
```

### 2. Forgetting to Mark Completions Seen

```c
io_uring_wait_cqe(&ring, &cqe);
// Process result...
io_uring_cqe_seen(&ring, cqe);  // MUST do this!
```

Without `cqe_seen`, completion queue fills up and blocks.

### 3. Memory Management

```c
// Bad: Buffer goes out of scope before I/O completes
{
    char buffer[4096];
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buffer, 4096, 0);
    io_uring_submit(&ring);
}  // buffer destroyed, but I/O still in flight!

// Good: Keep buffer alive until completion
char *buffer = malloc(4096);
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buffer, 4096, 0);
io_uring_sqe_set_data(sqe, buffer);
io_uring_submit(&ring);
// Later...
io_uring_wait_cqe(&ring, &cqe);
free(io_uring_cqe_get_data(cqe));
```

## Debugging and Monitoring

### Enable Tracing

```bash
# Trace io_uring system calls
strace -e io_uring_setup,io_uring_enter,io_uring_register ./program

# Monitor with perf
perf record -e 'io_uring:*' ./program
perf report
```

### Check Queue Status

```c
// Get SQ utilization
unsigned sq_ready = io_uring_sq_ready(&ring);
unsigned sq_space = io_uring_sq_space_left(&ring);
printf("SQ: %u ready, %u space\n", sq_ready, sq_space);

// Get CQ utilization  
unsigned cq_ready = io_uring_cq_ready(&ring);
printf("CQ: %u completions ready\n", cq_ready);
```

## Resources

### Official Documentation
- [Kernel documentation](https://kernel.dk/io_uring.pdf) - Jens Axboe's paper
- [liburing GitHub](https://github.com/axboe/liburing) - Official userspace library
- [Man pages](https://man7.org/linux/man-pages/man7/io_uring.7.html)

### Tutorials
- [Lord of the io_uring](https://unixism.net/loti/) - Comprehensive tutorial
- [io_uring by Example](https://github.com/shuveb/io_uring-by-example)

### Talks
- ["Efficient IO with io_uring"](https://www.youtube.com/watch?v=-5T4Cjw46ys) - Jens Axboe at LSFMM 2019

### Articles
- [How io_uring and eBPF Will Revolutionize Programming in Linux](https://thenewstack.io/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/)
- [What is io_uring?](https://cor3ntin.github.io/posts/iouring/)

## Kernel Version Requirements

| Kernel Version | Features Available |
|----------------|-------------------|
| 5.1 (May 2019) | Basic I/O operations |
| 5.4 | accept, connect, sendmsg, recvmsg |
| 5.5 | openat, statx, timeout |
| 5.6 | Fixed files/buffers, SQPOLL improvements |
| 5.7 | Faster poll, multishot accept |
| 5.11+ | Even more operations and optimizations |

Check your kernel: `uname -r`

Most modern distros (Ubuntu 20.04+, Fedora 32+, RHEL 9+) support io_uring.

## Summary

io_uring represents a fundamental shift in how Linux handles async I/O:

**Key advantages**:
- 🚀 3-8× better performance than traditional I/O
- 💰 Up to 500× reduction in syscalls
- ⚡ True async for all I/O types (buffered, direct, network, filesystem)
- 🔧 Rich feature set (chaining, timeouts, polling, zero-copy)
- 🎯 Future-proof: actively developed, adding new features

**When to use**:
- High-performance I/O workloads
- Applications requiring low latency
- Systems with many concurrent I/O operations
- Modern Linux environments (kernel 5.1+)

**Trade-offs**:
- Requires kernel 5.1+ (2019)
- More complex API than traditional I/O
- Need to manage ring buffers and completions

For new projects on modern Linux, io_uring should be your default choice for async I/O.

---

**Next Steps**:
- Try the basic example code
- Benchmark io_uring vs traditional I/O on your workload
- Explore advanced features (polling, chaining, fixed files)
- Integrate into your disk-based ANN search project

**Related Notes**:
- [Linux AIO Limitations](./linux-aio-limitations.md) - Why io_uring was needed
- Buffered vs Direct I/O
- SSD Performance Optimization
