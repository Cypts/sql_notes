# SQL GROUP BY & Aggregate Functions — Detailed Notes

---

## 1. What is an Aggregate Function?

An **aggregate function** performs a calculation on a **set of rows** and returns a **single summarized value**. Unlike scalar functions (which operate row-by-row), aggregates collapse multiple rows into one result.

### Common Aggregate Functions

| Function | Purpose | Ignores NULLs? |
|----------|---------|-----------------|
| `COUNT()` | Counts number of rows/values | `COUNT(col)` ignores NULLs; `COUNT(*)` counts all rows including NULLs |
| `SUM()` | Adds up numeric values | Yes |
| `AVG()` | Calculates average of numeric values | Yes |
| `MIN()` | Returns smallest value | Yes |
| `MAX()` | Returns largest value | Yes |
| `GROUP_CONCAT()` / `STRING_AGG()` | Concatenates values from multiple rows into one string (MySQL / PostgreSQL respectively) | Yes |
| `VARIANCE()` / `STDDEV()` | Statistical variance/standard deviation | Yes |

### Key Behavior Notes
- **`COUNT(*)` vs `COUNT(column)`**:
  - `COUNT(*)` counts **all rows**, regardless of NULLs.
  - `COUNT(column)` counts only rows where that **column is NOT NULL**.
  ```sql
  SELECT COUNT(*) FROM employees;         -- total rows
  SELECT COUNT(bonus) FROM employees;     -- rows where bonus IS NOT NULL
  ```
- **`COUNT(DISTINCT column)`** counts unique non-null values:
  ```sql
  SELECT COUNT(DISTINCT department) FROM employees;
  ```
- Aggregate functions **ignore NULL values** in their calculations (except `COUNT(*)`).
  ```sql
  -- If salary column has values: 50000, NULL, 70000
  SELECT AVG(salary) FROM employees;  -- averages only 50000 and 70000, NOT divided by 3
  ```
- If a group has **no rows at all**, `SUM`/`AVG`/`MAX`/`MIN` return `NULL`, but `COUNT` returns `0`.

---

## 2. What is GROUP BY?

`GROUP BY` groups rows that share the **same value(s)** in specified column(s) into summary rows. It is almost always used **together with aggregate functions** to compute per-group statistics.

### Basic Syntax
```sql
SELECT column1, AGG_FUNC(column2)
FROM table_name
[WHERE condition]
GROUP BY column1
[HAVING agg_condition]
[ORDER BY column1];
```

### Example
```sql
SELECT department, COUNT(*) AS total_employees, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
```
This returns **one row per unique department**, with the count and average salary computed within each group.

### The Golden Rule of GROUP BY
> Every column in the `SELECT` list must either:
> 1. Be included in the `GROUP BY` clause, **or**
> 2. Be wrapped inside an aggregate function.

```sql
-- ❌ INVALID (in strict SQL mode / PostgreSQL / standard SQL)
SELECT department, employee_name, COUNT(*)
FROM employees
GROUP BY department;
-- employee_name is neither grouped nor aggregated → error
```

```sql
-- ✅ VALID
SELECT department, COUNT(*) AS emp_count
FROM employees
GROUP BY department;
```

> **Note:** MySQL historically allowed "loose" GROUP BY (returning an arbitrary row's value for non-aggregated, non-grouped columns) unless `ONLY_FULL_GROUP_BY` mode is enabled. This is considered bad practice and disallowed by ANSI SQL standard and most other RDBMS (PostgreSQL, SQL Server, Oracle).

---

## 3. Grouping by Multiple Columns

You can group by more than one column — this creates a group for **every unique combination** of the specified columns.

```sql
SELECT department, job_title, COUNT(*) AS cnt
FROM employees
GROUP BY department, job_title;
```
This produces one row per **(department, job_title)** pair, not per department alone.

---

## 4. WHERE vs HAVING (Critical Distinction)

| Aspect | `WHERE` | `HAVING` |
|--------|---------|----------|
| Operates on | Individual rows | Groups (after aggregation) |
| Executes | Before `GROUP BY` | After `GROUP BY` |
| Can use aggregate functions? | ❌ No | ✅ Yes |
| Purpose | Filters raw data before grouping | Filters summarized/grouped results |

### Example Showing Both Together
```sql
SELECT department, COUNT(*) AS active_employees
FROM employees
WHERE status = 'active'        -- filter rows BEFORE grouping
GROUP BY department
HAVING COUNT(*) > 10;          -- filter groups AFTER aggregation
```

> **Interview trap:** Trying to filter on an aggregate using `WHERE` will throw an error:
> ```sql
> SELECT department, COUNT(*) 
> FROM employees
> WHERE COUNT(*) > 10   -- ❌ ERROR: aggregate functions not allowed in WHERE
> GROUP BY department;
> ```
> The correct version uses `HAVING`.

---

## 5. Execution Order Recap (Where GROUP BY Fits)

```
FROM  → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

- Rows are first fetched and filtered (`WHERE`).
- Remaining rows are bucketed into groups (`GROUP BY`).
- Aggregate functions are computed **per group**.
- Groups are then filtered (`HAVING`).
- Finally, `SELECT` projects out the desired columns/aggregates.

This is why `HAVING` can reference aggregate results, but `WHERE` cannot — by the time `WHERE` runs, aggregation hasn't happened yet.

---

## 6. GROUP BY with ORDER BY

Grouped results have **no guaranteed order** unless explicitly sorted:
```sql
SELECT department, COUNT(*) AS cnt
FROM employees
GROUP BY department
ORDER BY cnt DESC;   -- sort groups by employee count, descending
```
Never assume `GROUP BY` output is sorted by the grouping column — always add `ORDER BY` if order matters.

---

## 7. GROUP BY with Expressions

You can group by computed expressions, not just raw columns:
```sql
SELECT YEAR(hire_date) AS hire_year, COUNT(*) AS hires
FROM employees
GROUP BY YEAR(hire_date);
```
This groups employees by the year they were hired, extracted from a date column.

---

## 8. ROLLUP, CUBE, and GROUPING SETS (Advanced Grouping)

These extend `GROUP BY` to produce **subtotal and grand-total rows** — commonly asked in advanced interviews.

### ROLLUP
Produces subtotals at each level of a hierarchy, plus a grand total.
```sql
SELECT department, job_title, COUNT(*) AS cnt
FROM employees
GROUP BY ROLLUP(department, job_title);
```
This returns:
- One row per (department, job_title) combo
- A subtotal row per department (job_title = NULL)
- A grand total row (department = NULL, job_title = NULL)

### CUBE
Produces subtotals for **every possible combination** of the grouped columns (not just hierarchical).
```sql
SELECT department, job_title, COUNT(*) AS cnt
FROM employees
GROUP BY CUBE(department, job_title);
```

### GROUPING SETS
Lets you explicitly specify which grouping combinations you want, instead of all combinations.
```sql
SELECT department, job_title, COUNT(*) AS cnt
FROM employees
GROUP BY GROUPING SETS ((department, job_title), (department), ());
```

---

## 9. Aggregate Functions Without GROUP BY

If you use an aggregate function **without** a `GROUP BY` clause, the entire table is treated as **one single group**.
```sql
SELECT COUNT(*), AVG(salary) FROM employees;
```
This returns exactly **one row** — total count and average salary across all rows in the table.

---

## 10. Common Interview Questions on GROUP BY / Aggregates

1. **Difference between `WHERE` and `HAVING`?**
   `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation and can use aggregate functions.

2. **Difference between `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)`?**
   - `COUNT(*)` → all rows.
   - `COUNT(column)` → non-NULL values in that column.
   - `COUNT(DISTINCT column)` → unique non-NULL values.

3. **Can you use `GROUP BY` without any aggregate function?**
   Yes — it behaves like `DISTINCT` on the grouped columns, returning one row per unique combination.

4. **What happens with `AVG()` when there are NULL values in the column?**
   NULLs are excluded from both the sum and the count used to compute the average — they don't count as zero.

5. **Why does adding a non-aggregated, non-grouped column to SELECT cause an error (in strict SQL)?**
   Because for each group, there could be many different values for that column across the rows — SQL doesn't know which one to display, so the engine forces you to either group by it or aggregate it.

6. **How would you find departments having more than 5 employees earning above 50,000?**
   ```sql
   SELECT department, COUNT(*) AS emp_count
   FROM employees
   WHERE salary > 50000
   GROUP BY department
   HAVING COUNT(*) > 5;
   ```

7. **Classic problem — Nth Highest Salary using GROUP BY/Aggregate concepts:**
   ```sql
   SELECT DISTINCT salary
   FROM employees
   ORDER BY salary DESC
   LIMIT 1 OFFSET N-1;
   ```
   (Alternative: use `DENSE_RANK()` window function for a more robust solution with ties.)

8. **Can `HAVING` be used without `GROUP BY`?**
   Yes — in that case, the entire table is treated as a single group, and `HAVING` filters based on an aggregate computed over the whole table.
   ```sql
   SELECT COUNT(*) AS total
   FROM employees
   HAVING COUNT(*) > 100;
   ```