# Buffer Managers

Notes on buffer managers, page caches, and how storage engines manage in-memory pages before reading from or flushing to disk.

## Why They Matter

Classical database systems were designed around the assumption that disk I/O dominated everything else. In that world, the main job of the buffer manager was to minimize page faults and disk traffic.

That hardware model is weaker now. SSDs and especially NVMe arrays make the gap between storage and memory much smaller than it was in the HDD era. That changes the optimization target:

* the hot working set should behave almost like an in-memory system
* the cold working set can stay on SSD
* the buffer manager becomes responsible for making that split efficient

One useful framing from the modern literature is that the resident-page fast path matters a lot. If a page is already in memory, accessing it should be almost as cheap as following a normal pointer.

## Traditional Buffer Managers

At a high level, a traditional buffer manager:

* maps `page_id -> buffer frame`
* pins/fixes a page while it is in use
* unpins/unfixes it when the caller is done
* tracks dirty state
* chooses victims for eviction
* coordinates concurrency around lookup, page state, and eviction

The usual implementation shape is:

* a page table, often a hash table
* fixed-size buffer frames
* a replacement policy such as clock or LRU variants
* latches or locks around page-table access and page state transitions

This sounds straightforward, but on modern hardware the lookup path itself can become expensive. Even when the page is resident, a traditional design often still pays for:

* hash lookup
* latch or lock traffic
* pointer indirection after the lookup

That means the buffer manager can consume meaningful CPU time even when storage is not the bottleneck.

## What Modern Designs Are Trying To Fix

The broad goal is to preserve the cost advantage of disk-backed systems without paying large overheads on every in-memory access.

More concretely:

* keep the in-memory case very close to pure pointer chasing
* degrade gracefully when data spills beyond RAM
* avoid pushing too much complexity into the rest of the storage engine
* still support concurrency, eviction, and recovery correctly

## Pointer Swizzling

Pointer swizzling tries to eliminate most of the traditional page-table lookup overhead on the hot path.

The idea is to store either:

* a real memory pointer if the page is resident
* an encoded page identifier if the page is not resident

Then access becomes:

* check a tag bit
* if swizzled, dereference directly
* if unswizzled, load the page, convert the reference, and continue

Why this is attractive:

* the in-memory case is very cheap
* the branch is predictable when most accesses hit resident pages
* it moves closer to in-memory execution without giving up out-of-memory support

Main drawbacks:

* it leaks buffer-management concerns into the storage-engine data structures
* eviction gets tightly coupled to unswizzling
* it tends to fit tree-shaped structures better than general graph-shaped ones
* concurrency and engineering complexity increase

This is a good example of a design that is very fast but somewhat intrusive.

## VM-Assisted Buffer Management

VM-assisted approaches such as VMCache try to keep a cleaner programming model while still making resident pages cheap to access.

The core idea is:

* reserve a large virtual-memory region for the database
* derive a page's address directly from its `page_id`
* maintain page residency in separate state
* evict pages manually with virtual-memory primitives instead of delegating full control to the OS page cache

This restores a more normal buffer-manager API. The fast path becomes close to:

* compute the virtual address
* check page state
* return the pointer if the page is present

Compared to pointer swizzling, this is usually less intrusive and easier to integrate into a real system.

## Why This Is Different From Plain `mmap`

There is an important distinction between:

* file-backed `mmap` where the OS effectively owns buffering and paging
* using virtual-memory mechanisms as building blocks inside a DB-managed buffer pool

The modern argument is usually against the first and more favorable toward the second.

The complaint about plain `mmap` is not that virtual memory is inherently bad. It is that a DBMS often wants tighter control over eviction, writeback, concurrency, and performance behavior than the OS page cache provides.

## VMCache Tradeoffs

VMCache-style designs still have real costs:

* `madvise(MADV_DONTNEED)` is not free
* heavy eviction can trigger TLB shootdowns
* multithreaded eviction can run into kernel contention
* large virtual mappings imply non-trivial page-table memory overhead

That means the design is very appealing for workloads where:

* the hot set fits comfortably in memory
* out-of-memory accesses exist but are not completely adversarial
* the system wants a cleaner interface than pointer swizzling offers

It becomes less attractive as the out-of-memory factor grows very large or the workload degenerates into heavy random misses.

## Notes from `In-Memory Performance for Big Data`

* Main question: can a disk-based DBMS keep a real buffer pool but perform close to an in-memory system when the working set fits in RAM?
* The paper's answer is pointer swizzling for B-tree pages.
* Instead of always following `page_id -> buffer pool lookup -> frame`, a parent page can store a direct pointer to a resident child page.
* Swizzling is only for in-memory navigation. The persistent on-disk representation still uses stable `page_id`s.
* Traditional buffer pools still pay for hash lookup, pin/unpin work, and latch traffic even when every relevant page is already resident.
* The design is intentionally narrow: swizzling focuses on parent-to-child pointers in B-trees, not arbitrary object graphs.
* Eviction requires un-swizzling: if a child page is going to leave memory, parent references to it must be rewritten back into `page_id`s.
* The paper proposes child-to-parent metadata in buffer-frame descriptors to make un-swizzling efficient.
* Swizzled parent references contribute to the child page's pin count, so pages with live structural references are protected from eviction.
* The broader takeaway is that a buffer pool is not only about handling misses; its own CPU overhead can become a bottleneck when the working set fits in memory.
* This paper is an important bridge from traditional buffer pools to later systems like `LeanStore` and `Umbra`.

## References

* Michael Zinsmeister, "Why databases found their old love of disk again", TUMuchData, February 7, 2024
  https://tumuchdata.club/post/hdd-to-ram-to-ssd/
* Michael Zinsmeister, "How to have your cake and eat it too with modern buffer management Pt. 1: Pointer Swizzling", TUMuchData, February 14, 2024
  https://tumuchdata.club/post/modern-buffer-managers-1/
* Michael Zinsmeister, "How to have your cake and eat it too with modern buffer management Pt. 2: VMCache", TUMuchData, February 26, 2024
  https://tumuchdata.club/post/modern-buffer-managers-2/

* Stavros Harizopoulos, Daniel J. Abadi, Samuel Madden, Michael Stonebraker, "OLTP Through the Looking Glass, and What We Found There", SIGMOD 2008
  https://www.cs.cmu.edu/~natassa/courses/15-823/Fall17/papers/harizopoulos-oltp.pdf
* Goetz Graefe, Harumi A. Kuno, Nicolás Norberto, "In-Memory Performance for Big Data", DaMoN 2014
  https://research.google/pubs/in-memory-performance-for-big-data/
* Viktor Leis, Florian Haas, Adriana Zuniga, Thomas Ziegler, Carsten Binnig, Hannes Muehleisen, "LeanStore: In-Memory Data Management Beyond Main Memory", ICDE 2018
  https://db.in.tum.de/~leis/papers/leanstore.pdf
* Thomas Neumann, Tobias Muehlbauer, and Alfons Kemper, "Umbra: A Disk-Based System with In-Memory Performance", CIDR 2020
  https://www.vldb.org/cidrdb/papers/2020/p29-neumann-cidr20.pdf
* Ryan Crotty, Alex Galakatos, and Andrew Pavlo, "Are You Sure You Want to Use MMAP in Your Database Management System?", CIDR 2022
  https://www.vldb.org/cidrdb/papers/2022/p13-crotty.pdf
* Florian Haas, Jan Wenzel, Bastian Schmidt, Thomas Neumann, Viktor Leis, "What Modern NVMe Storage Can Do, And How To Exploit It: High-Performance I/O for High-Performance Storage Engines", VLDB 2023
  https://www.vldb.org/pvldb/vol16/p2090-haas.pdf
* Viktor Leis, Florian Haas, Jan Wenzel, Bastian Schmidt, "Virtual-Memory Assisted Buffer Management", SIGMOD 2023
  https://www.ibr.cs.tu-bs.de/vss/Publications/2023/leis_23_sigmod.pdf
