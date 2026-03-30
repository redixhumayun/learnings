General notes from the paper:

- batch I/O operations
- batch I/O operations better (with adaptive batching). Use a counter to determine how many SQE's to submit at once
- without I/O batching, io_uring will tie you to device latency
- benefit only comes through at high I/O concurrency. Otherwise you're better off with pread/pwrite
- async execution model is a pre-requisite - coroutines/fibers that yield on I/O allow you to generate high queue depth.
- use ring-per-thread with computation and I/O colocated on same thread. pin worker threads and NIC's to the same chiplet (chiplets consist of cores). this is effectively thread-per-core with a ring-per-core.
- register buffers up front to avoid cost of validation and pinning physical pages for DMA access
- use NVMe passthrough + IOPoll under high load. Bypasses a lot of layers but IOPoll burns a CPU core.
- for network, zero-copy send + recv via io-uring will give benefit over epoll but only for messages above ~1KiB.
- use SQPoll so kernel thread continuously polls SQ for entries. burns a core but useful in high throughput I/O scenarios

## Storage
- Using io_uring without batching ties you to device latency effectively
- Queue depth is the primary variable — io_uring only beats pread/pwrite when you have many concurrent I/O requests in flight. Low concurrency → no benefit.
- Async execution model is a prerequisite — coroutines/fibers that yield on I/O are how you generate high queue depth without many OS threads. Thread-per-request can't do this efficiently.
- Batching amortizes syscall overhead for reads, hides latency for writes — but larger batches increase latency variance. Adaptive batching (scale batch size with load) is the practical solution.
- Registered buffers reduce per-I/O CPU cost by eliminating repeated kernel buffer pinning.
- One ring per thread (or one dedicated I/O thread with a lock-free channel) avoids contention. Shared ring with a mutex is the worst case.
- NVMe passthrough + IOPoll give the biggest throughput gains at scale (3.4–3.5×) but require bypassing the filesystem — only viable when the DBMS manages its own caching (i.e. has a buffer pool).
- fsync is broken in io_uring — it's blocking, falls back to worker threads, and can't be used with IOPoll rings at all. This is a serious problem for WAL durability. The only clean async durability path is NVMe passthrough flush, which requires raw device access.
- Block size matters — larger blocks amortize CPU cost per byte but exceeding kernel thresholds triggers async worker fallback, reintroducing overhead.

## Networking
- Overall, zero-copy with registered buffers is the most efficient configuration for large messages
- Ring-per-thread with computation and I/O colocated on the same thread
- io-uring provides zero-copy send + recv which is a boost over epoll which just does zero-copy send
- RECVSEND_POLL_FIRST flag: skip the speculative non-blocking attempt when you already know the socket is empty
- Pin worker threads and NIC's to the same chiplet (chiplets consist of cores). Cross-chiplet interrupt handling adds latency
- SQPoll will have kernel thread continuously polling SQ for entries. Only useful in high throughput I/O scenarios