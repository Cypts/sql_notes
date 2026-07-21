Detailed Notes on Working Table and Result Table in SQL
Introduction

When SQL executes a query, it does not directly produce the final output in a single step. Instead, it processes the query in a series of logical stages.

At each stage, SQL creates an intermediate table called a Working Table (also called an intermediate result set). After all stages are completed, SQL returns the Result Table, which is the final output displayed to the user.

Important: Working tables are part of SQL's logical processing model. Database engines may optimize execution and not physically create each one.

What is a Working Table?
Definition

A Working Table is a temporary table created internally by the SQL engine while executing a query.

It stores the intermediate results after each logical processing step.

The user cannot normally see these tables.

Characteristics
Temporary
Created automatically by SQL
Exists only while the query is executing
Cannot be accessed directly
Used to process the next clause of the query
What is a Result Table?
Definition

The Result Table is the final table produced after SQL completes processing all clauses in the query.

This is the table displayed to the user.

Characteristics
Final output
Visible to the user
Returned by the database
Can be stored using statements like CREATE TABLE AS or inserted into another table
Difference Between Working Table and Result Table
Working Table	Result Table
Temporary	Final output
Invisible to user	Visible to user
Created after each SQL clause	Created after all processing
Used internally	Returned to the user
May be optimized away by the database	Represents the query result
SQL Logical Processing Order

Although we write SQL like this:

SELECT Name, Salary
FROM Employees
WHERE Department = 'IT'
ORDER BY Salary DESC;


SQL processes it in the following logical order:

FROM
WHERE
GROUP BY
HAVING
SELECT
DISTINCT
ORDER BY
LIMIT / TOP / FETCH

A working table is produced after each stage, and the last stage produces the result table.

Example 1
Employees Table
EmpID	Name	Department	Salary
1	Alice	HR	50000
2	Bob	IT	70000
3	Charlie	IT	60000
4	David	HR	55000
5	Eva	Sales	65000

Query:

SELECT Name, Salary
FROM Employees
WHERE Department='IT'
ORDER BY Salary DESC;

Step 1: FROM Clause

SQL first reads the table.

Working Table 1
EmpID	Name	Department	Salary
1	Alice	HR	50000
2	Bob	IT	70000
3	Charlie	IT	60000
4	David	HR	55000
5	Eva	Sales	65000
Step 2: WHERE Clause

Condition:

Department='IT'


SQL removes rows that do not satisfy the condition.

Working Table 2
EmpID	Name	Department	Salary
2	Bob	IT	70000
3	Charlie	IT	60000
Step 3: SELECT Clause

Columns selected:

Name
Salary

Working Table 3
Name	Salary
Bob	70000
Charlie	60000
Step 4: ORDER BY Clause

Sort salaries in descending order.

Result Table
Name	Salary
Bob	70000
Charlie	60000
Example 2 (GROUP BY)
Sales Table
Product	Quantity	Price
Laptop	2	60000
Laptop	1	60000
Phone	3	25000
Phone	2	25000
Tablet	4	30000

Query:

SELECT Product,
       SUM(Quantity) AS TotalQty
FROM Sales
GROUP BY Product;

Working Table 1 (FROM)

Entire Sales table.

Working Table 2 (GROUP BY)

Rows are grouped.

Laptop Group
Product	Quantity
Laptop	2
Laptop	1
Phone Group
Product	Quantity
Phone	3
Phone	2
Tablet Group
Product	Quantity
Tablet	4
Working Table 3 (SELECT)

Aggregate values are calculated.

Product	TotalQty
Laptop	3
Phone	5
Tablet	4

This becomes the result table.

Example 3 (HAVING)
SELECT Department,
       AVG(Salary)
FROM Employees
GROUP BY Department
HAVING AVG(Salary)>55000;

Step 1

Working table after FROM:

Entire Employees table.

Step 2

Working table after GROUP BY:

HR

50000

55000

Average = 52500

IT

70000

60000

Average = 65000

Sales

65000

Average = 65000

Step 3

HAVING removes groups whose average salary is not greater than 55000.

Remaining groups:

Department	Average Salary
IT	65000
Sales	65000
Result Table
Department	AVG(Salary)
IT	65000
Sales	65000
Visual Representation
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
LIMIT/TOP
       │
       ▼
RESULT TABLE

Why Does SQL Use Working Tables?

Working tables help SQL process each clause in the correct order.

For example:

WHERE filters rows before grouping.
GROUP BY creates groups.
HAVING filters those groups.
SELECT chooses the output columns.
ORDER BY sorts the final rows.

Each step depends on the output of the previous step.

Important Points
A working table is temporary and used internally.
A result table is the final output shown to the user.
There may be several working tables during query execution.
Modern database systems often optimize execution and may not physically create every working table.
Understanding working tables makes it easier to understand why SQL clauses behave the way they do (for example, why WHERE cannot use aggregate functions but HAVING can).
Interview Questions
1. What is a working table?

A working table is a temporary intermediate table created by the SQL engine during query execution. It stores partial results after each logical processing stage and is used internally to process the next stage.

2. What is a result table?

A result table is the final table returned by SQL after all query clauses have been processed. It is the output displayed to the user.

3. Is a working table visible to the user?

No. Working tables are internal to the SQL engine and are not normally visible.

4. What is the main difference between a working table and a result table?

A working table is a temporary intermediate result used during query execution, while a result table is the final output returned to the user.

5. Does SQL always physically create working tables?

Not necessarily. The concept of working tables describes the logical execution order. Database optimizers may process queries in more efficient ways without materializing every intermediate table.
