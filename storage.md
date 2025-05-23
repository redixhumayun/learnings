## Performance Characteristics of LSM trees

How do LSM trees work?
Do append only memtables. Memtables are frozen to an SST and then stored on disk. The sorted order of keys is either done at insertion time or(slower writes) or when freezing the memtable to an SST (doesn’t affect writes). At some later point, SST’s are compacted with the compaction strategy being different. 

When doing writes to an LSM engine, data is written to the WAL first and then to the memtable. The WAL could either be flushed on every write (transaction) (fscynced, inefficient but durable) or checkpointed after a certain number of logs (efficient but not durable). The WAL can be flushed and re-created when a single memtable is converted to an SSTable. [Note: This doesn’t have to be default behaviour but it seems like this is how RocksDB does it]
How does compaction work?

Compaction strategies can differ -> elaborate on compaction strategies

Levelled compaction seems most popular - see here for more details
Essentially, each level has a certain number of SST files. When the number of files grows beyond the number allowed at that level, one file is picked and merged with the overlapping range of files below it. (Question: what happens when there isn’t an overlapping range available? How is the file chosen?)

Amplification (comparison with B-tree)

Based on definitions from here
Write amplification = data written to disk / data written to DB (number of times data is written to disk / data written to DB?)
Read amplification = data read from disk / data returned for query
Space amplification = data on disk / logical data on DB


Write Amplification
My initial assumption was that LSM trees have more write amplification than B-trees because of append only characteristics as opposed to B-tree update in place. But, this article disagrees. However, when I tweeted that out, I found this tweet which said that it’s slightly tricker to answer.

Assuming a B-tree that writes to disk on every update (assuming no WAL & no buffer pool) every single write would result in a disk write. For an LSM-tree every write goes to a WAL (no fsync per write) and then memtable and then gets frozen to SST. Background compaction occurs later which leads to greater write amplification.My answer is that it really seems to depend on the workload. For instance, what happens in the case where a workload can fit entirely in the buffer pool and no disk writes are required unless fsync is manually called at some constant interval?

Read Amplification
Here, in my mind it seems clear cut that B-trees would exhibit less read amplification. Assuming N as the size of the data and each node has C children and the block size is B this gives us N/B nodes and O(log(N/B)) to the base C.

So, assuming that we start with an empty cache, it would still require only that many disk seeks.

I’m not sure how to quantify the number of reads for an LSM tree in a formula but it would depend on the number of levels and the number of files and size of each file at a particular level. In the worst case scenario, a read would propagate all the way down to the last file in the last level.

Space Amplification

Because it does In-place updates, I think a B-tree has less space amplification than an LSM-tree. Space amplification in an LSM-tree can be reduced by compaction.

Hardware

Since LSM-trees offer higher write throughput I think SSD’s would be a better fit because SSD’s offer the ability to do writes anywhere on disk without the overhead of any mechanical parts moving. However, I also know that each SSD cell has a limited number of times that it can be written to so that needs to be balanced out against the higher cost of SSD’s as compared to HDD’s (reference: https://superuser.com/questions/1107320/why-do-ssd-sectors-have-limited-write-endurance) (I remember reading about this in Alex Petrov’s Database Internals book)


For LSM-trees since reads are done randomly on disk to accumulate all the SST files to iterate over them to find a key SSD’s would still be better because again no mechanical moving parts.

# Resources

#### B-Trees
1. [Optimistic B-trees](https://cedardb.com/blog/optimistic_btrees/)