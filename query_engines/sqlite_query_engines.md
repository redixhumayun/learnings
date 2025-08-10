# SQLite Query Engines

The query engine is interesting because it doesn't follow the volcano model which constructs a query tree of iterators. Instead, it uses the VDBE to build the query program and emit it. I still don't know what limitations or opportunities this creates.

When a query has a AND term in the WHERE clause like `SELECT price FROM FruitsForSale WHERE fruit='Orange' AND state='CA';`, it can use a single multi-column (maybe covering) index to answer the query.

If, on the other hand, a query has OR terms in the WHERE claude like `SELECT price FROM FruitsForSale WHERE fruit='Orange' OR state='CA';`, SQLite conceptually treats each OR term independently and applies the index to each term

```sql
SELECT price FROM fruitsforSale WHERE fruit='Orange' OR state='CA';
-- SQLite will translate this to:
SELECT price FROM fruitsforSale WHERE fruit='Orange'
UNION ALL
SELECT price FROM fruitsforSale WHERE state='CA';
```

This technique is called the OR-by-UNION technique. An alternative to the AND technique above is to do the AND-by-INTERSECT technique.

```sql
SELECT price FROM fruitsforSale WHERE fruit='Orange' AND state='CA';
-- SQLite will translate this to:
SELECT price FROM fruitsforSale WHERE fruit='Orange'
INTERSECT
SELECT price FROM fruitsforSale WHERE state='CA';
```

## Links
1. [SQLite Query Planner](https://www.sqlite.org/queryplanner.html)
1. [SQLite Query Optimizer](https://www.sqlite.org/optoverview.html)