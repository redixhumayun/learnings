## BTree Latching

This is a summarization of how the popular database engines do btree latching for concurrent operations

### SQLite

There is only a file-level lock. Single writer which acquires an X lock on the entire database file.

### PostgreSQL

Uses Lehman-Yao btrees. For reads, lock on parent is released before acquiring child lock. This is safe because of the high link and right pointers in Lehman-Yao btrees.

### InnoDB

Has two separate paths - optimistic and pessimistic. The optimistic path will take S latch on the index tree, traverse downt to the leaf and take an X latch there and attempt to insert. If split, re-descend with an SX latch on the index tree, go down to the leaf, perform the insert and the split.

## BTree Descent

### PostgreSQL

If thread A is holding X-latch on root, and identifies a missing child page it will:

  - calls the buffer pool's read operation to get a pinned buffer. This returns a pinned frame with an I/O in progress flag set
  - the thread will give up the X-latch on root
  - I/O will complete and the flag is cleared and the thread can continue

### InnoDB

Release all latches, do I/O, then restart the root-to-leaf traversal. This is simpler but slower. Optimized by checking the LSN (log sequence number) of ancestor pages on the way back down — if nothing changed, you can skip re-validating those nodes.

InnoDB uses a version of this in its optimistic path via the modify clock: a per-page counter that increments on every structural change. After re-acquiring a latch, the thread checks if the clock changed; if so, it falls back to a full re-descent.

## Links

1. [InnoDB BTree Concurrency](https://baotiao.github.io/2024/06/09/english-btree.html)