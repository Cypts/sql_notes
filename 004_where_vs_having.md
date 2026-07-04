# SQL: HAVING vs WHERE — Detailed Notes + Query Error Analysis

---

## 1. Core Conceptual Difference

| Aspect | `WHERE` | `HAVING` |
|--------|---------|----------|
| Operates on | Individual rows (before grouping) | Groups / aggregated results (after grouping) |
| Execution order | Runs **before** `GROUP BY` | Runs **after** `GROUP BY` |
| Can reference aggregate functions? | ❌ No (`SUM`, `AVG`, `COUNT`, etc. not allowed) | ✅ Yes |
| Can reference raw/non-aggregated columns? | ✅ Yes | ⚠️ Only if that column is in `GROUP BY`, or the whole table is being treated as one group and every referenced value is unambiguous (rare) |
| Typical use | Filter rows based on column values | Filter groups based on aggregate conditions |
| Used without GROUP BY? | Yes, commonly | Yes, but then the *entire table* is treated as a single group |

---

## 2. Why WHERE Can't Use Aggregate Functions

Recall the **logical execution order**:
```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

`WHERE` executes **before** grouping and aggregation even happen. At that point in execution, values like `AVG(salary)` or `COUNT(*)` **don't exist yet** — there's nothing to aggregate over, because rows haven't been grouped. This is why:

```sql
SELECT department, COUNT(*)
FROM employees
WHERE COUNT(*) > 5   -- ❌ ERROR
GROUP BY department;
```
fails — you must use `HAVING` instead, since it runs after aggregation:
```sql
SELECT department, COUNT(*)
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;  -- ✅ Works
```

---

## 3. Now Let's Break Down Your Query

```sql
SELECT * FROM Employee HAVING salary > AVG(salary);
```

This fails — but understanding **why** requires looking closely, because the error isn't simply "you used HAVING without GROUP BY" (that part is actually legal). The real problem is more subtle.

### Step 1: HAVING Without GROUP BY Is Legal — But Changes Meaning
When there's no `GROUP BY`, SQL treats the **entire table as a single group**. So `HAVING` is allowed to reference aggregate functions computed over that one big group:
```sql
SELECT COUNT(*) FROM Employee HAVING COUNT(*) > 10;  -- ✅ Valid
```
This works fine because `COUNT(*)` is a single aggregate value representing the *whole table* — there's nothing ambiguous about it.

### Step 2: The Real Problem — Mixing an Aggregated Value with a Non-Aggregated Column
Your query does this:
```sql
HAVING salary > AVG(salary)
```

Here:
- `AVG(salary)` → a **single aggregate value** (fine, computed over the whole table since there's no GROUP BY).
- `salary` (on its own, not wrapped in an aggregate function) → refers to an **individual row's value**.

But since the entire table is being treated as **one group**, `salary` on its own is **ambiguous** — which row's salary should it use? The whole point of a "group" in HAVING context is that it produces exactly **one row of output per group**. Since there's one group (the whole table), HAVING can only see **aggregated/summarized** values — not the value of `salary` for any specific individual row.

> This is the exact same rule as the "GROUP BY golden rule" from before:
> **Every column referenced in a HAVING/SELECT clause, when grouping is involved (even a single implicit group), must either be aggregated, or be part of the grouping columns.**
>
> Since there's no `GROUP BY column` here, and `salary` isn't wrapped in an aggregate function, it violates this rule.

### Step 3: Why SELECT * Makes It Worse
```sql
SELECT * FROM Employee ...
```
`SELECT *` tries to return **every column of every row** — but `HAVING` (with no `GROUP BY`) collapses the result into a single summarized group. There's a fundamental conflict:
- `HAVING` (in this context) expects to return **one summarized row**.
- `SELECT *` expects to return **all raw rows and columns**.

You cannot mix "give me every individual row" with "treat everything as one aggregated group" in the same query.

### The Correct Error (conceptually)
Most databases will throw something like:
```
ERROR: column "employee.salary" must appear in the GROUP BY clause or be used in an aggregate function
```

---

## 4. How to Actually Write What You Probably Meant

If your intent is: **"Show all employees whose salary is greater than the average salary in the company"** — this is a classic **subquery** problem, not a GROUP BY/HAVING problem at all.

### ✅ Correct Approach — Use a Subquery in WHERE
```sql
SELECT *
FROM Employee
WHERE salary > (SELECT AVG(salary) FROM Employee);
```

**Why this works:**
- The subquery `(SELECT AVG(salary) FROM Employee)` is evaluated **first**, independently, producing a single scalar value (e.g., `65000`).
- The outer query then filters **individual rows** using `WHERE`, comparing each row's `salary` against that single computed number.
- No grouping or aggregation conflict — `WHERE` here compares a row-level column to a **pre-computed constant**, not to a live aggregate function call.

---

## 5. Side-by-Side: What Works and What Doesn't

| Query | Valid? | Why |
|-------|--------|-----|
| `SELECT AVG(salary) FROM Employee;` | ✅ | Aggregate over whole table, one output row |
| `SELECT COUNT(*) FROM Employee HAVING COUNT(*) > 10;` | ✅ | HAVING references only the aggregate, no raw column mixed in |
| `SELECT department, AVG(salary) FROM Employee GROUP BY department HAVING AVG(salary) > 50000;` | ✅ | `department` is grouped, `AVG(salary)` is aggregated — consistent |
| `SELECT * FROM Employee HAVING salary > AVG(salary);` | ❌ | Mixes raw column (`salary`, via `SELECT *`) with an aggregate, without GROUP BY |
| `SELECT * FROM Employee WHERE salary > (SELECT AVG(salary) FROM Employee);` | ✅ | Subquery computes the aggregate separately; WHERE compares row values to a constant |

---

## 6. Mental Model to Remember This Forever

> **HAVING filters *groups*, not individual rows.**
> If you want to compare an individual row's raw column to an aggregate computed over the whole table (or some other subset), you need a **subquery** or a **window function** — not `HAVING`.

### Bonus: Window Function Alternative
You can also solve the "employees earning above average" problem using a window function, which computes the average **without collapsing rows** — keeping every individual row intact:
```sql
SELECT *
FROM (
    SELECT *, AVG(salary) OVER () AS avg_salary
    FROM Employee
) sub
WHERE salary > avg_salary;
```
Here, `AVG(salary) OVER ()` computes the average across all rows but **still returns one row per employee**, unlike `GROUP BY`, which collapses rows into summarized groups. This avoids the row-vs-aggregate conflict entirely.

---

## 7. Quick Interview Recap

1. **Can HAVING be used without GROUP BY?**
   Yes — the whole table is treated as one group, and HAVING can filter based on an aggregate over that group.

2. **Why does `HAVING salary > AVG(salary)` fail without GROUP BY?**
   Because `salary` is a per-row value while `AVG(salary)` is a single aggregated value over the whole table (one implicit group) — you can't reference an individual row's column when the query context expects only grouped/aggregated results.

3. **How do you find rows where a column exceeds the table's average?**
   Use a subquery in `WHERE` (`WHERE col > (SELECT AVG(col) FROM table)`), or a window function (`AVG(col) OVER ()`) — not `HAVING`.

4. **Difference between subquery approach and window function approach?**
   - Subquery: computes the aggregate once, separately, then filters rows against that constant. Usually two logical passes over data (though the optimizer may optimize this).
   - Window function: computes the aggregate **alongside** each row in a single pass, without collapsing the row set — more flexible for combining aggregates with per-row detail (e.g., "each employee's salary compared to their department's average," which needs `AVG(salary) OVER (PARTITION BY department)`).