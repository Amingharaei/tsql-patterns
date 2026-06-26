# Walking an org chart with a recursive CTE
Here I'll explain how to create a complete top-down organization hierarchy, as well as how to find a specific employee's level and a path from that employee down. 
I will also demonstrate how to limit the results to a specific depth, and finally, how to perform a bottom-up traversal to see the management chain from an individual employee up to the root.

## Create a table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS employees_test;

CREATE TABLE employees_test (
    employee_id INT PRIMARY KEY,
    first_name  VARCHAR(50),
    last_name   VARCHAR(50),
    job_title   VARCHAR(50),
    manager_id  INT
);

CREATE NONCLUSTERED INDEX ncix_employees_test_manager_id ON employees_test(manager_id);

INSERT INTO employees_test (employee_id, first_name, last_name, job_title, manager_id) VALUES
(1,  'James',    'Smith',  'CEO',           NULL),
(2,  'Jennifer', 'Allen',  'VP of Sales',   1),
(3,  'Michael',  'Lee',    'VP of Tech',    1),
(4,  'Sarah',    'Connor', 'Sales Manager', 2),
(5,  'John',     'Doe',    'Sales Rep',     4),
(6,  'Emily',    'Davis',  'Sales Rep',     4),
(7,  'David',    'Wilson', 'Tech Lead',     3),
(8,  'Jessica',  'Brown',  'Developer',     7),
(9,  'Daniel',   'Miller', 'Developer',     7),
(10, 'Robert',   'Taylor', 'Intern',        9);

```

## Top-Down Organization Hierarchy

```sql
WITH org_chart AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        job_title,
        manager_id,
        1 AS hierarchy_level,
        CAST(job_title AS VARCHAR(MAX)) AS hierarchy_path
    FROM employees_test
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.job_title,
        e.manager_id,
        oc.hierarchy_level + 1,
        CAST(oc.hierarchy_path + '- ' + e.job_title AS VARCHAR(MAX))
    FROM employees_test AS e
    INNER JOIN org_chart AS oc
        ON e.manager_id = oc.employee_id
)
SELECT
    hierarchy_level,
    employee_id,
    first_name + ' ' + last_name AS full_name,
    job_title,
    hierarchy_path
FROM org_chart
ORDER BY hierarchy_level, manager_id;

```

```text
hierarchy_level  employee_id  full_name       job_title      hierarchy_path
---------------  -----------  --------------  -------------  ----------------------------------------------
1                1            James Smith     CEO            CEO
2                2            Jennifer Allen  VP of Sales    CEO- VP of Sales
2                3            Michael Lee     VP of Tech     CEO- VP of Tech
3                4            Sarah Connor    Sales Manager  CEO- VP of Sales- Sales Manager
3                7            David Wilson    Tech Lead      CEO- VP of Tech- Tech Lead
4                5            John Doe        Sales Rep      CEO- VP of Sales- Sales Manager- Sales Rep
4                6            Emily Davis     Sales Rep      CEO- VP of Sales- Sales Manager- Sales Rep
4                8            Jessica Brown   Developer      CEO- VP of Tech- Tech Lead- Developer
4                9            Daniel Miller   Developer      CEO- VP of Tech- Tech Lead- Developer
5                10           Robert Taylor   Intern         CEO- VP of Tech- Tech Lead- Developer- Intern

```

SQL Server limits recursive CTEs to 100 iterations by default. For hierarchies deeper than that, we nned to add OPTION (MAXRECURSION N) at the end of the query, where N is the maximum depth taht we expect, and there is also the OPTION (MAXRECURSION 0) to remove the cap entirely which is not recommended to be used blindly.

## How to get only an employee's part of the tree?

To get everyone under a particular employee instead of the whole company, we point the anchor at that person rather than at the `NULL` root.

Here we start from employee 3:

```sql
WITH org_chart AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        job_title,
        manager_id,
        1 AS hierarchy_level,
        CAST(job_title AS VARCHAR(MAX)) AS hierarchy_path
    FROM employees_test
    WHERE employee_id = 3

    UNION ALL
    
    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.job_title,
        e.manager_id,
        oc.hierarchy_level + 1,
        CAST(oc.hierarchy_path + '- ' + e.job_title AS VARCHAR(MAX))
    FROM employees_test AS e
    INNER JOIN org_chart AS oc
        ON e.manager_id = oc.employee_id
)
SELECT
    hierarchy_level,
    employee_id,
    first_name + ' ' + last_name AS full_name,
    job_title,
    hierarchy_path
FROM org_chart
ORDER BY hierarchy_level, manager_id;

```

```text
hierarchy_level  employee_id  full_name       job_title      hierarchy_path
---------------  -----------  --------------  -------------  ----------------------------------------------
1                3            Michael Lee     VP of Tech     VP of Tech
2                7            David Wilson    Tech Lead      VP of Tech- Tech Lead
3                8            Jessica Brown   Developer      VP of Tech- Tech Lead- Developer
3                9            Daniel Miller   Developer      VP of Tech- Tech Lead- Developer
4                10           Robert Taylor   Intern         VP of Tech- Tech Lead- Developer- Intern

```

## What if the requirement is to limit the output to hierarchy level 3?

If we only want to see the organization down to a specific depth, like hierarchy_level 3, the efficient way to limit the depth is to put the filter in the recursive member.

```sql
WITH org_chart AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        job_title,
        manager_id,
        1 AS hierarchy_level,
        CAST(job_title AS VARCHAR(MAX)) AS hierarchy_path
    FROM employees_test
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.job_title,
        e.manager_id,
        oc.hierarchy_level + 1,
        CAST(oc.hierarchy_path + '- ' + e.job_title AS VARCHAR(MAX))
    FROM employees_test AS e
    INNER JOIN org_chart AS oc
        ON e.manager_id = oc.employee_id
    WHERE oc.hierarchy_level < 3 -- prevents generating rows past level 3
)
SELECT
    hierarchy_level,
    employee_id,
    first_name + ' ' + last_name AS full_name,
    job_title,
    hierarchy_path
FROM org_chart
ORDER BY
    hierarchy_level,
    manager_id;

```

```text
hierarchy_level  employee_id  full_name       job_title      hierarchy_path
---------------  -----------  --------------  -------------  ----------------------------------------------
1                1            James Smith     CEO            CEO
2                2            Jennifer Allen  VP of Sales    CEO- VP of Sales
2                3            Michael Lee     VP of Tech     CEO- VP of Tech
3                4            Sarah Connor    Sales Manager  CEO- VP of Sales- Sales Manager
3                7            David Wilson    Tech Lead      CEO- VP of Tech- Tech Lead

```

## Finding all managers above a specific employee

To get the management chain going up the hierarchy from a specific employee, we seed the anchor member with the target `employee_id`. 

In the recursive member, we need to invert the join condition: 
So instead of matching the manager's ID to the employees' manager ID, we join the current row's `manager_id` to the parent table's `employee_id`.

Here we find the management chain above employee 5 (John Doe):

```sql
WITH org_management_chain AS (
    SELECT
        employee_id,
        first_name,
        last_name,
        job_title,
        manager_id,
        1 AS hierarchy_level,
        CAST(job_title AS VARCHAR(MAX)) AS management_path
    FROM employees_test
    WHERE employee_id = 5

    UNION ALL

    SELECT
        e.employee_id,
        e.first_name,
        e.last_name,
        e.job_title,
        e.manager_id,
        omc.hierarchy_level + 1,
        CAST(e.job_title + '- ' + omc.management_path AS VARCHAR(MAX))
    FROM employees_test AS e
    INNER JOIN org_management_chain AS omc
        ON omc.manager_id = e.employee_id
)
SELECT
    hierarchy_level,
    employee_id,
    first_name + ' ' + last_name AS full_name,
    job_title,
    management_path
FROM org_management_chain
ORDER BY hierarchy_level;

```

```text
hierarchy_level  employee_id  full_name       job_title      management_path
---------------  -----------  --------------  -------------  ----------------------------------------------
1                5            John Doe        Sales Rep      Sales Rep
2                4            Sarah Connor    Sales Manager  Sales Manager- Sales Rep
3                2            Jennifer Allen  VP of Sales    VP of Sales- Sales Manager- Sales Rep
4                1            James Smith     CEO            CEO- VP of Sales- Sales Manager- Sales Rep

```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS employees_test;

```