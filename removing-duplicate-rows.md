# Removing duplicate rows from a table
Here I'll explain how to remove duplicate rows from a table and I also explain at the end how we can handle a scenario when a specific set of rows are more important for us and we need to keep them.

## Create a table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id     INT,
    order_date   DATE,
    customer_id  INT,
    order_amount DECIMAL(10,2)
);

INSERT INTO orders_test (order_id, order_date, customer_id, order_amount) VALUES
(1, '2026-01-05', 1, 100.00),
(2, '2026-01-10', 2, 200.00),
(3, '2026-01-15', 3, 300.00),
(4, '2026-01-20', 4, 400.00),
(5, '2026-01-25', 5, 500.00);

-- We duplicate the whole table twice, so every row now appears four times.
INSERT INTO orders_test SELECT * FROM orders_test;
INSERT INTO orders_test SELECT * FROM orders_test;

```

## Numbering the copies, then deleting the extras

The trick is to give the rows something so that we can differentiate between them. 

We use the `ROW_NUMBER` function to number the copies inside each set of identical rows and then we delete everything that is above 1.

We also need to partition by every single column so that each group represents one set of identical rows.

```sql
WITH numbered_rows AS (
    SELECT
        ROW_NUMBER() OVER (
            PARTITION BY order_id, order_date, customer_id, order_amount
            ORDER BY (SELECT NULL)
        ) AS rn
    FROM orders_test
)
DELETE FROM numbered_rows
WHERE rn > 1;

```

## Check the result

```sql
SELECT
    order_id,
    order_date,
    customer_id,
    order_amount
FROM orders_test;

```

We are back to the five unique rows:

```text
order_id  order_date  customer_id  order_amount
--------  ----------  -----------  ------------
1         2026-01-05  1            100.00
2         2026-01-10  2            200.00
3         2026-01-15  3            300.00
4         2026-01-20  4            400.00
5         2026-01-25  5            500.00

```

### why did we order by using (SELECT NULL)?

Because there was no meaningful order among the identical rows and we **didn't care which copy to keep**, so we ordered by `(SELECT NULL)`. 
This tells SQL Server to skip the sorting step and just number the rows as it finds them.


## What happens when one copy is better than the rest?

In the scenario above, the duplicates were truly identical, so it made no difference which copy we kept. But sometimes rows share the same business keys but have an audit column, like a timestamp, where one record is clearly the most current or accurate one to keep.

Reset the table with data where the rows share the same keys but have different entry timestamps:

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id     INT,
    order_date   DATE,
    customer_id  INT,
    order_amount DECIMAL(10,2),
    entry_time   DATETIME
);

INSERT INTO orders_test (order_id, order_date, customer_id, order_amount, entry_time) VALUES
(1, '2026-01-05', 1, 100.00, '2026-01-05 09:00:00'),
(1, '2026-01-05', 1, 100.00, '2026-01-05 14:30:00'), -- latest copy for order 1
(2, '2026-01-10', 2, 200.00, '2026-01-10 10:15:00'), -- latest copy for order 2
(2, '2026-01-10', 2, 200.00, '2026-01-10 08:00:00'),
(3, '2026-01-12', 2, 190.00, '2026-01-12 08:48:00');

```

Now if we want to keep only the newest record for each order, we cannot use `(SELECT NULL)` anymore. We must explicitly order by `entry_time DESC` inside the window partition so the latest row gets marked as 1.

```sql
WITH numbered_rows AS (
    SELECT
        ROW_NUMBER() OVER (
            PARTITION BY order_id, order_date, customer_id, order_amount
            ORDER BY entry_time DESC
        ) AS rn
    FROM orders_test
)
DELETE FROM numbered_rows
WHERE rn > 1;

```

Checking the table shows that the older entries are dropped, leaving only the latest timestamps:

```sql
SELECT
    order_id,
    order_date,
    customer_id,
    order_amount,
    entry_time
FROM orders_test;

```

```text
order_id  order_date  customer_id  order_amount  entry_time
--------  ----------  -----------  ------------  -----------------------
1         2026-01-05  1            100.00        2026-01-05 14:30:00.000
2         2026-01-10  2            200.00        2026-01-10 10:15:00.000
3         2026-01-12  2            190.00        2026-01-12 08:48:00.000

```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;
```