# Linux AIO: Understanding Its Limitations

## Overview

Linux AIO (Asynchronous I/O) is a kernel interface designed to provide non-blocking I/O operations. However, it has significant limitations that make it unsuitable for many use cases. This document explains what AIO is, how it works, and why modern applications should consider alternatives like io_uring.

## What is Linux AIO?

Linux AIO provides an API for submitting I/O operations that theoretically complete asynchronously without blocking the calling thread.

### Basic API

```c
#include <libaio.h>

// Setup
io_context_t ctx;
io_setup(128, &ctx);

// Prepare I/O operation
struct iocb iocb;
io_prep_pread(&iocb, fd, buffer, size, offset);

// Submit
struct iocb *list[1] = {&iocb};
io_submit(ctx, 1, list);

// Poll for completion
struct io_event events[1];
io_getevents(ctx, 1, 1, events, NULL);

// Cleanup
io_destroy(ctx);
```

## Critical Limitations

### 1. Only Works Asynchronously with O_DIRECT

**The biggest problem**: AIO only provides true async behavior when using `O_DIRECT` flag.

```c
// True async (works as expected)
fd = open("file", O_RDONLY | O_DIRECT);
io_submit(ctx, 1, &iocb);  // Returns immediately

// FALSE async (blocks despite API!)
fd = open("file", O_RDONLY);  // Buffered I/O
io_submit(ctx, 1, &iocb);     // BLOCKS until I/O completes!
```

**Why this matters**:
- `O_DIRECT` bypasses the kernel page cache
- You lose all automatic caching benefits
- Must handle 512-byte alignment manually
- For many workloads, this is slower than buffered I/O
- Buffered I/O with AIO silently becomes synchronous

### 2. Limited Operation Support

AIO only supports a small subset of operations:

**Supported**:
- `pread()` / `preadv()`
- `pwrite()` / `pwritev()`
- `fsync()` / `fdatasync()` (limited)

**NOT Supported**:
- `open()` - must open files synchronously first
- `close()` - must close synchronously
- `stat()` / `fstat()` - can't async check metadata
- `accept()` - can't async accept network connections
- `connect()` - can't async connect to sockets
- Network I/O (`send`/`recv`) - doesn't work reliably
- Many other syscalls

This forces mixing of blocking and async operations, negating the benefits.

### 3. High Syscall Overhead

Even when async, AIO requires many syscalls:

```c
// Submit 100 I/Os
for (int i = 0; i < 100; i++) {
    io_submit(ctx, 1, &list[i]);  // 100 syscalls
}

// Poll for completions
for (int i = 0; i < 100; i++) {
    io_getevents(ctx, 1, 1, events, NULL);  // 100 syscalls
}

// Total: 200 syscalls for 100 I/Os
```

**Context switches are expensive** (~1-2μs each), adding significant overhead.

### 4. Broken Cancellation

```c
// Attempt to cancel in-flight I/O
io_cancel(ctx, &iocb, &result);

// Returns success, but...
// The I/O often completes anyway!
```

Cannot reliably cancel operations, making timeout handling problematic.

### 5. Poor Batching Support

While you can submit multiple operations at once, the implementation doesn't batch efficiently:

```c
struct iocb *list[10] = {...};
io_submit(ctx, 10, list);  // Still enters kernel for each

// Can't build up a queue of operations without submitting
```

### 6. No Operation Chaining

Cannot express dependencies between operations:

```c
// Want: "Read X, then when complete, write Y"
// Must do:
io_submit(ctx, 1, &read_iocb);
io_getevents(ctx, 1, 1, events, NULL);  // Wait
// Now submit dependent operation
io_submit(ctx, 1, &write_iocb);
```

No way to chain operations in a single submission.

### 7. Platform-Specific Behavior

- Works differently across filesystems (ext4, xfs, btrfs)
- Some storage drivers don't support it properly
- Behavior varies by kernel version
- Many edge cases and gotchas

## O_DIRECT Requirements and Downsides

If using AIO, you're forced to use `O_DIRECT`, which has strict requirements:

### Alignment Requirements

```c
// All of these must be aligned to 512 bytes (or filesystem block size):
// - Buffer address
// - Read/write size  
// - File offset

void *buffer;
posix_memalign(&buffer, 512, 4096);  // Aligned buffer

size_t size = 4096;  // Must be multiple of 512
off_t offset = 8192; // Must be multiple of 512

pread(fd, buffer, size, offset);
```

### Loss of Page Cache Benefits

```c
// With buffered I/O (normal):
read(fd, buf, 4096);  // First read: disk I/O (100μs)
read(fd, buf, 4096);  // Second read: from cache (0.1μs) - 1000× faster!

// With O_DIRECT (required for AIO):
read(fd, buf, 4096);  // First read: disk I/O (100μs)
read(fd, buf, 4096);  // Second read: STILL disk I/O (100μs) - no cache!
```

You lose:
- Automatic caching of hot data
- Read-ahead optimization
- Write coalescing
- Sharing cached data between processes

## Real-World Impact

Major database projects have abandoned or avoided Linux AIO:

**PostgreSQL**:
> "Linux AIO is not suitable for PostgreSQL's needs. It's effectively useless for buffered I/O, and O_DIRECT has its own problems."

**MongoDB**: Uses thread pools instead of AIO for async I/O on Linux.

**Most applications**: Either use blocking I/O with thread pools, or epoll for network-only async I/O.

## Comparison with io_uring

io_uring (available since Linux kernel 5.1, 2019) was created specifically to address AIO's limitations:

| Feature                  | Linux AIO | io_uring |
|--------------------------|-----------|----------|
| Buffered I/O async       | ❌ Blocks  | ✅ True async |
| Direct I/O (O_DIRECT)    | ✅         | ✅        |
| Batching efficiency      | Poor      | Excellent |
| Syscall overhead         | High      | Very low  |
| Network I/O              | ❌         | ✅        |
| File metadata ops        | ❌         | ✅        |
| Operation chaining       | ❌         | ✅ IOSQE_IO_LINK |
| Cancellation             | Broken    | ✅ Works  |
| Polling mode             | ❌         | ✅ IORING_SETUP_IOPOLL |
| Zero-copy                | No        | ✅ Yes    |
| Minimum kernel version   | 2.6+ (2003) | 5.1+ (2019) |

### io_uring Example

```c
#include <liburing.h>

struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

// Works with buffered I/O!
fd = open("file", O_RDONLY);  // No O_DIRECT needed

// Queue multiple operations (no syscalls)
for (int i = 0; i < 100; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buffers[i], sizes[i], offsets[i]);
}

// Submit batch (1 syscall for 100 I/Os!)
io_uring_submit(&ring);

// Wait for completions (1 syscall)
for (int i = 0; i < 100; i++) {
    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);
    // Process result
    io_uring_cqe_seen(&ring, cqe);
}

io_uring_queue_exit(&ring);
```

**Benefits**:
- 2 syscalls total vs 200 with AIO
- True async with buffered I/O (keeps page cache benefits)
- Lower latency, higher throughput
- Much simpler mental model

## Alternatives to Linux AIO

### 1. io_uring (Recommended)

Modern replacement for AIO with none of the limitations.

**Use when**: Linux kernel 5.1+ available

### 2. Thread Pool + Blocking I/O

Simple and effective for many workloads:

```c
// Worker threads handle blocking I/O
void* io_worker(void* arg) {
    while (true) {
        Task task = queue_pop();
        result = read(task.fd, task.buffer, task.size);  // Blocks
        callback(result);
    }
}
```

**Use when**: Simplicity preferred, moderate I/O load

### 3. mmap + madvise

For read-heavy workloads:

```c
void* data = mmap(NULL, file_size, PROT_READ, MAP_SHARED, fd, 0);

// Hint what you'll need
madvise(data + offset, size, MADV_WILLNEED);  // Prefetch

// Access directly
Node* node = (Node*)(data + node_offset);
```

**Use when**: Read-heavy, want simple code, file fits in memory

### 4. epoll (Network I/O only)

For network sockets:

```c
int epfd = epoll_create1(0);
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

struct epoll_event events[MAX_EVENTS];
int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);
```

**Use when**: Network-only async I/O needed

## Recommendations

### For New Projects

- **Use io_uring** if kernel 5.1+ available
- Provides true async for all I/O types
- Best performance and flexibility
- Future-proof

### For Existing Projects

- **Don't add Linux AIO** - limitations outweigh benefits
- Consider io_uring migration if on modern kernels
- Thread pools often simpler and good enough

### For Disk-Based ANN Search

For projects like disk-based approximate nearest neighbor search:

```
Recommended Stack:
┌─────────────────────────────────────┐
│ Application Cache (Hot nodes)      │
├─────────────────────────────────────┤
│ io_uring with buffered I/O         │ ← Best choice
├─────────────────────────────────────┤
│ Kernel Page Cache                  │ ← Free caching
├─────────────────────────────────────┤
│ SSD                                │
└─────────────────────────────────────┘
```

**Why**: 
- ANN search has many small random reads
- Hot nodes accessed repeatedly (benefits from page cache)
- io_uring provides async + caching
- Can use `posix_fadvise()` for prefetching hints

## References

- [Linux AIO man page](https://man7.org/linux/man-pages/man2/io_submit.2.html)
- [io_uring introduction](https://kernel.dk/io_uring.pdf) by Jens Axboe
- [liburing documentation](https://github.com/axboe/liburing)
- [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf)

## License

This document is provided for educational purposes. Feel free to use and adapt for your projects.

---

**TL;DR**: Linux AIO promises async I/O but only delivers for O_DIRECT (bypassing page cache). With buffered I/O, it blocks despite the "async" API. Use io_uring instead for true async I/O with all the benefits of the kernel page cache.
