# Computer System Learning Notes

A collection of in-depth notes and explorations on computer systems, covering topics from low-level hardware to high-level system design.

## 📚 Table of Contents

### 🖥️ CPU & Memory
- Processor Architecture
- Cache Hierarchies
- Memory Models
- NUMA Systems
- CPU Performance Counters

### 💾 Disk & I/O
- **[Linux AIO Limitations](./disk-io/linux-aio-limitations.md)** - Understanding Linux Asynchronous I/O and why it's problematic
- Block Layer Architecture
- Filesystem Internals
- SSD vs HDD Characteristics
- io_uring Modern Async I/O
- Direct I/O (O_DIRECT) vs Buffered I/O
- Page Cache Management
- I/O Scheduling Algorithms

### 🌐 Networking
- TCP/IP Stack
- Socket Programming
- Zero-Copy Techniques
- Network Performance Tuning
- epoll/io_uring for Network I/O

### 🔄 Concurrency & Synchronization
- Thread Models
- Lock-Free Data Structures
- Memory Ordering & Barriers
- Deadlock Prevention
- Concurrent Algorithms

### 🏗️ System Design
- Cache Design Patterns
- Database Internals
- Distributed Systems
- Performance Optimization
- Profiling & Debugging Tools

### 🔧 Operating Systems
- Process Management
- Virtual Memory
- Scheduling Algorithms
- System Calls & Interrupts
- Kernel Space vs User Space

## 📝 Contributing

These are personal learning notes, but suggestions and corrections are welcome! If you spot an error or have additional insights, please open an issue or PR.

## 🌟 Featured Notes

### Recent Additions
- **Linux AIO Limitations** - Deep dive into why Linux AIO fails with buffered I/O and comparison with io_uring

### Coming Soon
- io_uring Advanced Patterns
- SSD Internal Architecture
- Page Cache Tuning for Database Workloads
- NUMA-Aware Data Structures

## 📚 Resources & References

### Books
- *Computer Systems: A Programmer's Perspective* - Bryant & O'Hallaron
- *Operating Systems: Three Easy Pieces* - Remzi H. Arpaci-Dusseau
- *The Linux Programming Interface* - Michael Kerrisk
- *Database Internals* - Alex Petrov

### Online Resources
- [Brendan Gregg's Blog](https://www.brendangregg.com/) - Performance analysis
- [LWN.net](https://lwn.net/) - Linux kernel news
- [The Morning Paper](https://blog.acolyer.org/) - CS paper summaries

### Papers
- "Efficient IO with io_uring" - Jens Axboe
- "The Design and Implementation of a Log-Structured File System" - Rosenblum & Ousterhout
- "Dynamo: Amazon's Highly Available Key-value Store" - DeCandia et al.

## 📧 Contact

Questions or suggestions? Open an issue or reach out via [your contact method].

## 📄 License

This repository is for educational purposes. Code examples are provided under MIT License unless otherwise specified.

---

**Last Updated**: September 2025

*"The best way to learn systems is to build them, break them, and understand why they broke."*
