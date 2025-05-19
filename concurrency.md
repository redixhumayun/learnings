# Concurrency
Concurrency is about executing multiple threads at once, either on a single core or multiple cores.

There are broadly two styles of concurrency:

* communicate by sharing memory
* share memory by communicating ([popularized by Go](https://go.dev/blog/codelab-share))

## Shared Memory
This is the more popular form of concurrency that involves sharing memory regions among multiple threads. We need to be able to guard those regions of memory to ensure exclusive access by guaranteeing *mutual exclusion*

*Note: To understand implementation details of mutexes look at the [atomics section below](#Atomics)*

### Mutexes
Mutexes ensure exclusive access to a shared resource in a multi-threaded environment. A thread locks a mutex before accessing a shared resource and unlocks it after. Example:

```c
pthread_mutex_lock(&mutex);
// Critical section
pthread_mutex_unlock(&mutex);
```

### Condition Variables
Condition variables allow threads to wait for certain conditions. They're used with mutexes to synchronize thread execution. Example:

```c
pthread_cond_wait(&cond, &mutex); // Wait
pthread_cond_signal(&cond);       // Signal another thread
```

### Semaphores
Semaphores manage access to shared resources, allowing multiple accesses. They're particularly useful in signal handlers. Example:

```c
sem_wait(&sem); // Decrement and wait if zero
sem_post(&sem); // Increment, potentially unblocking a waiter
```

### Seq Locks
Seq locks are optimistic concurrency control mechanisms that allow readers to operate without blocking while ensuring writers have exclusive access. They're particularly efficient for read-heavy workloads with occasional writes:

```c
// Writer
do {
    seq = read_seqlock_begin(&lock);
    // Modify shared data
} while (write_seqlock_retry(&lock, seq));

// Reader
do {
    seq = read_seqlock_begin(&lock);
    // Read shared data
} while (read_seqlock_retry(&lock, seq));
```

> **Note:** Seqlocks can only be used with primitive types that do not contain pointers. See [here](https://docs.kernel.org/locking/seqlock.html) for more details

### Atomics
Atomics are a complicated low-level topic that form the foundation on top of which mutexes are built. 

An atomic operation is an indivisible operation.

#### Operations In Atomics

1. `load()`
2. `store()`
3. `exchange()`
4. `compare_exchange_weak()`
5. `compare_exchange_strong()`

#### Memory Order Operations

1. `memory_order_relaxed`
2. `memory_order_acquire`
3. `memory_order_consume`
4. `memory_order_acq_rel`
5. `memory_order_release`
6. `memory_order_seq_cst`

Each atomic operation has a certain subset of memory orderings that can be specified for it. 
Atomic operations combined with memory ordering can create certain relationships.

For a small set of implementations using atomics in Rust, [see here](https://github.com/redixhumayun/rs-examples)

### Lock-Free & Wait-Free Programming
Lock-free and wait-free are terms used to describe progress guarantees in concurrent algorithms.

**Lock-free** algorithms guarantee that at least one thread makes progress in a finite number of steps. While some threads might experience starvation, the system as a whole always advances.

**Wait-free** algorithms provide stronger guarantees: every thread makes progress in a bounded number of steps, regardless of the actions of other threads. This eliminates starvation entirely.

The key differences:
- Lock-free: System-wide progress guaranteed, but individual threads may starve
- Wait-free: Individual thread progress guaranteed in bounded time
- Blocking: No progress guarantees (e.g., mutex-based code)

This comparison table shows how different synchronization primitives relate to these guarantees:

| Primitive   | Readers                                             | Writers                                         | Whole Algorithm                                                         |
|-------------|-----------------------------------------------------|-------------------------------------------------|-------------------------------------------------------------------------|
| **seqlock** | *lock-free* (they just load counter; may retry)     | **NOT** lock-free (grab internal spin-lock)     | at best *lock-free*: somebody makes progress, writers can starve readers |
| **RCU**     | *wait-free* (bounded operations, no retries)        | *lock-free* (grace period, but no blocking)     | *wait-free* for readers, *lock-free* for writers                        |


## Communicating Memory
This is the style of concurrent programming popularized by Go. It's safer due to not locking regions of memory, thus avoiding classic problems like deadlocks. This style is also called Communicating Sequential Processes (CSP).

```go
package main

import (
    "fmt"
    "time"
)

func worker(done chan bool) {
    fmt.Print("working...")
    time.Sleep(time.Second)
    fmt.Println("done")

    done <- true
}

func main() {
    done := make(chan bool, 1)
    go worker(done)

    <-done
}
```

The main thread blocks until the value of done is set to true. 

MPSC in Rust is an example of Communicating Sequential Processes. The core principle of CSP is, “don’t communicate by sharing memory, share memory by communicating”.

### CPS (Continuation Passing Style)
An alternative to CSP above is called CPS which is a complicated term for passing callbacks to functions. JS is a popular example of this.

```javascript
function fetchData(url, callback) {
  // Simulating an asynchronous API call
  setTimeout(() => {
    const data = { id: 1, name: "John Doe" };
    callback(data);
  }, 1000);
}

function processData(data) {
  console.log("Processing data:", data);
}

fetchData("<https://example.com/api>", processData);
```

Blog post detailing this more: https://matt.might.net/articles/by-example-continuation-passing-style/

Promises are a nicer version of this CPS paradigm.

```javascript
step1(5)
    .then(result1 => step2(result1))
    .then(result2 => step3(result2))
    .then(finalResult => console.log("Final result:", finalResult))
    .catch(error => console.error("An error occurred", error));
```

## Async Runtimes, Web Servers & Async IO
This section is a little specific to Rust.
Async runtimes are about managing concurrency in the context of an application - a somewhat higher level than mutexes and CSP.

Roughly what they do

![](/assets/img/async_runtime.png)

Any runtime has the following core concepts:
1. Scheduler
2. Tasks & Futures
3. Wakers
4. Reactors

### Scheduler
The scheduler roughly combines:
* event loops
* task poller

It is the "hot" loop of the entire async runtime.

### Tasks & Futures
`Futures` (atleast in Rust) are anything that implement the `Future` trait.

```rust
struct SharedState {
    completed: bool,
    waker: Option<Waker>,
}

pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

impl Future for TimerFuture {
    type Output = ();
    fn poll(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            return Poll::Ready(());
        }
        shared_state.waker = Some(cx.waker().clone());
        return Poll::Pending;
    }
}
```

### Wakers
A waker is an object that can be used to put the task/future back onto a run queue.

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let arc_clone = Arc::clone(&arc_self);
        arc_self.task_sender.send(arc_clone).unwrap();
    }
}
```

### Reactors
Reactors are components that can react to the readiness of events. These are connected to the async runtime and can enqueue tasks onto the scheduler, once they are ready.

Mechanisms typically include `poll` , `epoll` (on Linux) & `kqueue` (on Unix). These are typically used to signal readiness of sockets so that events can be read from them.

It's important to discuss here the difference between the different types of async io models - readiness based & completion based.

>epoll and kqueue fall under the readiness based model. These syscalls only indicate when a file descriptor is available to be read from or written to. There's the overhead of an additional syscall to actually do the writing or reading in this case.
>
>io_uring on the other hand falls under the completion based model. This model involves providing the file descriptors you're interested in reading from or writing to and getting the actual results back. It avoids the overhead of the additional syscall.
>
>One interesting difference is that it makes no sense to use the readiness based model when doing file I/O. It doesn't mean anything to wait for file "readiness" in this case since files are always ready for I/O.

## Cooperative & Preemptive Schedulers
It's important to discuss these two terms when talking about async runtimes. To understand this, consider the following two classes of coroutines:
1. Stackful (have an OS-like stack allocated) -> also called green threads, fibers etc.
2. Stackless (no call stack allocated) -> also called futures, promises etc.

Stackful coroutines can be pre-empted by the scheduler. Stackless coroutines cannot be pre-empted by the scheduler.

![](/assets/img/async_terms.png)

Mix and match above two combinations.

stackless + cooperative → Rust futures, JS promises

stackful + preemptive → Goroutines

stackless + preemptive → isn’t possible

stackful + cooperative → need examples

Stackless coroutines require the rewiring of the language to use “coloured” functions. Stackful coroutines don’t involve this. [Bob Nystrom blog post](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)

## Styles Of Runtimes
Broadly:
1. single-threaded
2. work-stealing
3. thread-per-core

# Resources

## External
### Atomics & Memory Ordering
1. [Understanding Atomics And Memory Ordering](https://dev.to/kprotty/understanding-atomics-and-memory-ordering-2mom)
1. [The memory order reference from cppreference](https://en.cppreference.com/w/cpp/atomic/memory_order)
1. [Atomics on the GCC Wiki](https://gcc.gnu.org/wiki/Atomic/GCCMM/AtomicSync)
1. [Memory Ordering At Compile Time](https://preshing.com/20120625/memory-ordering-at-compile-time/)
1. [Memory Barriers From The Linux Kernel Documentation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/memory-barriers.txt?id=HEAD)
1. [ArangoDB's blog on memory barriers in C++](https://arangodb.com/2021/02/cpp-memory-model-migrating-from-x86-to-arm/)
1. [The Danger Of Atomic Operations](https://abseil.io/docs/cpp/atomic_danger)
1. [What Every Systems Programmer Should Know About Concurrency](https://assets.bitbashing.io/papers/concurrency-primer.pdf)
1. [Seq Locks](https://docs.kernel.org/locking/seqlock.html)

### Sync & Async Web Servers
1. https://tenthousandmeters.com/blog/python-behind-the-scenes-12-how-asyncawait-works-in-python/
2. https://eli.thegreenplace.net/2017/concurrent-servers-part-1-introduction/ 
3. https://doc.rust-lang.org/book/ch20-00-final-project-a-web-server.html
4. https://rust-lang.github.io/async-book/09_example/00_intro.html
5. https://ibraheem.ca/posts/too-many-web-servers/

### Aysnc Runtimes

#### Schedulers
1. [Tokio Scheduler](https://tokio.rs/blog/2019-10-scheduler)
2. [How Tokio Schedules Tasks](https://rustmagazine.org/issue-4/how-tokio-schedule-tasks/)
3. [InfluxDB Using Tokio For CPU Bound Tasks](https://www.influxdata.com/blog/using-rustlangs-async-tokio-runtime-for-cpu-bound-tasks/)
4. [Scheduling Internals](https://tontinton.com/posts/scheduling-internals/)

#### Reactors / Async IO
1. [Abstraction Over kqueue & iouring](https://tigerbeetle.com/blog/a-friendly-abstraction-over-iouring-and-kqueue)
2. [io_uring basics](https://notes.eatonphil.com/2023-10-19-write-file-to-disk-with-io_uring.html)
3. [monoio implementation blog post](https://en.ihcblog.com/rust-runtime-design-1/)

#### Miscellaneous
1. [Async I/O In Depth Playlist](https://www.youtube.com/watch?v=_3LpJ6I-tzc&list=PLb1VOxJqFzDd05_aDQEm6KVblhee_KStX&index=4)
2. [GitHub Repo For Above Playlist](https://github.com/nyxtom/async-in-depth-rust-series)
3. [Withoutboats on thread-per-core](https://without.boats/blog/thread-per-core/) [[paper referenced in post](https://penberg.org/papers/tpc-ancs19.pdf)]
4. [Thread-Per-Core In Async Rust](https://emschwartz.me/async-rust-can-be-a-pleasure-to-work-with-without-send-sync-static/)
5. [Phil's tweet comparing runtime approaches](https://x.com/eatonphil/status/1773518000043299309)

## Personal Blog
1. [Async Runtimes](https://redixhumayun.github.io/async/2024/08/05/async-runtimes.html)
2. [Async Runtimes II](https://redixhumayun.github.io/async/2024/09/18/async-runtimes-part-ii.html)
3. [Async Runtimes III](https://redixhumayun.github.io/async/2024/10/10/async-runtimes-part-iii.html)
4. [Atomics & Memory Ordering](https://redixhumayun.github.io/systems/2024/01/03/atomics-and-concurrency.html)

