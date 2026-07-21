# Working Table and Result Table in SQL

## Introduction

When SQL executes a query, it does **not** directly produce the final output in one step. Instead, it processes the query through a series of logical stages.

At each stage, SQL creates an **intermediate table** called a **Working Table**. After all stages are completed, SQL produces the **Result Table**, which is the final output displayed to the user.

> **Note:** Working tables are part of SQL's logical query processing model. Modern database systems may optimize execution and may not physically create every working table.

---

# What is a Working Table?

## Definition

A **Working Table** is a temporary table created internally by the SQL engine during query execution.

It stores intermediate results after each logical processing step.

The user cannot normally see these tables.

### Characteristics

- Temporary
- Created automatically by SQL
- Exists only while the query is executing
- Cannot be accessed directly by users
- Used internally for processing the next SQL clause

---

# What is a Result Table?

## Definition

A **Result Table** is the final table produced after SQL completes processing all clauses in a query.

This table is returned to the user as the query output.

### Characteristics

- Final output of the query
- Visible to the user
- Returned by the database
- Can be stored into another table if required

---

# Difference Between Working Table and Result Table

| Working Table | Result Table |
|---------------|--------------|
| Temporary | Final output |
| Created internally | Returned to the user |
| Invisible to users | Visible to users |
| Used during query execution | Produced after query execution |
| May exist multiple times | Only one final result |

---

# SQL Logical Processing Order

Although SQL queries are written as:

```sql
SELECT Name, Salary
FROM Employees
WHERE Department = 'IT'
ORDER BY Salary DESC;
```

SQL logically processes them in this order:

1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. DISTINCT
7. ORDER BY
8. LIMIT / TOP / FETCH

Each stage produces a **Working Table**, and the final stage produces the **Result Table**.

---

# Example 1

## Employees Table

| EmpID | Name | Department | Salary |
|------:|------|------------|-------:|
|1|Alice|HR|50000|
|2|Bob|IT|70000|
|3|Charlie|IT|60000|
|4|David|HR|55000|
|5|Eva|Sales|65000|

Query:

```sql
SELECT Name, Salary
FROM Employees
WHERE Department = 'IT'
ORDER BY Salary DESC;
```

---

## Step 1 — FROM Clause

SQL reads the Employees table.

### Working Table 1

| EmpID | Name | Department | Salary |
|------:|------|------------|-------:|
|1|Alice|HR|50000|
|2|Bob|IT|70000|
|3|Charlie|IT|60000|
|4|David|HR|55000|
|5|Eva|Sales|65000|

---

## Step 2 — WHERE Clause

Condition:

```sql
WHERE Department = 'IT'
```

Rows not matching the condition are removed.

### Working Table 2

| EmpID | Name | Department | Salary |
|------:|------|------------|-------:|
|2|Bob|IT|70000|
|3|Charlie|IT|60000|

---

## Step 3 — SELECT Clause

Only the required columns are selected.

### Working Table 3

| Name | Salary |
|------|-------:|
|Bob|70000|
|Charlie|60000|

---

## Step 4 — ORDER BY Clause

Rows are sorted in descending order of salary.

### Result Table

| Name | Salary |
|------|-------:|
|Bob|70000|
|Charlie|60000|

---

# Example 2 — GROUP BY

## Sales Table

| Product | Quantity | Price |
|---------|---------:|------:|
|Laptop|2|60000|
|Laptop|1|60000|
|Phone|3|25000|
|Phone|2|25000|
|Tablet|4|30000|

Query:

```sql
SELECT Product,
       SUM(Quantity) AS TotalQty
FROM Sales
GROUP BY Product;
```

---

## Working Table 1

Entire Sales table is read.

---

## Working Table 2 — GROUP BY

Rows are grouped by Product.

### Laptop Group

| Product | Quantity |
|---------|---------:|
|Laptop|2|
|Laptop|1|

### Phone Group

| Product | Quantity |
|---------|---------:|
|Phone|3|
|Phone|2|

### Tablet Group

| Product | Quantity |
|---------|---------:|
|Tablet|4|

---

## Working Table 3 — SELECT

Aggregate functions are calculated.

| Product | TotalQty |
|---------|---------:|
|Laptop|3|
|Phone|5|
|Tablet|4|

This becomes the Result Table.

---

# Example 3 — HAVING

Query:

```sql
SELECT Department,
       AVG(Salary) AS AvgSalary
FROM Employees
GROUP BY Department
HAVING AVG(Salary) > 55000;
```

---

## Step 1 — FROM

Entire Employees table.

---

## Step 2 — GROUP BY

Groups are formed.

### HR

50000

55000

Average = 52500

### IT

70000

60000

Average = 65000

### Sales

65000

Average = 65000

---

## Step 3 — HAVING

Groups with average salary greater than 55000 remain.

### Working Table

| Department | AvgSalary |
|------------|----------:|
|IT|65000|
|Sales|65000|

---

## Step 4 — SELECT

Required columns are selected.

---

## Result Table

| Department | AvgSalary |
|------------|----------:|
|IT|65000|
|Sales|65000|

---

# Query Processing Flow

```text
Original Table
      │
      ▼
FROM
      │
Working Table 1
      │
      ▼
WHERE
      │
Working Table 2
      │
      ▼
GROUP BY
      │
Working Table 3
      │
      ▼
HAVING
      │
Working Table 4
      │
      ▼
SELECT
      │
Working Table 5
      │
      ▼
DISTINCT
      │
Working Table 6
      │
      ▼
ORDER BY
      │
Working Table 7
      │
      ▼
LIMIT / TOP
      │
      ▼
RESULT TABLE
```

---

# Why Does SQL Use Working Tables?

Working tables allow SQL to process each clause in the correct logical order.

For example:

- WHERE filters rows before grouping.
- GROUP BY creates groups.
- HAVING filters groups.
- SELECT chooses the output columns.
- ORDER BY sorts the final rows.

Each step depends on the output of the previous step.

---

# Important Points

- A Working Table is temporary.
- A Result Table is the final output.
- Working tables are normally invisible to users.
- SQL may create multiple working tables during execution.
- Database optimizers may avoid physically creating every working table.
- Working tables are a logical concept used to explain SQL execution.

---

# Summary

| Feature | Working Table | Result Table |
|----------|---------------|--------------|
| Purpose | Intermediate processing | Final output |
| Visibility | Internal only | Visible to users |
| Lifetime | Temporary | Until returned to the client |
| Created During | Every logical processing stage | End of query execution |
| Accessible | No | Yes |

---

# Interview Questions

### 1. What is a Working Table?

A Working Table is a temporary intermediate table created internally by the SQL engine during query execution. It stores partial results after each logical processing stage.

---

### 2. What is a Result Table?

A Result Table is the final output returned to the user after SQL finishes executing all query clauses.

---

### 3. Is a Working Table visible to users?

No. Working tables are internal to the SQL engine and are not normally accessible.

---

### 4. Can SQL create multiple Working Tables?

Yes. SQL may logically create a new Working Table after processing each clause such as `FROM`, `WHERE`, `GROUP BY`, and `SELECT`.

---

### 5. Does SQL physically create every Working Table?

Not always. Working tables describe the **logical** execution model. Modern database engines often optimize execution and may avoid materializing every intermediate table.
