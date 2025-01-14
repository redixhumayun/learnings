## Chapter 9 - Scheduling: Proportional Share

The chapter talks about using tickets to represent the share of a resource (in this case, CPU time) that a process should receive. The concept of tickets seems to be a very powerful abstraction, as it allows you to generalize shares of any resource (CPU, memory etc.). 

### Lottery Scheduling
The example used here conveys the idea quite well. You have 2 processes A and B and they are given 75% and 25% of CPU time. For every instance of a time interval, a random number is generated and if that number is between 0 to 74, A is given CPU time. If the number is 75-99, B is given CPU time. Of course, this is contingent on true randomness in your random number generation.

A very interesting graph is the one depicting the relationship between unfairness and job length. As job length goes up, unfairness reduces and this is because of the [law of large numbers](https://www.probabilitycourse.com/chapter7/7_1_1_law_of_large_numbers.php). Think of this in terms of flipping a coin:
if you flip a coin 10 times, getting 7 heads is not out of the ordinary.
if you flip a coin 1000 times, getting 700 heads is extremely unlikely.
Simply put, with a larger sample size, your variance goes down.

### Stride Scheduling
A more deterministic version of lottery scheduling, where the state of each process (how much time share of the CPU it's received) is tracked at a fine granularity. It requires the maintenance of a global state that lottery scheduling does not.

### Linux Completely Fair Scheduler(CFS)
This uses what seems to be a variation of the ticketing system with a different name - virtual runtime(vruntime). It tracks how long each process has run, so it needs to maintain some form of global state. The interesting thing is that it exposes a variety of tuning knobs like the scheduling latency, min granularity and weighting to allow users of this program to determine what job should be given priority.

The CFS uses a [red-black tree](https://medium.com/basecs/painting-nodes-black-with-red-black-trees-60eacb2be9a5) (which is essentially a self-balancing BST) to keep track of which process needs to run next. All processes that require I/O are removed from this tree so that they are not unnecessarily scheduled. Here's a [fun visualisation](https://www.cs.usfca.edu/~galles/visualization/RedBlack.html) of the red-black tree. A few [other use cases](https://stackoverflow.com/a/22558777/6593789) of red-black trees.

A major design decision of the CFS seems to be the time slice given to a process before the scheduler will attempt to interrupt it and replace it with another process. Interestingly, [Go 1.19.1 has a time slice of 10ms](https://github.com/golang/go/blob/go1.19.1/src/runtime/proc.go#L5279-L5281). I'm curious to know how other runtimes set this parameter (specifically, runtimes that allow pre-emption).

## Chapter 10 - Multiprocessor Scheduling

This has been, by far, my favorite chapter in the book because it starts venturing into the nitty-gritty details of how modern schedulers work. Most computers today are multi-processor machines, which is both a blessing and a curse.

The chapter briefly mentions the idea of temporal locality, spatial locality and cache coherence. All these concepts are super critical when it comes to performance, and is the basis for cache-conscious data structures. A simple example to illustrate the difference is thinking about handling hash table collissions with [linear probing vs chaining](https://stackoverflow.com/questions/36531119/what-is-the-difference-between-chaining-and-probing-in-hash-tables). 

The former is better for cache performance because the latter does a bunch of pointer chasing into non-contiguous memory addresses. A great talk for anyone interested in more about this is [Data Oriented Design by Mike Acton](https://www.youtube.com/watch?v=rX0ItVEVjHc). Also, Abhinav Upadhyay goes into a ton of detail on this in his great live session on [performance engineering lessons from 1BRC](https://blog.codingconfessions.com/p/recording-six-key-performance-engineering).

Another important concept discussed here is cache affinity (which is also referred to as [processor affinity or CPU affinity](https://en.wikipedia.org/wiki/Processor_affinity)). Essentially, I ran job A on core 1, and I would like to continue running job A on core 1 without scheduling alternate jobs on core 1. This sounds quite similar to [thread-per-core architecture](https://without.boats/blog/thread-per-core/).

There are broadly two classes of schedulers here:
### Single Queue Scheduling
You have a single global queue, where all jobs are pushed onto it and then either:
* randomly assigned to a core
* assigned to a specific core to maintain cache affinity.

The problem with the former approach is cache thrashing while the issue with the latter approach is complexity. But overall, there is a big problem of contention with a single queue approach.

### Multi Queue Scheduling
You have a run queue per processor, and a job is placed on a single queue based on some heuristics.
The big problem with multi-queue scheduling is the possibility of load imbalance, and the typical approach to deal with this is work-stealing. [Tokio](https://tokio.rs/blog/2019-10-scheduler), part of the Rust ecosystem, is a famous example of a work-stealing scheduler.

I mentioned thread-per-core above and I think it's quite relevant to this. This is something which ScyllaDB famously implements via SeaStar. I've shared these links before but I think this [2 part series](https://www.scylladb.com/2016/04/14/io-scheduler-1/) by ScyllaDB on their production grade I/O scheduler sheds a lot of light in what goes into thinking about the design behind these systems. They talk about what sounds like their own user-space ticketing mechanism so it's quite relevant to these chapters.

A question that has come to mind before and that I continue to think about:
What workloads are best to run on different kinds of schedulers? For instance, web servers are best suited to run on a work stealing scheduler. But, what about non work-stealing schedulers? And what about thread-per-core systems?