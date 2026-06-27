# Dynamic unpivot

Here we read the columns from the catalog and build it dynamically, then deal with the `NULL`s that `UNPIVOT` operator drops.

## Create a wide table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS financial_report_test;

CREATE TABLE financial_report_test (
    product_id     INT,
    product_name   VARCHAR(50),
    [2022_Revenue] DECIMAL(12,2),
    [2023_Revenue] DECIMAL(12,2),
    [2024_Revenue] DECIMAL(12,2),
    [2025_Revenue] DECIMAL(12,2),
    [2026_Revenue] DECIMAL(12,2)
);

INSERT INTO financial_report_test (
    product_id, 
    product_name, 
    [2022_Revenue], 
    [2023_Revenue], 
    [2024_Revenue], 
    [2025_Revenue], 
    [2026_Revenue]
) VALUES
(101, 'Cloud Service', 500.00, 600.00, 800.00, 950.00, 1200.00),
(102, 'On-Prem Lic',   900.00, 850.00, 600.00, 400.00,  300.00),
(103, 'Consulting',    NULL,   200.00, 250.00, 300.00,  450.00);   -- 2022 with no value

```

## Building and running the dynamic unpivot

```sql
DECLARE @columns NVARCHAR(MAX);
DECLARE @sql     NVARCHAR(MAX);
DECLARE @table   NVARCHAR(128) = N'financial_report_test';

SELECT @columns = STRING_AGG(QUOTENAME(c.name), ',') WITHIN GROUP (ORDER BY c.column_id)
FROM sys.columns AS c
WHERE c.object_id = OBJECT_ID(@table)
  AND c.name LIKE '%_Revenue';

IF @columns IS NULL
BEGIN
    PRINT 'No columns to unpivot.';
    RETURN;
END;

SET @sql = N'
    SELECT
        product_id,
        product_name,
        LEFT(period, 4) AS report_year,
        revenue
    FROM ' + QUOTENAME(@table) + '
    UNPIVOT (
        revenue FOR period IN (' + @columns + ')
    ) AS u;';

EXEC sp_executesql @sql;

GO
```

```text
product_id  product_name   report_year  revenue
----------  -------------  -----------  -------
101         Cloud Service  2022         500.00
101         Cloud Service  2023         600.00
101         Cloud Service  2024         800.00
101         Cloud Service  2025         950.00
101         Cloud Service  2026         1200.00
102         On-Prem Lic    2022         900.00
102         On-Prem Lic    2023         850.00
102         On-Prem Lic    2024         600.00
102         On-Prem Lic    2025         400.00
102         On-Prem Lic    2026         300.00
103         Consulting     2023         200.00
103         Consulting     2024         250.00
103         Consulting     2025         300.00
103         Consulting     2026         450.00

```

## What happens when we have Null in the data?

The `UNPIVOT` operator drops `NULL` values automatically. For example, product 103 has no value in the year 2022 and you can see that it has been dropped by the `UNPIVOT`.

If the business requirement dictates that we need to keep the missing values, then we need to use the `CROSS APPLY` operator.

```sql
DECLARE @columns NVARCHAR(MAX);
DECLARE @sql     NVARCHAR(MAX);
DECLARE @table   NVARCHAR(128) = N'financial_report_test';

SELECT @columns = STRING_AGG(
                      '(''' + LEFT(c.name, 4) + ''', f.' + QUOTENAME(c.name) + ')',
                      ',') WITHIN GROUP (ORDER BY c.column_id)
FROM sys.columns AS c
WHERE c.object_id = OBJECT_ID(@table)
  AND c.name LIKE '%_Revenue';

SET @sql = N'
    SELECT
        f.product_id,
        f.product_name,
        u.report_year,
        u.revenue
    FROM ' + QUOTENAME(@table) + ' AS f
    CROSS APPLY (VALUES
        ' + @columns + '
    ) AS u(report_year, revenue);';

EXEC sp_executesql @sql;

```

Now product 103 returns a standard `NULL` value for 2022:

```text
product_id  product_name   report_year  revenue
----------  -------------  -----------  -------
101         Cloud Service  2022         500.00
101         Cloud Service  2023         600.00
.
.
.
103         Consulting     2022         NULL
103         Consulting     2023         200.00
103         Consulting     2024         250.00
103         Consulting     2025         300.00
103         Consulting     2026         450.00
```

`CROSS APPLY` also provides another critical architectural advantage over native `UNPIVOT`. Native `UNPIVOT` fails instantly if the target data types differ, even slightly (such as trying to unpivot a mix of `INT` and `DECIMAL` columns). Inside a `CROSS APPLY (VALUES ...)` loop, you can explicitly add an inline wrapper—like `CAST(f.[2022_Revenue] AS DECIMAL(12,2))`—to enforce type matching before the restructuring step runs.



## What if we need 0 instead of NULL in the output?

To show zero instead of NULL, we need to wrap the incoming table column inside an ISNULL function during the string aggregation. 

This way, we make sure that when the dynamic query string is built, the engine injects the zero-handling logic into the virtual matrix.

```sql
DECLARE @columns NVARCHAR(MAX);
DECLARE @sql     NVARCHAR(MAX);
DECLARE @table   NVARCHAR(128) = N'financial_report_test';

SELECT @columns = STRING_AGG(
                      '(''' + LEFT(c.name, 4) + ''', ISNULL(f.' + QUOTENAME(c.name) + ', 0.00))',
                      ',') WITHIN GROUP (ORDER BY c.column_id)
FROM sys.columns AS c
WHERE c.object_id = OBJECT_ID(@table)
  AND c.name LIKE '%_Revenue';

SET @sql = N'
    SELECT
        f.product_id,
        f.product_name,
        u.report_year,
        u.revenue
    FROM ' + QUOTENAME(@table) + ' AS f
    CROSS APPLY (VALUES
        ' + @columns + '
    ) AS u(report_year, revenue);';

EXEC sp_executesql @sql;
```

```text
product_id  product_name   report_year  revenue
----------  -------------  -----------  -------
101         Cloud Service  2022         500.00
101         Cloud Service  2023         600.00
.
.
.
103         Consulting     2022         0.00
103         Consulting     2023         200.00
103         Consulting     2024         250.00
103         Consulting     2025         300.00
103         Consulting     2026         450.00
```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS financial_report_test;

```