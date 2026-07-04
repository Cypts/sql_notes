# SQL Subqueries — Complete Deep-Dive Notes

Covers the basics everyone teaches, plus the internals, edge cases, and traps that interviewers love but courses skip.

---

## 1. What is a Subquery?

A **subquery** (or inner query/nested query) is a query embedded inside another SQL statement. The outer query is called the **main query** or **outer query**.

```sql
SELECT * FROM Employee
WHERE salary > (SELECT AVG(salary) FROM Employee);
```

- The subquery `(SELECT AVG(salary) FROM Employee)` executes and produces a value.
- The outer query uses that value to filter its own rows.

Subqueries can appear almost anywhere a value or table is expected: in `SELECT`, `FROM`, `WHERE`, `HAVING`, and even inside `INSERT`/`UPDATE`/`DELETE`.

---

## 2. Classification by Return Type

This is the classification interviewers actually probe on — most tutorials just say "subquery" without distinguishing these.

| Type | Returns | Example Use |
|------|---------|--------------|
| **Scalar subquery** | Single value (1 row, 1 column) | `WHERE salary > (SELECT AVG(salary) FROM Employee)` |
| **Row subquery** | Single row, multiple columns | `WHERE (dept_id, salary) = (SELECT dept_id, MAX(salary) FROM Employee)` |
| **Table/multi-row subquery** | Multiple rows, one or more columns | `WHERE dept_id IN (SELECT dept_id FROM Department WHERE location='Delhi')` |
| **Correlated subquery** | Depends on outer query's current row; re-evaluated per row | `WHERE salary > (SELECT AVG(salary) FROM Employee e2 WHERE e2.dept_id = e1.dept_id)` |

> **Trap:** If a **scalar subquery unexpectedly returns more than one row**, the engine throws a runtime error:
> ```
> ERROR: more than one row returned by a subquery used as an expression
> ```
> This happens often when people forget a `WHERE` condition inside the subquery and it silently returns multiple rows only for certain inputs — meaning the query can pass testing and fail in production on different data.

---

## 3. Subqueries by Location (Where They Can Appear)

### a) In `WHERE`
Most common usage — filtering rows based on a computed value or set.
```sql
SELECT * FROM Employee
WHERE dept_id IN (SELECT dept_id FROM Department WHERE budget > 1000000);
```

### b) In `SELECT` (as a Scalar Column)
Runs once **per row** of the outer query — can be a hidden performance killer.
```sql
SELECT name,
       (SELECT COUNT(*) FROM Projects p WHERE p.emp_id = e.id) AS project_count
FROM Employee e;
```
> **Rarely taught fact:** This is logically a **correlated subquery**, and most engines execute it **once per outer row** unless the optimizer rewrites it into a join (some do, some don't, depending on engine/version). On a table with 100,000 rows, this can mean 100,000 subquery executions — a huge, often invisible performance cost. Always check the execution plan (`EXPLAIN`) before assuming a subquery-in-SELECT is "just as fast" as a join.

### c) In `FROM` (Derived Table / Inline View)
The subquery acts as a temporary, unnamed table for the outer query to select from. **Must be given an alias.**
```sql
SELECT dept_id, avg_salary
FROM (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM Employee
    GROUP BY dept_id
) dept_avg
WHERE avg_salary > 50000;
```
> **Why this matters:** This is how you filter on an aggregate **without using HAVING** — by pre-aggregating in a derived table, then applying a normal `WHERE` on the outer query. This sidesteps the exact `HAVING salary > AVG(salary)` problem from earlier.

### d) In `HAVING`
```sql
SELECT dept_id, AVG(salary) AS avg_sal
FROM Employee
GROUP BY dept_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM Employee);
```

### e) In `INSERT`
```sql
INSERT INTO HighEarners (emp_id, salary)
SELECT id, salary FROM Employee WHERE salary > 100000;
```

### f) In `UPDATE`
```sql
UPDATE Employee
SET salary = salary * 1.10
WHERE dept_id IN (SELECT id FROM Department WHERE performance_rating = 'A');
```

### g) In `DELETE`
```sql
DELETE FROM Employee
WHERE dept_id IN (SELECT id FROM Department WHERE is_active = 0);
```

---

## 4. Correlated vs Non-Correlated Subqueries (The Part Most Courses Gloss Over)

### Non-Correlated Subquery
- Can run **independently** of the outer query.
- Executed **once**, and its result is reused for every row of the outer query.
```sql
SELECT * FROM Employee
WHERE salary > (SELECT AVG(salary) FROM Employee);
```
Here, `AVG(salary)` doesn't depend on anything from the outer `Employee` reference — it's computed once.

### Correlated Subquery
- References a column from the **outer query** — cannot run independently.
- Conceptually (and often literally), the engine re-evaluates it **once for every row** of the outer query.
```sql
SELECT e1.name, e1.salary
FROM Employee e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM Employee e2
    WHERE e2.dept_id = e1.dept_id   -- references outer table e1
);
```

### How Correlated Subqueries Actually Execute (Internals — Rarely Explained)
Conceptually, the engine behaves like a **nested loop**:
```
FOR each row in outer query (e1):
    Execute inner subquery using e1's current value (e.g., e1.dept_id)
    Compare/use the result
```
This is why correlated subqueries have a reputation for being **slow on large tables** — naively, it's O(N × M) where N = outer rows, M = cost of inner subquery per row.

> **What no one tells you:** Modern optimizers (PostgreSQL, SQL Server, Oracle, MySQL 8+) often **don't actually run it this naively**. They perform **subquery unnesting/decorrelation** — rewriting the correlated subquery into an equivalent `JOIN` or semi-join internally, which can be executed far more efficiently (e.g., via a single hash join instead of N nested lookups). This is why `EXPLAIN` output is essential — you often can't predict actual performance just by looking at the SQL text; you need to see the transformed physical plan.

---

## 5. EXISTS vs IN vs =ANY/=ALL — The Real Differences

### `IN`
Checks if a value matches **any value** in a list/subquery result.
```sql
SELECT * FROM Employee
WHERE dept_id IN (SELECT id FROM Department WHERE location = 'Mumbai');
```

### `EXISTS`
Checks whether the subquery returns **at least one row** — doesn't care about actual values, just row existence. Almost always used with a **correlated** subquery.
```sql
SELECT * FROM Employee e
WHERE EXISTS (
    SELECT 1 FROM Department d
    WHERE d.id = e.dept_id AND d.location = 'Mumbai'
);
```

### Key Behavioral Difference — Performance
- `EXISTS` typically **short-circuits**: as soon as one matching row is found, it stops scanning and returns `TRUE` — it doesn't need to fetch or compare all values.
- `IN` traditionally required materializing the entire result set of the subquery first, especially on older optimizers — though modern engines often optimize `IN` into a semi-join anyway, largely closing this performance gap.

> **Rule of thumb still taught in interviews:** `EXISTS` is generally preferred over `IN` for **correlated** existence checks, especially on large subquery result sets, because it avoids the need to build/compare against a large intermediate list. But always verify using `EXPLAIN` on your specific engine — the "IN is always slower" claim is outdated for many modern optimizers.

---

## 6. THE NULL TRAP — `NOT IN` vs `NOT EXISTS` (The #1 Thing No One Teaches)

This is one of the most dangerous, silent bugs in SQL, and it's rarely covered in basic courses.

### The Problem
```sql
SELECT * FROM Employee
WHERE dept_id NOT IN (SELECT dept_id FROM Department);
```

If **even one row** in the subquery's result (`Department.dept_id`) is `NULL`, this query returns **ZERO rows** — even if there are obviously valid non-matching values! This is almost never what the developer intended, and it fails silently (no error, just wrong/empty results).

### Why This Happens
SQL's `NOT IN` is internally evaluated as a series of `<> ` (not-equal) comparisons combined with `AND`:
```sql
dept_id <> val1 AND dept_id <> val2 AND dept_id <> NULL ...
```
In three-valued logic (`TRUE`/`FALSE`/`UNKNOWN`), any comparison against `NULL` returns `UNKNOWN` — not `TRUE` or `FALSE`. Since the whole condition is an `AND` chain, if even **one** comparison evaluates to `UNKNOWN`, the **entire expression can never evaluate to TRUE** (because `AND` with `UNKNOWN` can't be certainly `TRUE`). So no rows are returned, silently.

### The Fix
**Option 1 — Filter out NULLs explicitly in the subquery:**
```sql
SELECT * FROM Employee
WHERE dept_id NOT IN (
    SELECT dept_id FROM Department WHERE dept_id IS NOT NULL
);
```

**Option 2 — Use `NOT EXISTS` instead (safer by default):**
```sql
SELECT * FROM Employee e
WHERE NOT EXISTS (
    SELECT 1 FROM Department d WHERE d.dept_id = e.dept_id
);
```
`NOT EXISTS` doesn't suffer from this problem because it checks for row existence via equality matching per row — it isn't building a giant `<>` chain across the whole set, so stray NULLs in the subquery don't poison the entire condition.

> **Interview gold:** *"Why is `NOT EXISTS` generally recommended over `NOT IN`?"* — The NULL trap above is the exact, precise answer. Most interviewers are specifically testing whether you know this, because it's a genuine production bug pattern, not just a style preference.

---

## 7. Subquery vs JOIN — When to Use Which

| Aspect | Subquery | JOIN |
|--------|----------|------|
| Purpose | Filtering / computing a value based on another table | Combining columns from multiple tables into one result set |
| Returns columns from both tables? | No (subquery result isn't directly selectable in outer SELECT unless using a derived table) | Yes |
| Readability for existence checks | Often clearer (`EXISTS`) | Can require `DISTINCT` to avoid duplicate rows |
| Performance | Can be optimized away into a join internally by the optimizer | Generally well-optimized, especially with proper indexes |
| Duplicate row risk | Lower (especially with `EXISTS`/`IN`) | Higher — a one-to-many join can duplicate outer rows unless handled carefully |

> **Rarely taught nuance:** A `JOIN` used purely for filtering (i.e., you join to another table just to check existence, but never select its columns) can accidentally **duplicate rows** if there are multiple matches in the joined table — a bug `EXISTS`/`IN` naturally avoids, since they only check for existence, not multiply rows.

```sql
-- Buggy: duplicates Employee rows if an employee has multiple matching projects
SELECT e.name FROM Employee e
JOIN Projects p ON p.emp_id = e.id
WHERE p.status = 'active';

-- Correct: no duplication, just an existence check
SELECT e.name FROM Employee e
WHERE EXISTS (
    SELECT 1 FROM Projects p WHERE p.emp_id = e.id AND p.status = 'active'
);
```

---

## 8. Subquery vs CTE (Common Table Expression) — What No One Compares Properly

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM Employee
    GROUP BY dept_id
)
SELECT * FROM dept_avg WHERE avg_salary > 50000;
```

- A CTE (`WITH` clause) is essentially syntactic sugar for a subquery in `FROM`, offering:
  - **Readability** — especially for multi-step logic, since you name intermediate results.
  - **Reusability** — a CTE can be referenced **multiple times** in the same query without re-writing the subquery.
  - **Recursive queries** — CTEs support `WITH RECURSIVE`, which plain subqueries cannot do (used for hierarchical data like org charts, category trees).

> **Performance myth to know about:** In PostgreSQL (pre-v12), a CTE used to act as an **optimization fence** — meaning the planner would materialize it fully before the outer query could apply filters, sometimes hurting performance compared to an equivalent subquery. From PostgreSQL 12 onward, CTEs are **inlined by default** (unless marked `MATERIALIZED` explicitly), closing this old gap. This kind of engine-specific historical detail is exactly the sort of thing interviewers use to test real depth versus memorized syntax.

---

## 9. LATERAL Subqueries (Advanced — Genuinely Rarely Taught)

A `LATERAL` subquery (PostgreSQL, and similar via `CROSS APPLY`/`OUTER APPLY` in SQL Server) is a subquery in the `FROM` clause that is allowed to **reference columns from preceding tables in the same FROM clause** — something a normal derived table cannot do.

```sql
SELECT e.name, top_project.title
FROM Employee e,
LATERAL (
    SELECT title FROM Projects p
    WHERE p.emp_id = e.id
    ORDER BY p.budget DESC
    LIMIT 1
) top_project;
```
This finds, **per employee**, their single highest-budget project — something that's awkward to express with a plain `JOIN` (which would require a separate aggregation + join-back step) but natural with `LATERAL`.

> Equivalent in SQL Server: `CROSS APPLY` / `OUTER APPLY`.

---

## 10. Nested Subqueries (Multiple Levels)

Subqueries can be nested arbitrarily deep, though beyond 2-3 levels readability collapses fast:
```sql
SELECT * FROM Employee
WHERE dept_id IN (
    SELECT id FROM Department
    WHERE region_id IN (
        SELECT id FROM Region WHERE country = 'India'
    )
);
```
> **Practical advice for interviews and real code:** Deeply nested subqueries are almost always better rewritten as **CTEs** (`WITH` clauses) for readability, or flattened into joins where possible — but you should know how to both read and write nested versions, since legacy codebases are full of them.

---

## 11. Subquery Optimization — What the Engine Actually Does (The Truly Under-Taught Part)

Most courses treat subqueries as if they execute exactly as written, top to bottom. In reality:

1. **Subquery Unnesting/Flattening**: Many correlated subqueries (especially those using `IN`, `EXISTS`, or scalar subqueries in `WHERE`) are automatically rewritten by the optimizer into equivalent **joins** or **semi-joins**, because joins are generally cheaper to execute (especially with hash joins) than a naive per-row nested loop.
2. **Materialization vs Pipelining**: A **non-correlated subquery** in `FROM` may be **materialized** (fully computed and stored in a temp structure) once, then reused — or it may be **pipelined** (rows streamed through without full materialization), depending on the optimizer's cost estimate.
3. **Semi-Joins and Anti-Joins**: `EXISTS`/`IN` conditions are typically compiled into **semi-join** operators (return outer rows that have *at least one* match, without duplicating), while `NOT EXISTS`/`NOT IN` (properly written) become **anti-join** operators (return outer rows that have *no* match). These are distinct, optimized physical join strategies — not literally "loop through and check."

> **Why this matters for interviews:** If asked "is a subquery always slower than a join?" — the correct, senior-level answer is: **"Not necessarily — a good optimizer will often transform a correlated subquery into a join-based plan internally, so the written form doesn't dictate the actual execution strategy. Always verify with EXPLAIN."**

---

## 12. Scalar Subquery Caching / Re-Evaluation Nuance

For a **non-correlated scalar subquery** used multiple times in one query, some engines evaluate it **once** and cache the result; others may re-evaluate it each time it's referenced, depending on where it appears. This is engine-specific behavior and not standardized — another reason to check `EXPLAIN` output rather than assume.

---

## 13. Common Errors & Their Root Causes

| Error / Symptom | Root Cause |
|------------------|-------------|
| `subquery returns more than one row` | Used a scalar context (`=`, `>`, etc.) but subquery returns multiple rows — use `IN`/`ANY`/`ALL` or add more filtering |
| Query silently returns 0 rows unexpectedly | The classic `NOT IN` + NULL trap (see Section 6) |
| Derived table error: "every derived table must have its own alias" | Forgot to alias a subquery used in `FROM` |
| Extremely slow query despite small tables | Correlated subquery in `SELECT` clause executing once per row without optimizer rewrite — check `EXPLAIN` |
| Duplicate rows in results | Used a `JOIN` for an existence check instead of `EXISTS`/`IN`, and the joined table had multiple matches per outer row |

---

## 14. Quick Reference Cheat Sheet

| Goal | Best Tool |
|------|-----------|
| Compare a row's value to an aggregate over the whole table | Subquery in `WHERE` (or window function `OVER()`) |
| Filter groups after aggregation | `HAVING` (not subquery needed) |
| Check for existence of related rows without needing their data | `EXISTS` (correlated subquery) |
| Check for absence of related rows (safely) | `NOT EXISTS` — never `NOT IN` unless NULLs are excluded |
| Get one row's related aggregate columns alongside detail rows | Window function or `LATERAL` |
| Reuse the same computed subquery result multiple times, readably | CTE (`WITH`) |
| Hierarchical/recursive data (org charts, categories) | `WITH RECURSIVE` CTE |
| Per-row "top N related rows" (e.g., top project per employee) | `LATERAL` / `CROSS APPLY` |

---

## 15. Interview Questions Specifically on Subqueries

1. **Why does `NOT IN` sometimes return zero rows unexpectedly, and how do you fix it?**
   Because of NULL values in the subquery's result causing three-valued logic to evaluate the whole `AND` chain to `UNKNOWN`. Fix: filter NULLs explicitly, or use `NOT EXISTS`.

2. **Are subqueries always slower than joins?**
   No — cost-based optimizers frequently rewrite correlated subqueries (especially `EXISTS`/`IN`) into semi-joins or joins internally. Written form ≠ execution form; verify with `EXPLAIN`.

3. **What's the difference between a correlated and non-correlated subquery?**
   A correlated subquery references a column from the outer query and conceptually re-evaluates per outer row; a non-correlated subquery is independent and evaluated once.

4. **When would you use `EXISTS` over `IN`?**
   When you only need to check for the presence of at least one matching row (not the actual matched value), especially in correlated scenarios — `EXISTS` short-circuits on the first match.

5. **What is a LATERAL join and when would you need it?**
   A subquery in `FROM` allowed to reference columns from earlier tables in the same `FROM` clause — used for "top-N per group" style queries without a separate join-back step.

6. **Why might a CTE perform differently from an equivalent subquery?**
   In some engines/versions (e.g., PostgreSQL pre-12), CTEs were optimization fences and materialized fully before filters could be pushed in; modern versions inline CTEs by default unless explicitly marked `MATERIALIZED`.

7. **What happens if a scalar subquery unexpectedly returns multiple rows?**
   The query throws a runtime error ("more than one row returned by a subquery used as an expression") — a common production bug when the subquery's filtering conditions aren't as tight as assumed.