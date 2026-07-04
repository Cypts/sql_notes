# SQL Syntax, Grammar & Order of Execution

A companion to the "SQL Query Execution Stages" notes — this covers the **written grammar** of a SQL query (the order you *type* clauses in) versus the **logical execution order** (the order the engine actually *processes* them in). This mismatch is one of the most commonly asked SQL fundamentals in interviews.

---

## 1. Full SQL Query Grammar (Written/Syntax Order)

This is the order clauses must appear in when you **write** a SQL query. Violating this order causes a syntax error.

```sql
SELECT [DISTINCT] column1, column2, ... | *
FROM table1
[JOIN table2 ON join_condition]
[WHERE condition]
[GROUP BY column1, column2, ...]
[HAVING group_condition]
[ORDER BY column1 [ASC|DESC], ...]
[LIMIT number [OFFSET number]];
```

### Clause-by-Clause Grammar Rules

| Clause | Purpose | Mandatory? |
|--------|---------|------------|
| `SELECT` | Specifies which columns/expressions to return | Yes |
| `DISTINCT` | Removes duplicate rows from the result | No |
| `FROM` | Specifies the source table(s) | Yes (except for expression-only queries) |
| `JOIN ... ON` | Combines rows from multiple tables based on a condition | No |
| `WHERE` | Filters individual rows **before** grouping | No |
| `GROUP BY` | Groups rows sharing the same value(s) into summary rows | No |
| `HAVING` | Filters **groups** (used after aggregation) | No |
| `ORDER BY` | Sorts the final result set | No |
| `LIMIT` / `OFFSET` | Restricts number of rows returned / skips rows | No |

### Syntax Rules to Remember
- Keywords are case-insensitive (`SELECT` = `select`), but conventionally written in uppercase for readability.
- Clauses **must appear in the fixed order above** — you cannot write `WHERE` before `FROM`, or `GROUP BY` before `WHERE`.
- String literals use single quotes: `'IT'`, not double quotes (double quotes are for identifiers in standard SQL/PostgreSQL).
- Statements typically end with a semicolon `;`.
- Multiple conditions in `WHERE`/`HAVING` are combined using `AND`, `OR`, `NOT`.
- `HAVING` is used **only** with aggregate functions/grouped data — `WHERE` cannot filter on aggregate results like `COUNT(*)` or `SUM(salary)`.

### Example Query
```sql
SELECT department, COUNT(*) AS emp_count
FROM employees
WHERE status = 'active'
GROUP BY department
HAVING COUNT(*) > 5
ORDER BY emp_count DESC
LIMIT 10;
```

---

## 2. Logical Order of Execution

This is the order in which the **database engine actually evaluates** the clauses — completely different from the order you type them in.

```
1. FROM        → identify source table(s)
2. JOIN        → combine rows from multiple tables
3. WHERE       → filter individual rows
4. GROUP BY    → group remaining rows
5. HAVING      → filter groups
6. SELECT      → compute/select final columns and expressions
7. DISTINCT    → remove duplicate rows
8. ORDER BY    → sort the result set
9. LIMIT/OFFSET→ restrict number of returned rows
```

### Side-by-Side Comparison

| Step | Written Order | Logical Execution Order |
|------|---------------|--------------------------|
| 1 | `SELECT` | `FROM` |
| 2 | `FROM` | `JOIN` |
| 3 | `JOIN` | `WHERE` |
| 4 | `WHERE` | `GROUP BY` |
| 5 | `GROUP BY` | `HAVING` |
| 6 | `HAVING` | `SELECT` |
| 7 | `ORDER BY` | `DISTINCT` |
| 8 | `LIMIT` | `ORDER BY` |
| 9 | — | `LIMIT` |

> **Interview angle:** *"Why can't you use a column alias defined in SELECT inside a WHERE clause?"*
> Because `WHERE` is logically executed **before** `SELECT`. At the point `WHERE` runs, the alias doesn't exist yet. This is why:
> ```sql
> SELECT salary * 12 AS annual_salary
> FROM employees
> WHERE annual_salary > 500000;  -- ❌ ERROR: alias not recognized yet
> ```
> But it **does** work in `ORDER BY`, because `ORDER BY` executes **after** `SELECT`:
> ```sql
> SELECT salary * 12 AS annual_salary
> FROM employees
> ORDER BY annual_salary DESC;  -- ✅ Works fine
> ```

> **Interview angle:** *"Why can `HAVING` use aggregate functions but `WHERE` cannot?"*
> Because `WHERE` operates on individual rows **before** grouping/aggregation happens, so aggregate values like `SUM()` or `COUNT()` don't exist yet at that stage. `HAVING` runs **after** `GROUP BY`, once aggregates have been computed.

---

## 3. Connecting Grammar to Execution — Full Flow

Here's how the two orders map onto the actual execution stages discussed earlier:

```
WRITTEN QUERY:
  SELECT dept, COUNT(*) AS cnt
  FROM employees
  WHERE status = 'active'
  GROUP BY dept
  HAVING COUNT(*) > 5
  ORDER BY cnt DESC
  LIMIT 10;

        ↓  (Parser rearranges into logical plan)

LOGICAL EXECUTION FLOW:
  1. FROM employees          → identify source table
  2. WHERE status='active'   → filter rows (row-level)
  3. GROUP BY dept           → bucket rows into groups
  4. HAVING COUNT(*) > 5     → filter groups (group-level)
  5. SELECT dept, COUNT(*)   → project final columns
  6. ORDER BY cnt DESC       → sort result
  7. LIMIT 10                → restrict output rows
```

This logical order is exactly why the earlier 12-stage execution pipeline places:
- **Filtering (WHERE)** before **Joins/Grouping/Sorting** (Stage 9 before Stage 10)
- **Projection (SELECT)** near the very end (Stage 11), just before the result is returned (Stage 12)

---

## 4. Grammar of Other Common SQL Statements

### INSERT
```sql
INSERT INTO table_name (col1, col2, ...)
VALUES (val1, val2, ...);
```

### UPDATE
```sql
UPDATE table_name
SET col1 = val1, col2 = val2
WHERE condition;
```

### DELETE
```sql
DELETE FROM table_name
WHERE condition;
```

### CREATE TABLE
```sql
CREATE TABLE table_name (
    column1 datatype [constraints],
    column2 datatype [constraints],
    PRIMARY KEY (column1),
    FOREIGN KEY (column2) REFERENCES other_table(column)
);
```

### JOIN Syntax Variants
```sql
-- INNER JOIN
SELECT a.col, b.col
FROM table_a a
JOIN table_b b ON a.id = b.a_id;

-- LEFT JOIN
SELECT a.col, b.col
FROM table_a a
LEFT JOIN table_b b ON a.id = b.a_id;

-- Multiple joins
SELECT a.col, b.col, c.col
FROM table_a a
JOIN table_b b ON a.id = b.a_id
JOIN table_c c ON b.id = c.b_id;
```

---

## 5. Quick Reference — Grammar vs Execution Cheat Sheet

| Rule | Explanation |
|------|-------------|
| Written order ≠ Execution order | `SELECT` is written first but executed 6th logically |
| `WHERE` before `GROUP BY`/`HAVING` | Because raw row filtering happens before aggregation |
| `HAVING` after `GROUP BY` | Filters aggregated groups, not raw rows |
| `SELECT` alias unusable in `WHERE`/`HAVING`/`GROUP BY` | These execute before `SELECT` computes the alias |
| `SELECT` alias usable in `ORDER BY` | `ORDER BY` executes after `SELECT` |
| `DISTINCT` applies after `SELECT` | Duplicates are removed only after final columns are chosen |
| `LIMIT` executes last | Applied only after sorting is complete |

---

## Common Interview Follow-Ups

1. **Can you use `WHERE` and `HAVING` in the same query?**
   Yes — `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation. They serve different stages and can coexist.

2. **What happens if you `GROUP BY` without any aggregate function?**
   It still groups rows into unique combinations of the grouped column(s), essentially acting like a `DISTINCT` on those columns.

3. **Why does `SELECT *` sometimes perform worse than selecting explicit columns?**
   Even though projection (Stage 11) happens late, `SELECT *` may force the storage engine to fetch full rows even when a covering index could have satisfied a narrower column list — bypassing potential index-only scans.

4. **Is `ORDER BY` guaranteed on a `GROUP BY` query without an explicit `ORDER BY`?**
   No — grouping order is **not guaranteed** unless you explicitly add `ORDER BY`. Never rely on implicit ordering.