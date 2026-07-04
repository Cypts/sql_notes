### `SELECT 1` in SQL (Short Note)

* `SELECT 1` returns the constant value **1** for every row that matches the `WHERE` condition.
* It **does not** mean "select the first row."
* It is commonly used with `EXISTS` because `EXISTS` only checks whether **any row is returned**, not what the row contains.

**Example:**

```sql
SELECT 1
FROM Employee
WHERE departmentId = 1;
```

If there are 2 employees in department 1, the result is:

| 1 |
| - |
| 1 |
| 1 |

If there are no matching employees, the result is **no rows**.

**With `EXISTS`:**

```sql
SELECT d.name
FROM Department d
WHERE EXISTS (
    SELECT 1
    FROM Employee e
    WHERE e.departmentId = d.id
);
```

Here, `SELECT 1` is used only to check if at least one employee exists in the department. The value `1` is ignored by `EXISTS`; only the presence of rows matters.
