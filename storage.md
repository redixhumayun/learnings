# Understanding LSM Trees and Their Optimization for Cloud Storage

This document serves as a set of notes and a guide to understanding Log-Structured Merge (LSM) trees, their performance characteristics, and particularly how they are being optimized for use with cloud/blob storage like S3.

## 1. Introduction to LSM Trees

LSM trees are data structures designed to provide high write throughput, making them suitable for write-intensive workloads. They achieve this by deferring the organization of data.

### 1.1. Core Mechanics

*   **Memtable:** All incoming writes (inserts, updates, deletes) are first written to an in-memory data structure, typically a sorted one like a skip list or a red-black tree. This is called the **memtable**.
    *   *Sorted Order:* Keys can be sorted upon insertion into the memtable (potentially slowing down individual writes slightly but simplifying flushes) or sorted when the memtable is "frozen" and prepared for flushing (keeping individual write latency minimal).
*   **SSTables (Sorted String Tables):** When a memtable becomes full (or after a certain time), it is "frozen" (made immutable) and flushed to persistent storage as an **SSTable**.
    *   SSTables are sorted by key and are immutable once written.
    *   This flush operation is typically a sequential write, which is efficient for disks.

```ascii
Writes -> [WAL] -> Memtable (In-Memory, Sorted)
                     |
                     | (Memtable Full / Time)
                     V
                  Flush to Disk/Blob Storage
                     |
                     V
           [SSTable 1 (Immutable, Sorted)]
```

### 1.2. Write-Ahead Log (WAL)
To ensure durability in case of crashes, writes are typically first appended to a Write-Ahead Log (WAL) before being acknowledged to the client or fully committed to the memtable.

#### Durability vs. Efficiency:
`fsync` per transaction: The WAL can be flushed (`fsync`'d) to disk after every write or transaction. This is highly durable but can be inefficient due to frequent synchronous disk operations.

Checkpointing/Batching: The WAL can be flushed periodically (e.g., after a certain number of log entries or a time interval). This is more efficient for write throughput but risks losing the most recent writes if a crash occurs before the last flush.

WAL Management: The WAL can be truncated or a new one started after its corresponding memtable data has been successfully flushed to an SSTable and is considered durable. For example, RocksDB may manage WALs in a way that once all data covered by a WAL file is persisted in SSTables, that WAL file can be archived or deleted. (Reference: [RocksDB Overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview#3-high-level-architecture))

## 2. Compaction: The Heart of LSM Maintenance
Over time, many SSTables accumulate on disk. Compaction is a background process crucial for LSM trees. It merges SSTables to:
* Reduce the number of files a read has to check.
* Remove deleted or overwritten (stale) data, reclaiming space.
* Maintain the sorted order across larger segments of data.

### 2.1. Compaction Strategies

There are several strategies for compaction, each with different trade-offs. The two most common are Leveled and Tiered (or Size-Tiered). (Reference: [RocksDB Compaction](https://github.com/facebook/rocksdb/wiki/Compaction))

### 2.1.1. Leveled Compaction
This is a popular strategy used by systems like RocksDB and LevelDB.
(Reference: [RocksDB Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction))

#### Structure

- **$L_0$ (Level 0):**  
  SSTables flushed directly from the memtable reside here. Files in $L_0$ can overlap in their key ranges.

- **$L_1,\; L_2,\; \ldots,\; L_n$:**  
  For these deeper levels, SSTables within the same level are guaranteed not to overlap in key ranges. Each level $L_i$ (for $i \ge 1$) represents a single, sorted run of data, though it might be physically stored in multiple non-overlapping SSTable files. The total size of data in $L_{i+1}$ is typically much larger (e.g., 10x) than that in $L_i$.

#### Process

When L0 fills up (reaches a certain number of files), one or more SSTables from L0 are selected.

These selected L0 SSTables are merged with all SSTables in L1 that their key ranges overlap with.

The result of this merge forms new SSTables in L1. Old L0 files and the L1 files that participated in the merge are discarded.

This process cascades: If merging into L1 causes L1 to exceed its size target, one or more SSTables from L1 are selected and merged with all their overlapping SSTables in L2, and so on.

**Leveled Compaction Example ($L_0 \rightarrow L_1$):**

- **$L_0$**:  
  $\displaystyle \left[ \text{SST}_A\,(a\text{-}e) \right] \quad \left[ \text{SST}_B\,(d\text{-}g) \right]$  
  *Overlapping keys allowed*

- **$L_1$**:  
  $\displaystyle \left[ \text{SST}_X\,(a\text{-}c) \right] \quad \left[ \text{SST}_Y\,(h\text{-}m) \right] \quad \left[ \text{SST}_Z\,(n\text{-}p) \right]$  
  *Non-overlapping within $L_1$*

- **Compaction Trigger:**  
  $L_0$ is full. Pick $\text{SST}_A$ and $\text{SST}_B$.

- **Identify Overlap:**  
  $\text{SST}_A$ and $\text{SST}_B$ overlap with $\text{SST}_X$ in $L_1$.  
  $\text{SST}_A$ and $\text{SST}_B$ do **not** overlap with $\text{SST}_Y$ or $\text{SST}_Z$.

- **Merge:**  
  Read $\text{SST}_A$, $\text{SST}_B$, and $\text{SST}_X$.

- **Write:**  
  Write new $\text{SST}_{X'}$ (containing merged data $a$–$g$) into $L_1$.  
  $\text{SST}_Y$ and $\text{SST}_Z$ in $L_1$ remain untouched by this specific operation.

- **Resulting $L_1$ (simplified):**  
  $\displaystyle \left[ \text{SST}_{X'}\,(a\text{-}g) \right] \quad \left[ \text{SST}_Y\,(h\text{-}m) \right] \quad \left[ \text{SST}_Z\,(n\text{-}p) \right]$

**Question:** What happens when there isn’t an overlapping range available in the level below?

**Answer:** If an SSTable (or a group of SSTables) from $L_i$ is chosen for compaction and its key range does not overlap with any existing SSTables in $L_{i+1}$ (e.g., $L_{i+1}$ is empty or all existing files are outside the key range), the chosen SSTable(s) are effectively “moved” to $L_{i+1}$ and become the first file(s) in that new range for $L_{i+1}$.

### 2.1.2. Tiered Compaction (Size-Tiered Compaction)

#### Structure

The LSM tree is organized into "tiers" (which can also be called levels, but the behavior is distinct).

Within each tier, you can have multiple SSTables, and their key ranges can overlap.
Tiers typically contain SSTables of similar sizes.

#### Process

When a tier accumulates a certain number of SSTables (e.g., N similar-sized SSTables), all N SSTables in that group are read and merged into a single, new, larger SSTable.

This new, larger SSTable might remain in the same tier (if the tier allows files of that new size) or "graduate" to the next tier, which is designated for larger files.

**Tiered Compaction Example:**  
*(Tier 1 compacts when it has $\geq 3$ similar-sized files)*

- **Tier 1:**  
  $$
  \left[ \text{SST}_1\,(a\text{-}e) \right] \quad 
  \left[ \text{SST}_2\,(d\text{-}h) \right] \quad 
  \left[ \text{SST}_3\,(b\text{-}f) \right] \quad 
  \left[ \text{SST}_4\,(k\text{-}m) \right]
  $$

- **Compaction Trigger:**  
  Tier 1 is full; the first 3 files (of similar size) are selected.

- **Merge:**  
  Read $\text{SST}_1$, $\text{SST}_2$, and $\text{SST}_3$.

- **Write:**  
  Write a new SSTable, $\text{SST}_{123\_merged}$ (covering $a$–$h$), into a higher tier (or replace them in Tier 1 if policy allows).

- **Resulting Tiers (simplified):**
  - **Tier 1:**  
    $$
    \left[ \text{SST}_4\,(k\text{-}m) \right]
    $$
  - **Tier 2:**  
    $$
    \left[ \text{SST}_{123\_merged}\,(a\text{-}h) \right]
    $$

## Performance Characteristics & Amplification Factors

Amplification factors help quantify the overheads associated with storage engines.
(Definitions based on TiKV Deep Dive: [B-Tree vs LSM-tree](https://tikv.org/deep-dive/key-value-engine/b-tree-vs-lsm/))

Write Amplification (WA): (Total Bytes Written to Disk) / (Bytes of Logical Data Written to DB) - Measures how many times data is physically written to storage for each logical byte written by the application.

Read Amplification (RA): (Total Bytes Read from Disk) / (Bytes of Logical Data Returned by Query) - Measures how much extra data needs to be read from storage to satisfy a query.

Space Amplification (SA): (Total Bytes on Disk) / (Bytes of Logical Live Data in DB) - Measures how much disk space is used beyond the actual live data.

### 3.1. Write Amplification (WA)
#### LSM Trees
Initial writes (memtable -> $\text{L}_0$) are efficient (low WA for that step).

Compactions are the primary source of WA in LSMs. Data is re-read and re-written as it moves through levels/tiers.

Leveled: Tends to have higher WA because data in $\text{L}_{i+1}$ is rewritten when merging with even a small amount of data from $\text{L}_i$.

Tiered: Tends to have lower WA because data is typically merged in larger, similarly-aged batches, reducing repetitive rewrites of the same data across sub-level compactions.

#### B-Trees
The debate: Your note mentioned an article suggesting B-trees might have lower WA. However, as [this tweet](https://x.com/sunbains/status/1760730549264732361?s=20) points out, it's nuanced.

If a B-tree performs in-place updates directly to disk (no buffer pool or WAL for batching), WA can be low for updates but high for inserts causing page splits.

With a buffer pool, many updates happen in memory. The actual disk WA depends on how dirty pages are flushed. A small change to a large page, when flushed, writes the whole page.

Conclusion: It "depends" on the specific B-tree implementation, workload (update-heavy vs. insert-heavy), and LSM compaction strategy. However, LSMs are designed to optimize for ingest by deferring merge work, even if it means higher WA during background compactions.

### 3.2. Read Amplification (RA)
#### LSM Trees
A read might need to check:

* The active memtable.
* SSTables in L0 (multiple, as they can overlap).
* SSTables in L1, L2, ..., Ln.

Bloom filters significantly reduce the need to read data blocks from every SSTable, but checking the filters themselves still incurs some overhead.

Leveled: Generally has lower RA. For L1 and deeper, only one SSTable per level needs to be consulted (due to non-overlapping key ranges). Total checks = Memtable + all L0 files + 1 file per deeper level.

Tiered: Generally has higher RA. Multiple SSTables within each tier can overlap, so a read might have to check all SSTables in a tier, and then all SSTables in the next tier, and so on.

#### B-Trees
Generally have very low RA.

A read operation involves traversing the tree from the root to a leaf. Suppose each node (block) holds $B$ bytes, and there are a total of $N$ bytes of data. Then, the estimated number of nodes is approximately: $\frac{N}{B}$

The height of the tree—which corresponds to the number of disk seeks in a cold cache scenario—is roughly

$$
O\Bigl(\log_C\Bigl(\frac{N}{B}\Bigr)\Bigr)
$$

where $C$ is the fanout (i.e., the number of children per node).

With an empty cache, this translates to a predictable number of disk seeks.

### 3.3. Space Amplification (SA)
#### LSM Trees
Can have higher SA due to:

* Stale data (old versions of keys, deleted data) waiting for compaction to remove it.
* Multiple SSTables containing different versions of the same key until compacted.

Leveled: Tends to have lower SA because compactions are more aggressive in merging overlapping data and reclaiming space.

Tiered: Can have higher SA as stale data might persist longer across multiple files within a tier before that entire tier is compacted.

#### B-Trees
Generally have lower SA. In-place updates mean old versions are overwritten (though fragmentation within pages can lead to some wasted space). WALs and undo logs can temporarily increase space usage.

### LSM Trees and Hardware Considerations

#### SSDs (Solid State Drives):

Pros for LSMs:
* High Write Throughput: SSDs excel at random writes compared to HDDs, which aligns well with the flushing of many small SSTables and the random reads/writes during compactions. LSM's sequential flushes from memtable are also fast on SSDs.
* Low Read Latency: Crucial for LSM reads that might need to access multiple SSTables. SSDs' fast random read access significantly benefits LSM read performance.

Cons/Considerations:
* Write Endurance: SSD cells have a limited number of P/E (program/erase) cycles. The higher write amplification of some LSM strategies (especially Leveled) can wear out SSDs faster. (Reference: SSD Write Endurance)
* Cost: SSDs are generally more expensive per GB than HDDs.

#### HDDs (Hard Disk Drives):
* LSMs can work on HDDs, but performance is often limited by the HDD's slow random access speeds, especially for reads and compactions involving many small files. The sequential nature of memtable flushes is a better fit for HDDs than completely random B-tree updates.
* General Fit: LSM trees, due to their write patterns (sequential flushes, potentially random reads/writes during compaction) and read patterns (potentially multiple random accesses), benefit greatly from the performance characteristics of SSDs.

### Optimizing LSM Trees for Cloud/Blob Storage (e.g., S3)
Using cloud blob storage like S3 as a backend for LSM tree engines introduces new challenges and opportunities for optimization. This section draws heavily from our previous conversation.

#### 5.1. Key Characteristics of Blob Storage (S3)
* High Latency (First Byte): GET (read) and PUT (write) operations have significantly higher latency than local SSDs.
* High Throughput: Can sustain high aggregate bandwidth once connections are established.
* "Infinite" Scalability & Durability: Pay-as-you-go, highly durable.
* Object-based, Not Block-based: Operations are on whole objects (files). Random writes within an object are not natively supported (updates mean rewriting the whole object). Appends are now supported by S3.
* Cost Model: Costs per GET/PUT/LIST operation, per GB stored, and per GB transferred out.

#### 5.2. Why Standard LSMs Need S3-Specific Optimizations

A naive LSM implementation treating S3 like a regular disk would suffer:
High Read Latency: Each SSTable check (if the file is on S3) incurs an S3 GET latency.

Expensive Compactions: Compactions involve many GETs (to read input SSTables from S3) and PUTs (to write new SSTables to S3).

During compaction on S3, compute nodes read SSTables (or blocks) from S3 (GET requests), perform the compaction in their memory/local disk, and then write the new, compacted SSTables back to S3 (PUT requests). They might keep a local cached copy of new SSTables for faster subsequent reads.

```text
Compute Node (Instance)                      Object Storage (S3)
+--------------------------------+           +--------------------------+
| Local Cache/SSD (Optional)     |           | SSTable_A (Old)          |
|                                |           | SSTable_B (Old)          |
|       COMPACTION LOGIC         |<--READ----| (Input for Compaction)   |
|       (CPU, Memory)            |  (S3 GETs)|                          |
|                                |----WRITE-->| SSTable_C (New,Compacted)|
|                                |  (S3 PUTs)|                          |
+--------------------------------+           +--------------------------+
```
#### 5.3. Discussed Optimization Strategies for LSMs on S3
##### 5.3.1. Local SSD Primary, Blob for Backup/Replication (e.g., "SlateDB Model")

Concept: The LSM tree operates primarily on fast local SSDs on the compute instance. SSTables are periodically copied/moved to S3 for durability, backup, or bootstrapping other nodes.

Why: Keeps active operations fast, leverages S3 for cheap, durable storage.
Limitations: Active dataset size limited by local SSD. S3 isn't an active tier of 
the LSM for live queries on cold data.

##### 5.3.2. Tiering vs. Leveling for S3-Resident Levels

Tiered Compaction on S3:

Pros:
* Lower Write Amplification on S3: Results in fewer S3 PUT operations per logical byte compacted because larger batches of data are merged at once. This is crucial given per-operation costs of S3.

Cons:
* Higher Read Amplification: More S3 objects might need to be inspected (or their metadata/Bloom filters checked via GETs) for a single read.

Leveled Compaction on S3:

Pros:
* Lower Read Amplification: Fewer S3 objects to check per read (for L1+).
* Lower Space Amplification: More efficient use of S3 storage.

Cons:
* Higher Write Amplification on S3: Can lead to more GETs and PUTs as data is rewritten more frequently across levels.

The Gist (from our chat):

Object Storage (S3): Prefer Tiered compaction to reduce GET/PUT costs during compaction (lower S3 WA), accepting potentially higher RA.

Local SSD (Cache/Hot Tier): Prefer Leveled compaction to minimize space (lower SA, delaying spill to S3) and provide good read performance (lower RA), accepting higher WA on the SSD itself.

#### 5.3.3. Disaggregated LSM Model (Shared WAL, S3 as Source of Truth, Local SSD Caches)

This is a more advanced architecture (e.g., Neon DB).

Components:
* Shared WAL on S3: All writes are durably logged to a WAL on S3 (using S3's conditional PUTs/CAS for consistency). This is the ultimate source of truth for writes.
* Primary LSM on S3: SSTables are constructed from the WAL and stored on S3, forming the complete, durable LSM tree. This might use Leveling for space efficiency and better scan performance on S3 data itself.
* Local SSD LSM (Cache): Compute nodes maintain a local LSM tree on their SSDs, caching a subset (e.g., a key range or hot data) of the data from the S3 LSM. This local cache might use Tiering for fast ingest.

Flow:

```text
Shared Blob Storage (S3)
+-----------------------------------------------------------------+
| Shared WAL (Appended using S3 CAS)                              |
|   [log_entry1, log_entry2, ...]                                 |
|                                                                 |
| Blob LSM (Source of Truth - e.g., Leveled)                      |
|   L0_blob: [SST_blob_A] --> L1_blob: [SST_blob_C] --> ...       |
+-----------------------------------------------------------------+
    ^    |                                  ^      |
    |    | WAL tailing / SST pull           |      | WAL tailing / SST pull
    |    V                                  |      V
Compute Node 1 (Local SSD)              Compute Node 2 (Local SSD)
+-----------------------------+         +-----------------------------+
| Local Memtable              |         | Local Memtable              |
| Local LSM (Tiered Cache)    |         | Local LSM (Tiered Cache)    |
| (e.g., KeyRange1 / Hot Data)|         | (e.g., KeyRange2 / Hot Data)|
+-----------------------------+         +-----------------------------+
```

Why: Scalable (compute/storage separate), cost-effective, performant for hot data. Allows optimizing S3 LSM and local SSD LSM differently. Compactions on the S3 LSM don't directly evict from the local SSD LSM cache.

#### 5.3.4. Effective Block Caching for Cloud LSMs

The Problem: As Miller noted, "having 9/10 blocks cached for some lookup means you're still paying the same latency as if you had 0 blocks cached" if that 1/10th uncached block requires an S3 GET.

Solutions:
* Cache entire SSTable index/filter blocks locally always.
* Cache entire small SSTables or large, contiguous segments of larger SSTables.
* Key-range affinity for compute nodes to keep their responsible data fully local on SSD.
* Aggressive prefetching from S3.

#### 5.3.5. Why B-Trees are Generally Not a Good Fit for S3 (as primary mutable store)
B-Trees rely on in-place updates. S3 objects are immutable. Updating a B-Tree node on S3 would mean rewriting the entire object representing that node, and potentially all its parent nodes. This is highly inefficient.

LSMs, with their append-only nature and immutable SSTable files, map much more naturally to S3's object storage model.

# Torn Writes

Torn writes are when a page isn't entirely written to disk and only part of it is written. The cause of this is that the page size is greater than the sector size of the disk (which is frequently the case, since sector sizes are typically 512 bytes).

Most of this content is taken from [this great post](https://transactional.blog/blog/2025-torn-writes) from Miller.

## Mitigration Strategies
1. Detection only - checksum or bits set on page header (cheapest but no prevention, only detection)
2. Sector-sized pages - ensure page size = sector size (cheap but causes performance issues since this affects branch out factor of page)
3. Log full page - write full page to WAL as SQLite does (2x write cost)
4. Log page on first write - write the first copy of the page after checkpoint like PostgreSQL (optimization on logging full page)
5. Double-Write Buffer - write to a "scratch area" on disk, fsync that and then write actual data pages like MySQL (doesn't incur 2x I/O because of sequential batches)
6. CoW - each update to a page creates a new page like LMBD (limited concurrency because all updates contend at the root)
7. Copy on first write - similar to 4 where first write to a page after checkpoint copies it
8. Atomic block writes - depend on hardware atomic writes spanning multiple blocks

### Resources
* [Chat with Gemini about tiered storage with LLM's](https://aistudio.google.com/prompts/1pQKJd-4oLRa5I4M69MHxVfO3BZgDVn92)
* [Optimistic B-trees](https://cedardb.com/blog/optimistic_btrees/)
* [RocksDB Wiki: Compaction](https://github.com/facebook/rocksdb/wiki/Compaction)
* [RocksDB Wiki: Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)
* [TiKV Deep Dive: B-Tree vs LSM-tree](https://tikv.org/deep-dive/key-value-engine/b-tree-vs-lsm/)
* [Torn Write Detection And Protection](https://transactional.blog/blog/2025-torn-writes)