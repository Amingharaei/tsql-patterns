# Counting orders in a rolling three-day window

Counting orders in the last three days looks simple until we face two complications: 
a single day can hold multiple orders, so stepping back two rows is not the same as stepping back two days, and days with no orders are missing from the result set completely, which causes the window to silently include more calendar days than intended.

## Create a table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id   INT PRIMARY KEY,
    order_date DATE
);

INSERT INTO orders_test (order_id, order_date) VALUES
(1, '2025-07-25'),
(2, '2025-07-25'),
(3, '2025-07-26'),
(4, '2025-07-27'),
(5, '2025-07-28'),
(6, '2025-07-28'),
(7, '2025-07-29'),
(8, '2025-07-30'),
(9, '2025-07-31');

```

Because a single day can hold several orders, we cannot simply go back two rows with a window frame. Two rows is not always the same as two days.

So what we need to do is to sum the daily counts over three days, and then bring that number back down to each order.

## Step 1: add up the daily counts over three days

First we run a window aggregate over the rows grouped by each order_date, using the frame `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` that covers the current day and the two days before it.

```sql
SELECT
    order_date,
    SUM(COUNT(*)) OVER (ORDER BY order_date
                        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS orders_in_last_3_days
FROM orders_test
GROUP BY order_date;

```

```text
order_date  orders_in_last_3_days
----------  ---------------------
2025-07-25  2
2025-07-26  3
2025-07-27  4
2025-07-28  4
2025-07-29  4
2025-07-30  4
2025-07-31  3
```


## Step 2: bring the total back to each order

The above result is per day, but we need it per order. So we wrap the query above in a CTE and join it back to the orders on the order_date, so every order gets its three-day total.

```sql
WITH daily AS (
    SELECT
        order_date,
        SUM(COUNT(*)) OVER (ORDER BY order_date
                            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS orders_in_last_3_days
    FROM orders_test
    GROUP BY order_date
)
SELECT
    o.order_id,
    o.order_date,
    d.orders_in_last_3_days
FROM orders_test AS o
INNER JOIN daily AS d
    ON o.order_date = d.order_date;

```

```text
order_id  order_date  orders_in_last_3_days
--------  ----------  ---------------------
1         2025-07-25  2
2         2025-07-25  2
3         2025-07-26  3
4         2025-07-27  4
5         2025-07-28  4
6         2025-07-28  4
7         2025-07-29  4
8         2025-07-30  4
9         2025-07-31  3

```

So as you can see both orders on 25th of July report 2, because that day has two orders and nothing comes before it. The 26th reports 3 (two on the 25th plus one on the 26th), and so on.

## What happens when a day has no orders?

The `ROWS` frame here counts back over the last three *rows that exist in the CTE*, not the last three calendar days. 
As long as the dates in the data are consecutive they mean the same thing, but the moment a day has zero transactions, it is completely missing from the `GROUP BY` output.

Then what happens is that the window function skips the missing calendar gap and grabs the next available row position, covering **more than three actual calendar days**.

Reset the table with a gap, leaving no orders on the 27th or 28th:

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id   INT PRIMARY KEY,
    order_date DATE
);

INSERT INTO orders_test (order_id, order_date) VALUES
(1, '2025-07-25'),
(2, '2025-07-26'),
(3, '2025-07-29'),   -- the 27th and 28th had no orders
(4, '2025-07-30');

```

If we run the query again, here is what we get:

```text
order_id  order_date  orders_in_last_3_days
--------  ----------  ---------------------
1         2025-07-25  1
2         2025-07-26  2
3         2025-07-29  3
4         2025-07-30  3

```

The order on the 29th reports 3. But the actual three-calendar-day window ending on the 29th is the 27th, 28th, and 29th, which has only one single order. 

**So how do we solve it?**

If SQL Server had support for Range + Interval, we could solve this with `RANGE BETWEEN INTERVAL '2' DAY PRECEDING AND CURRENT ROW`, which counts by values rather than rows. However, T-SQL does not implement date intervals inside window frames yet.

So we have to give the window the actual dates themselves rather than whichever rows happen to exist.

```sql
SELECT
    o.order_id,
    o.order_date,
    (SELECT COUNT(*)
     FROM orders_test AS o2
     WHERE o2.order_date BETWEEN DATEADD(day, -2, o.order_date) AND o.order_date)
        AS orders_in_last_3_calendar_days
FROM orders_test AS o;
```

```text
order_id  order_date  orders_in_last_3_calendar_days
--------  ----------  ------------------------------
1         2025-07-25  1
2         2025-07-26  2
3         2025-07-29  1
4         2025-07-30  2
```

So now the 29th correctly reports 1.

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;

```