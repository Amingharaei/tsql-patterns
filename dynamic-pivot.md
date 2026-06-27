# Dynamic pivot

`PIVOT` makes us name the columns up front, so the day a new category gets added to the underlying data, the report drops it. The fix is to first read the categories from the data and then build the `PIVOT` as a string at runtime.

## Create a table with artificial data in it

Run this in a test database. "Books" only shows up in March, exactly the kind of value a hardcoded pivot would miss.

```sql
DROP TABLE IF EXISTS sales_test;

CREATE TABLE sales_test (
    sales_id   INT IDENTITY(1,1) PRIMARY KEY,
    sales_date DATE,
    category   VARCHAR(25),
    amount     DECIMAL(10,2)
);

INSERT INTO sales_test (sales_date, category, amount) VALUES
('2025-01-15', 'Electronics', 1000.00),
('2025-01-20', 'Clothing',     500.00),
('2025-01-25', 'Home',         750.00),
('2025-02-10', 'Electronics', 1200.00),
('2025-02-15', 'Clothing',     600.00),
('2025-03-05', 'Electronics', 1100.00),
('2025-03-10', 'Clothing',     550.00),
('2025-03-20', 'Home',         800.00),
('2025-03-25', 'Books',        300.00);

```

## Building the column lists and running the query

First we pull the distinct categories and put them into the bracketed, comma-separated list that `PIVOT` expects.

We need to create two separate string lists. 
`@pivot_cols` will hold the raw column names for the internal `IN (...)` pivot clause. 
`@select_cols` will build a `ISNULL([Category], 0.00) AS [Category]` wrapper for the outer query so that missing values become zero instead of `NULL`.

Then we put those column strings into our query text and run it.

```sql
DECLARE @pivot_cols  NVARCHAR(MAX);
DECLARE @select_cols NVARCHAR(MAX);
DECLARE @sql         NVARCHAR(MAX);

SELECT 
    @pivot_cols = STRING_AGG(QUOTENAME(category), ','),
    @select_cols = STRING_AGG(
                        'ISNULL(' + QUOTENAME(category) +
                         ', 0.00) AS ' + QUOTENAME(category),
                          ',')
FROM (SELECT DISTINCT category FROM sales_test) AS c;
IF @pivot_cols IS NULL
BEGIN
    PRINT 'No data to pivot.';
    RETURN;
END;

SET @sql = N'
    SELECT
        sales_month,
        ' + @select_cols + '
    FROM (
        SELECT
            CONVERT(VARCHAR(7), sales_date, 120) AS sales_month,
            category,
            amount
        FROM sales_test
    ) AS src
    PIVOT (
        SUM(amount) FOR category IN (' + @pivot_cols + ')
    ) AS p
    ORDER BY sales_month;';

EXEC sp_executesql @sql;

GO

```

```text
sales_month  Books   Clothing  Electronics  Home
-----------  ------  --------  -----------  ------
2025-01      0.00    500.00    1000.00      750.00
2025-02      0.00    600.00    1200.00      0.00
2025-03      300.00  550.00    1100.00      800.00

```

## Making the query deterministic

`STRING_AGG` function without `ORDER BY` is non-deterministic so we might get different column order in the output between runs. 
In order to make the query deterministic, we need to add `Add WITHIN GROUP (ORDER BY category)` to both STRING_AGG calls so that we get consistent alphabetical ordering every time.

```sql
DECLARE @pivot_cols  NVARCHAR(MAX);
DECLARE @select_cols NVARCHAR(MAX);
DECLARE @sql         NVARCHAR(MAX);

SELECT
    @pivot_cols =
        STRING_AGG(QUOTENAME(category), ',') WITHIN GROUP (ORDER BY category),

    @select_cols =
        STRING_AGG(
            'ISNULL(' + QUOTENAME(category) +
            ', 0.00) AS ' + QUOTENAME(category),
            ','
        )
        WITHIN GROUP (ORDER BY category)
FROM (
    SELECT DISTINCT category
    FROM sales_test
) AS c;

IF @pivot_cols IS NULL
BEGIN
    PRINT 'No data to pivot.';
    RETURN;
END;

SET @sql = N'
    SELECT
        sales_month,
        ' + @select_cols + '
    FROM (
        SELECT
            CONVERT(VARCHAR(7), sales_date, 120) AS sales_month,
            category,
            amount
        FROM sales_test
    ) AS src
    PIVOT (
        SUM(amount) FOR category IN (' + @pivot_cols + ')
    ) AS p
    ORDER BY sales_month;';

EXEC sp_executesql @sql;

```
```text
sales_month  Books   Clothing  Electronics  Home
-----------  ------  --------  -----------  ------
2025-01      0.00    500.00    1000.00      750.00
2025-02      0.00    600.00    1200.00      0.00
2025-03      300.00  550.00    1100.00      800.00

```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS sales_test;
```