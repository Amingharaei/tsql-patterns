# First and last value of each group

`FIRST_VALUE` and `LAST_VALUE` repeat their answer on every row of the group, so getting just the first and last amount onto one row per customer takes a bit more.
At the end I will also show an alternative way of solving the same problem and when it breaks.

## Create a table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    customer_id  INT,
    order_id     INT,
    order_date   DATE,
    order_amount DECIMAL(10,2)
);

INSERT INTO orders_test (customer_id, order_id, order_date, order_amount) VALUES
(1, 101, '2025-01-05', 100.00),
(1, 102, '2025-01-10', 200.00),
(1, 103, '2025-02-15', 250.00),
(2, 201, '2025-01-08', 300.00),
(2, 202, '2025-01-20', 400.00),
(3, 301, '2025-01-12', 500.00),
(3, 302, '2025-02-25', 600.00),
(3, 303, '2025-03-10', 520.00);

```

## One row per customer with FIRST_VALUE and LAST_VALUE

`FIRST_VALUE` and `LAST_VALUE` give us the values we want, but because they are window functions they return their answer on every single row of the partition. If a customer has six orders, we get the same first and last value repeated six times. To collapse it down to one row per customer, we add a `ROW_NUMBER` and keep only row 1.

```sql
WITH order_bounds AS (
    SELECT
        customer_id,
        FIRST_VALUE(order_amount) OVER (PARTITION BY customer_id
                                        ORDER BY order_date, order_id) AS first_order_amount,
        LAST_VALUE(order_amount)  OVER (PARTITION BY customer_id
                                        ORDER BY order_date, order_id
                                        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING) AS last_order_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id 
                           ORDER BY order_date ASC, order_id ASC) AS rn
    FROM orders_test
)
SELECT
    customer_id,
    first_order_amount,
    last_order_amount
FROM order_bounds
WHERE rn = 1;

```

```text
customer_id  first_order_amount  last_order_amount
-----------  ------------------  -----------------
1            100.00              250.00
2            300.00              400.00
3            500.00              520.00

```
FIRST_VALUE works correctly with the default frame because the first row of the partition is always within reach. LAST_VALUE needs the explicit ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING because its default frame only looks as far as the current row — without it, on the first row of the partition, LAST_VALUE would return that same first row's value instead of the actual last one.

## Another way of solving without Window functions

We can get the same result as before by capturing the first and last order_dates using group by inside a CTE, and then join them back to the table.

```sql
WITH bounds AS (
    SELECT
        customer_id,
        MIN(order_date) AS first_date,
        MAX(order_date) AS last_date
    FROM orders_test
    GROUP BY customer_id
)
SELECT
    b.customer_id,
    f.order_amount AS first_order_amount,
    l.order_amount AS last_order_amount
FROM bounds AS b
INNER JOIN orders_test AS f
    ON f.customer_id = b.customer_id 
    AND f.order_date = b.first_date
INNER JOIN orders_test AS l
    ON l.customer_id = b.customer_id 
    AND l.order_date = b.last_date;

```

This pattern won't return correct results if a customer has multiple orders on their first or last days, the join will multiply the rows out and return duplicates. 

In such cases, the window function method with a strict, unique tiebreaker is our safest option for multi-row days.

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;

```