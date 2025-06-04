## Indexes

- Indexes are used to look up records in a table quickly.
- Indexes can be on multiple fields. If an index contains all fields within a query, it is called a covering index, although book doesn't cover this.
- If an index contains multiple fields, order of fields matters for queries. Multiple indexes defining different orders of fields are created.
- On a table with indexes, every write = write to table + write to each index. This adds overhead.
- Indexes typically containing the search key and the row identifier for the row to which that search key belongs.
- The big take away from the usefulness of an index on field A is proportional to the number of different values of A in the table. Statistics are important here.

    Take the example from the book. If the number of departments is high, then the number of students per department is low, in which case you will end up reading fewer blocks via an index search even if each student belonging to the same department is in a different block of the students table.

    This is why indexes on PK's are useful. Each PK is unique.

    Indexes become useless if distinct values < records per block
- Book doesn't really talk about clustered indexes or tables where the leaf pages of the index are the data pages.
- Multidimensional indexes aren't really covered.

- Book talks about 3 types of indexes
  - Static hash indexes
  - Extendible hash indexes
  - B-Trees

### Static Hash Indexes
Consider a hash table with a bunch of bucket, and each bucket contains some blocks. The cost here is the number of blocks to search.

So, if a bucket contains 3 blocks, it would be the cost of accessing 3 blocks from disk.
Generally, `cost = B/N` (blocks in index / number of buckets)

### Extendable Hash Indexes
- CMU Course to build extendible hash index (https://15445.courses.cs.cmu.edu/fall2023/project2/)
- Improves over static hash indexes by allowing a dynamic directory which can grow and allows splitting of buckets.
- Essential mechanism is that there are two depths - global and local. 
- The global depth is used by the top level directory and each individual bucket uses a local depth. 
- The invariant you have to maintain is that `global_depth >= local_depth`
- Indexing mechanism is to use the rightmost global depth number of bits to determine which bucket to hash an entry into. 
- Makes growing directory easy because newly assigned buckets are mirrors of the original buckets with some bits flipped.
- Does not work when index contains more records with the same value than can fit in a block. This will require overflow blocks (although B-trees also have overflow blocks, so I'm not sure what the problem is).
- Constant time access (2 block accesses)

### B-Trees
- CMU Course to build b-tree index (https://15445.courses.cs.cmu.edu/spring2025/project2/)
- Originally called b+-trees, b-trees were precursors to this although everyone uses terms interchangeably now. Binary trees are not b-trees.
- High fan-out makes them ridiculously efficient for searching. 2-level b-tree can hold 2 billion records.
- High fan-out also makes them great for disk seeks because they reduce the amount of I/O you incur.
- Book claims that algorithm for coalescing half-full blocks is error-prone and rarely implemented. Latter half is hard to believe. Claim is that deletions are usually followed by inserts.
- Implementation of b-tree misses out on sibling pointers and doesn't support range scans, only point queries.
- Implementation puts index records with same search key value in overflow block in case block can't fit them.
- Logarithmic block access

### Query Planning with Indexes
- Database can use indexes for both selection (WHERE clauses) and joins
- Query planner must choose between table scan vs index scan based on selectivity
- All indexes must be maintained during data modifications (insert/update/delete)

### Index Selection
Tweet basically talks about index selection.

Query - `select * from issue where creatorID = $1 order by created desc limit 20;`. Assume you have indexes on both `created_by` and `created_at`, so which index do you use for this query?
If you use the index on `created_by` you get the records created by a user but you don't get them sorted and limited correctly. On the other hand, if you use the index on `created_at` you get the records sorted but you have to potentially go through a lot of records.
For a user with a lot of records, use second index and vice versa.

[tweet link](https://x.com/aboodman/status/1929458643306790916)