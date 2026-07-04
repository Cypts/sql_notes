# SQL Query Execution — Detailed Internal Flow

A step-by-step breakdown of what happens inside a database engine between the moment a query is submitted and the moment results are returned. Useful for interview prep (TCS Prime, backend/DB internals rounds).

---

## Overview Table

| Stage | What Happens |
|-------|--------------|
| 1 | User submits SQL query |
| 2 | Parser validates syntax and object references |
| 3 | Parse tree is created |
| 4 | Query may be rewritten/simplified |
| 5 | Optimizer evaluates possible execution strategies |
| 6 | Best execution plan is selected |
| 7 | Executor runs the plan |
| 8 | Storage engine retrieves data from memory or disk |
| 9 | Filters (`WHERE`) are applied |
| 10 | Joins, grouping, and sorting are performed if required |
| 11 | Requested columns are projected |
| 12 | Final result set is returned to the client |

---

## Stage 1: User Submits SQL Query

The client — an application, ORM, CLI tool, or driver — sends a SQL statement (`SELECT`, `INSERT`, `UPDATE`, etc.) to the database server over an established connection.

- At this point the query is just **plain text**.
- The engine has no understanding of its meaning yet — it hasn't been checked, structured, or planned.
- The connection/session layer receives it and hands it off to the parser.

---

## Stage 2: Parser Validates Syntax and Object References

The **parser** performs two distinct types of validation:

### a) Syntactic Validation
Checks whether the query obeys SQL grammar rules:
- Are keywords in the correct order (`SELECT` before `FROM` before `WHERE`)?
- Are parentheses balanced?
- Are commas, quotes, and clause structures correctly formed?

If this fails → **syntax error**, and the query is rejected immediately without touching any table.

### b) Semantic Validation
Checks whether the query makes sense against the actual database:
- Do the referenced tables and columns exist?
- Are data types compatible (e.g., comparing a string to an integer column)?
- Does the current user have the required privileges (SELECT/INSERT/UPDATE rights) on those objects?

This requires a lookup into the **system catalog / data dictionary** (metadata tables that describe schema, types, constraints, and permissions).

> **Interview angle:** Syntax errors are caught purely by grammar rules; semantic errors require metadata lookups. A query can be syntactically perfect but semantically invalid (e.g., referencing a non-existent column).

---

## Stage 3: Parse Tree is Created

Once validated, the parser converts the SQL text into a **parse tree** (also called a syntax tree or AST — Abstract Syntax Tree).

- This is a **hierarchical structure** representing the query's components: SELECT list, FROM clause, WHERE predicate, JOIN conditions, GROUP BY, ORDER BY, etc.
- Every subsequent stage operates on this tree — not on the raw SQL text.
- Example: `SELECT name FROM employees WHERE dept = 'IT'` becomes a tree with a projection node (name), a scan node (employees), and a filter node (dept = 'IT').

> **Interview angle — Parse tree vs. Execution plan:**
> - Parse tree = structural representation of *what the query means*.
> - Execution plan = *how* the engine will physically carry it out (comes later, in Stage 6).

---

## Stage 4: Query May Be Rewritten/Simplified

The **query rewriter**, sometimes called the **logical optimizer**, transforms the parse tree into a logically equivalent but cleaner/more efficient form. This doesn't decide *how* to execute the query — only produces a better logical version of it.

Common rewrite techniques:
- **View expansion** — replacing a view reference with its underlying query definition.
- **Subquery flattening** — converting nested subqueries into equivalent joins where possible (joins are often cheaper to optimize than correlated subqueries).
- **Constant folding** — simplifying expressions like `WHERE 1=1 AND salary > 5000` to `WHERE salary > 5000`.
- **Predicate simplification** — removing redundant or always-true/false conditions.
- **IN-to-semi-join conversion** — rewriting `WHERE id IN (SELECT ...)` as a semi-join for efficiency.

---

## Stage 5: Optimizer Evaluates Possible Execution Strategies

This is the most computationally complex and interview-relevant stage. For any non-trivial query, there are **many different physical ways** to execute the same logical query, and the **query optimizer** must evaluate them.

Key considerations the optimizer weighs:

| Factor | Details |
|--------|---------|
| **Access paths** | Full table scan vs. index scan vs. index seek |
| **Join order** | For multi-table queries, the order in which tables are joined drastically changes cost |
| **Join algorithm** | Nested loop join, hash join, or merge join |
| **Available indexes** | Which indexes exist and whether they're usable for the given predicates |
| **Statistics** | Row counts, data distribution (histograms), NULL ratios, cardinality estimates |

Most modern relational databases (MySQL/InnoDB, PostgreSQL, Oracle, SQL Server) use a **Cost-Based Optimizer (CBO)**:
- It estimates the cost (I/O operations + CPU cycles) of each candidate plan.
- It picks the plan with the **lowest estimated cost** — not necessarily the objectively fastest one, since it relies on estimates.

> **Interview angle:** *"Why does adding an index sometimes not improve performance?"*
> Because the optimizer's cost estimate — based on stored statistics — may still favor a full table scan, especially for small tables or low-selectivity columns (e.g., a boolean column where an index scan isn't cheaper than reading the whole table).

> **Interview angle:** *Outdated statistics* are a classic cause of poor query plans even when the query itself is well-written and properly indexed.

---

## Stage 6: Best Execution Plan is Selected

Among all the candidate plans generated in Stage 5, the optimizer commits to the one with the lowest estimated cost. This becomes the **execution plan** (or query plan) — a tree of **physical operators**: scan, filter, join, sort, aggregate — each with a defined order of execution.

- This is exactly what you see when running `EXPLAIN` on a query.
- The plan specifies not just *what* operations happen, but the *exact algorithm and access method* for each.

> **Interview angle — `EXPLAIN` vs `EXPLAIN ANALYZE`:**
> - `EXPLAIN` shows the **estimated** plan (no query execution).
> - `EXPLAIN ANALYZE` actually **executes** the query and reports real, measured statistics (actual rows scanned, actual time taken) alongside the estimates — useful for spotting where estimates were wrong.

---

## Stage 7: Executor Runs the Plan

The **execution engine** walks through the chosen physical plan and invokes each operator in sequence.

Most engines use an **iterator model** (also called the **Volcano/Pull model**):
- Each operator implements `open()`, `next()`, `close()`.
- Operators are composed as a tree: a join operator calls `next()` on its child scan operators to pull rows, applies join logic, and passes the combined row upward.
- This allows **pipelining** — rows can flow through multiple operators without waiting for an entire intermediate result set to materialize (in many cases).

> **Interview angle:** This is why some operations (like a hash join's build phase, or a full sort) *must* materialize a complete intermediate result, while others (like nested loop joins) can stream row-by-row.

---

## Stage 8: Storage Engine Retrieves Data From Memory or Disk

The **storage engine** (e.g., InnoDB for MySQL, or the storage layer in PostgreSQL) is responsible for physically fetching the requested data pages.

The retrieval sequence:
1. Check the **buffer pool / cache** in memory first.
   - **Cache hit** → data is already in RAM, returned quickly.
   - **Cache miss** → the required page must be read from disk into the buffer pool, then returned.
2. Access method follows what the plan specified in Stage 6:
   - **Sequential/full scan** — reads pages in order, touching every row.
   - **Index scan/seek** — uses a B-tree (or similar) index to jump directly to relevant pages, touching far fewer pages than a full scan.

> **Interview angle:** This stage is *why* indexing matters so much — a good index reduces the number of physical page reads (I/O), which is usually the most expensive part of query execution.

---

## Stage 9: Filters (WHERE) are Applied

As rows are retrieved, the **filter operator** evaluates the `WHERE` clause condition against each row and discards non-matching rows.

Depending on the chosen plan, filtering can happen at different points:
- **Predicate pushdown**: filtering is pushed down as close to the storage layer as possible (e.g., applied during the index scan itself), so unnecessary rows/columns are never even fully materialized. This is more efficient.
- **Post-retrieval filtering**: rows are fetched first, then filtered in a separate step by the execution layer — less efficient, more I/O and CPU wasted.

> **Interview angle:** Predicate pushdown is a major optimization technique, especially relevant in distributed/big-data systems (e.g., Spark, Hive) where it reduces network shuffling too.

---

## Stage 10: Joins, Grouping, and Sorting are Performed if Required

This stage combines and organizes intermediate results depending on the query's clauses.

### Joins
Combine rows from multiple tables per the join condition, using the algorithm chosen earlier:
- **Nested Loop Join** — for each row in the outer table, scan the inner table; efficient for small datasets or when a good index exists on the join key.
- **Hash Join** — builds a hash table on the smaller relation's join key, then probes it with the other relation; efficient for large, unsorted datasets.
- **Merge Join** — requires both inputs sorted on the join key, then merges them in a single pass; efficient when data is already sorted (e.g., via an index).

### Grouping (`GROUP BY`)
Rows are bucketed by grouping key(s), typically via:
- **Hash-based grouping** — builds a hash table keyed by group values.
- **Sort-based grouping** — sorts rows by group key first, then aggregates consecutive matching rows.

Aggregate functions (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) are computed per group here.

### Sorting (`ORDER BY`)
Rows are ordered by the specified column(s):
- **In-memory sort** — if the data fits within available memory.
- **External merge sort** — if data exceeds memory, intermediate sorted runs are spilled to disk and merged.
- **"Free" sort** — if an index already provides the required order, no separate sort step is needed.

> **Interview angle:** Joins, grouping, and sorting are typically the **most resource-intensive stages** of query execution — they can require significant working memory (`sort_buffer`/`work_mem` in various DBs) or temporary disk space (`tempdb`, temp tablespaces) for large datasets.

---

## Stage 11: Requested Columns are Projected

The **projection operator** trims each intermediate row down to only the columns actually listed in the `SELECT` clause.

- Columns used only for filtering or joining (but not requested in the output) are dropped here.
- Computed expressions and aliases are evaluated at this point — e.g., `SELECT price * qty AS total_amount`.

> **Interview angle:** Projection typically happens *late* in the pipeline (after filtering/joining/grouping) so that intermediate operators still have access to all columns they need, even if the final output doesn't include them.

---

## Stage 12: Final Result Set is Returned to the Client

The resulting rows are packaged into the wire protocol format and sent back to the client over the connection.

- For large result sets, rows are often **streamed in batches** (cursors/fetch-size) rather than materializing and sending everything at once — this reduces memory pressure on both the server and the client.
- The client application/driver then processes or displays the returned rows.

---

## Quick Recap — The Full Pipeline

```
Submit Query
   ↓
Parse (Syntax + Semantic Validation)
   ↓
Build Parse Tree
   ↓
Rewrite / Simplify (Logical Optimization)
   ↓
Optimizer Evaluates Strategies (Cost-Based)
   ↓
Select Best Execution Plan
   ↓
Executor Runs Plan (Iterator Model)
   ↓
Storage Engine Fetches Data (Cache → Disk)
   ↓
Apply WHERE Filters
   ↓
Perform Joins / Grouping / Sorting
   ↓
Project Requested Columns
   ↓
Return Result Set to Client
```

---

## Common Interviewer Follow-Ups on This Topic

1. **Why does adding an index sometimes not help performance?**
   The cost-based optimizer relies on table statistics; for small tables or low-selectivity columns, a full scan may still be estimated as cheaper than an index scan.

2. **Difference between logical and physical query plans?**
   - *Logical plan* — describes **what** operations are needed (the rewritten/simplified query tree from Stage 4).
   - *Physical plan* — describes **how** those operations are actually carried out (specific algorithms and access paths, decided in Stage 6).

3. **Difference between `EXPLAIN` and `EXPLAIN ANALYZE`?**
   `EXPLAIN` shows the estimated plan without running the query; `EXPLAIN ANALYZE` executes it and reports actual runtime statistics.

4. **Why are joins/sorting/grouping expensive?**
   They often require full materialization of intermediate results and significant memory or temp disk space, unlike filtering, which can be pushed down and applied row-by-row.

5. **What is predicate pushdown, and why does it matter?**
   It's the technique of applying filters as early and as close to the data source as possible, minimizing the volume of data that flows through later stages — critical for performance in both single-node and distributed databases.

6. **What is the iterator/Volcano model?**
   A standard execution model where each operator exposes `open()/next()/close()` methods, allowing operators to be composed into a pipeline that pulls rows on demand rather than materializing entire result sets at every step.