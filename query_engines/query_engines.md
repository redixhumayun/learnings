# Overview

Query engines can be divided into two phases - Query Execution and Query Optimization.

## Query Execution

Roughly two models:
* pull based model (volcano model)
* push based model (morsel model)

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

## Query Optimization

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

### Query Equivalence
Queries can be mathematically transformed as long as they can be proven to be equivalent. Look at the concept of graph equivalence for this.

Here are some general rules for performing transformations

```text
Product commutativity: A × B ≡ B × A
Product associativity: (A × B) × C ≡ A × (B × C)
Selection splitting: σ(p1 AND p2)(R) ≡ σ(p2)(σ(p1)(R))
Selection pushdown: σ(p)(A × B) ≡ A × σ(p)(B) [when p only uses B's fields]
```

### Heuristics Based Optimizer
Heuristics based optimizers use some simple heuristics to optimize queries instead of using statistics.

1. Selection Pushdown

      Push selections down as far as possible. This reduces data volume early in the pipeline

2. Replace Select-Products With Joins

      Select-Product combinations are where the output of a product node is fed into a select node with some predicate. This is equivalent to a join node with the same predicate. This also helps reduce data volume early.

3. Consider Only Left Deep Query Trees

      It is not entirely clear to my why this helps. Only reason I can think of is that our join algorithms are written in a way that benefits left deep query trees. [This link](https://www.cs.emory.edu/~cheung/Courses/554/Syllabus/5-query-opt/left-deep-trees1.html) suggestst the same.

      Another reason is that by considering only left deep query trees, we need to consider only n! plans vs (2n!)/n! plans

4. Each Table In Join Order Should Join Only Previously Chosen Tables, If Possible

      That is, ideally, the only product nodes in a query tree should correspond to a join node.

      This seems to be a rehash of heuristic 2.

5. Join Order Selection Heuristics
      a. Choose table producing smallest output
      b. Choose most restrictive predicate

      5b will reduce the number of intermediate records so a big win

6. Use index select if possible to implement a select node

7. Implement a join node according to following priority
      - use index join
      - use hash join if one of the input tables is small
      - use merge join

8. The planner should add a project node as the child of each materialized node, to remove all fields that are not required

#### Choosing Join Order
A join order can be chosen by heuristics 5a and 5b above. However, it can also be chosen by exhaustive enumeration which will be more accurate.

For a set of tables like `(STUDENT, ENROLL, SECTION, COURSE, DEPT)` use dynamic programming to compute the correct order in which to join them.

#### Predicate Pushdown Quick Rules

##### AND (conjunctive): 
Push down any conjunct that applies to the target schema; keep the rest above.

Safe because (A ∧ X) ⇒ A.

Example: On R(a), from a=1 ∧ b=2 push a=1 to R.

##### OR (disjunctive): 
Push down only if all disjuncts apply to the same schema; otherwise push none.

Unsafe to partially push because A ∨ X ⇏ A.

Example: On R(a), from a=1 ∨ b=2 push nothing (unless both a and b are in R).

##### NOT
Push down only if its inner predicate fully applies to the schema.

Optionally normalize with De Morgan’s laws before pushdown.

##### Field = Constant
Push down if the field belongs to the schema.

Example: R(a) with a=5 → push to R.

##### Field1 = Field2 (join predicate)
Push down only if both fields are in the same schema; otherwise keep at join.

Example: R(a) ⋈ S(b) with R.a = S.b → keep for join.

##### Mixed forms:
A ∧ (B ∨ C): You can push A; keep (B ∨ C) above.

A ∨ (B ∧ C): Don’t push anything unless the entire OR applies; consider, but usually avoid, distributive rewrite: (A ∨ B) ∧ (A ∨ C).

Empty-at-this-level operands: Treat as “not applicable here,” not false.

AND: drop inapplicable parts and push the rest.

OR: if any part is inapplicable, don’t push the OR.

## Notes From Papers

### Notes From Volcano Paper (1994 Paper Dealing With Execution)
Broadly, two forms of data flow defined in Volcano:

- Demand driven data flow (lazy, iterator style with lazy evaluation) [also called pull based flow]
- Data driven data flow (eager with immediate materialization) [also called push based flow]

[This](https://justinjaffray.com/query-engines-push-vs.-pull/) is a good resource to read more about these two styles.

1. Volcano is an extensible query execution engine. 
      What makes it extensible:
      
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

This image is a great overview of how Volcano works as an execution engine and fits into the pipeline with an optimizer, buffer pool etc.

![](/assets/img/query_engines.png)

A one page infographic generated by ChatGPT

![](/assets/img/volcano_infographic.png)

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

To put it more concisely, Volcano separates optimization into two phases - logical & physical. Cascades doesn't make this distinction.

#### Rules
Rules are an abstraction for the kinds of transformations that can be applied to expressions and rewrite them. They can do logical -> logical or logical -> physical mappings
Rules consist of a pattern (antecedent) that they match, and a substitute expression they produce when applied.. Custom rules can be registered.

#### Tasks
Tasks are an abstraction for how transformations actually happen to the query plan. Cascade maintains a heap allocated priority queue and dumps tasks into them. The value of the promise associated with a rule is what dictates where the task lands in the priority queue [cost = estimated cost so far + cost for rule].

This heap allocated task queue is what allows for multi-threaded query plan exploration as long as task queue and memoization hash map are properly synchronized.

### Notes From An Overview Of Query Optimization In Relational Systems

- My impressions before reading this paper were, there are 3 clear classes of optimizers - Rule Based Optimizers (RBO's), Heuristic Optimizers and Cost Based Optimizers (CBO's). But, this mental model is incorrect.

  Rather, Heuristic and CBO's make use of rules to apply transformations, (logical -> logical or logical -> physical). Heuristic Optimizer's might make use of no cost (pure rules) or fixed cost estimates when deciding which plan to go with. CBO's on the other hand make use of dynamic cost computation at runtime to decide which plan is better.

- There was a gradual evolution of query engines (both execution and optimization) following System R -> Starburst -> Volcano -> Cascades and each successor refined the ideas of the previous engine.

- The big change with the evolution has been the extensability of these engines. With each generation it became easier to add new rules, new operators etc.

- System R started with bottom-up memoization for plan enumeration with only linear join orderering consideration (no bushy ordering). This contrasts with Cascade's top-down memoization approach.

- The key cost factors used in estimation are:
      * I/O
      * CPU
      * memory (buffer pool estimation)
      * network (especially important for distributed databases)

- Scheduling, in the context of distributed and parallel databases, is a major point that can be optimized.

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