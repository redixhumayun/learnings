# Overview

Roughly two models:
* pull based model (volcano model)
* push based model

The volcano model is the predominant one. It works on the assumption that the driving force is the top level consumer which forces each of the underlying iterators to move forward. This proceeds recursively.

```
      +------------+
      |  Consumer  |
      +------------+
            ^
            |
      +------------+
      |  Filter    |
      +------------+
            ^
            |
      +------------+
      |   Scan     |
      +------------+
```

The push based model seems a more recent one. Here the driving force starts from the "bottom", with each node forcing its parents to take ownership of a result set.

```
      +------------+
      |  Consumer  |
      +------------+
            |
            v
      +------------+
      |  Filter    |
      +------------+
            |
            v
      +------------+
      |   Scan     |
      +------------+
```

### Notes From Volcano Paper (1994 Paper Dealing With Execution)
Broadly, two forms of data flow defined in Volcano:
- Demand driven data flow (lazy, iterator style with lazy evaluation) [also called pull based flow]
- Data driven data flow (eager with immediate materialization) [also called push based flow]
[This](https://justinjaffray.com/query-engines-push-vs.-pull/) is a good resource to read more about these two styles.

1. Volcano is an extensible query execution engine. What makes it extensible:
  a. All operators (and meta-operators) follow this iterator protocol - `open()`, `next()`, `close()`
  b. Support functions to provide independence from the data model
  c. Support functions/parameters to provide policy/mechanism separation
  d. Support functions to allow predicate matching, comparison, hashing etc.
2. For materializing intermediate results, Volcano uses special devices called virtual devices. Think of these as pages that exist only in the buffer pool, but aren't necessarily written to disk.
3. An optimization for Join style operators is to pass around RowID pointers instead of duplicating records and materializing the result immediately.
4. Joins are implemented in `one-to-one` and `one-to-many` operators.
  a. Uses either hashing or sorting
  b. Hashing uses [hybrid hash join](https://www.youtube.com/watch?v=GRONctC_Uh0) to handle large inputs
  c. Sorting uses [external merge sort](https://opendsa-server.cs.vt.edu/ODSA/Books/CS3/html/ExternalSort.html) for large inputs
5. Volcano supports the following meta operators:
  - `choose-plan` (Allows Volcano to defer choice of query plan for prepared statements where late binding happens)
  - `exchange` (Allows Volcano to support intra-operator & inter-operator parallelism)
6. The `exchange` operator supports both:
  - Vertical Parallelism
  - Horizontal Parallelism
  Vertical parallelism allows running operators at different levels of the query tree within their own thread/process and `exchange` managers coordination between these levels.
  Horizontal parallelism allows running multiple instances of an operator at the same level within its own thread/process on a partition input
  It is while enabling these different forms of parallelism that Volcano switches from demand-driven dataflow to data-driven dataflow

### Notes From Cascade
The key concepts in Cascade - groups, expressions, rules & tasks

#### Groups & Expressions
Groups are a set of logically equivalent logical and physical expressions which produce the same result. Look at the table below as an example

```text
GROUP: {AB}
┌────────────────────────────┬────────────────────────────────┐
│     Logical Expressions    │      Physical Expressions      │
├────────────────────────────┼────────────────────────────────┤
│ Join(A,B)                  │  HashJoin(A,B)                 │
│ Join(B,A)                  │  HashJoin(B,A)                 │
│                            │  MergeJoin(A,B)                │
│                            │  NestedLoopJoin(A,B)           │
│                            │  Sort(MergeJoin(A,B))          │
└────────────────────────────┴────────────────────────────────┘
```
The big difference from Volcano is that at a specific point in the query tree, not all logically equivalent plans are generated all at once. Instead, in Cascades a logical plan is generated and the query tree is followed further down. Cascades tends towards early cost materialization.
To put it more concisely, Volcano separates optimization into two phases - logical & physical. Cascade doesn't make this distinction.

#### Rules
Rules are an abstraction for the kinds of transformations that can be applied to expressions and rewrite them. They can do logical -> logical or logical -> physical mappings
Rules consist of a pattern (antecedent) that they match, and a substitute expression they produce when applied.. Custom rules can be registered.

#### Tasks
Tasks are an abstraction for how transformations actually happen to the query plan. Cascade maintains a heap allocated priority queue and dumps tasks into them. The value of the promise associated with a rule is what dictates where the task lands in the priority queue [cost = estimated cost so far + cost for rule].
This heap allocated task queue is what allows for multi-threaded query plan exploration as long as task queue and memoization hash map are properly synchronized.

This image is a great overview of how Volcano works as an execution engine and fits into the pipeline with an optimizer, buffer pool etc.

![](/assets/img/query_engines.png)

A one page infographic generated by ChatGPT
![](/assets/img/volcano_infographic.png)

### Estimate Costs Of A Query Plan

At a very high level, estimating this cost is about the cost of I/O, specifically blocks. Cost estimation assumes that every block has to be fetched from disk or written to disk, thus giving worst case cost estimates.

#### Scanning
```text
TABLE SCAN
     |
   STUDENT  (4500 blocks)
```
Cost = 4500 blocks because every block has to be read.

#### Filtering
```text
 SELECT(GradYear = 2022)
         |
      STUDENT
```
Cost = 4500 blocks because every block has to be read before it can be filtered

#### Projecting
```text
 PROJECT(SId, SName)
         |
      STUDENT
```
Cost = 4500 blocks because every block has to be read before fields are selected

Here's a chart from Database Design & Implementation showing a table explaining the costing of different kinds of scans
![](/assets/img/query_cost_analysis_table_abstract.png)
![](/assets/img/query_cost_analysis_table_concrete.png)

#### Materialization
Every materialization requires scanning a table and then writing to a temp table
```text
PREPROCESSING (write to temp table)
     +
 SCANNING (read temp table)
```
An actual abstract plan
```text
     T1
     |
 MATERIALIZE
     |
     T2 (B blocks)
```
Preprocessing Cost = B(read) + Cost(T2), where Cost(T2) represents writing into temp table
Scanning Cost = B(read), which is the cost of reading from the temp table
Cost = Preprocessing Cost + Scanning Cost

A concrete version of the above plan
If T2 reads ENROLL (50,000 blocks), and after filter, the result is 3572 blocks:
```
Materialize(Filter(Grade="A"))
         |
       ENROLL
```
Cost:
Read ENROLL: 50,000
Write temp: 3572
Read temp: 3572
Total: 50,000 + 3572 + 3572 = 57,144 block accesses

#### Sorting
Sorting is physically done by the external merge sort algorithm.

The cost of this is split up into splitting and merging

*Splitting cost*
The splitting cost is = 2B → read B blocks + write B sorted blocks
Note: This is different from in-memory merge sort which has a splitting cost $\log_2(n)$

*Merging cost*
($\log_k(r)$ - 1) since the final merge is done by the scanner.
Each merge iteration reads B blocks (from k input runs) and writes sorted B blocks
2B * ($\log_k(r)$ - 1)
Finally, add B blocks for the final scan

So, 2B(split) + 2B * ($\log_k(r)$ - 1) + B

## Links
1. [Volcano paper](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf)
1. [Andy's Lecture On Volcano](https://www.youtube.com/watch?v=kguocoOxXKM)
1. [Cascades paper](https://15799.courses.cs.cmu.edu/spring2025/papers/05-cascades/graefe-ieee1995.pdf)
1. [Andy's Lecture On Cascades](https://www.youtube.com/watch?v=bjOIbEg0Stc)
1. [Microsoft SQL Server Lecture On Cascades](https://www.youtube.com/watch?v=pQe1LQJiXN0)
1. [Push vs Pull-Based Loop Fusion In Query Engines](https://arxiv.org/pdf/1610.09166)
1. [Justin's Post On Push vs Pull Engines](https://justinjaffray.com/query-engines-push-vs.-pull/)
1. [How Query Engines Work](https://howqueryengineswork.com/02-apache-arrow.html)
1. [Blog Post On Building An Optimizer](https://www.skyzh.dev/blog/2025-02-06-optimizer-lesson-01/)
1. [Timely Dataflow](https://timelydataflow.github.io/differential-dataflow/introduction.html)
1. [ChatGPT conversation](https://chatgpt.com/c/67fa5318-354c-8005-ba34-73bedcee482a)