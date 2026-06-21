# Running count of unique customers per salesperson

When there is the need to calculate the running count of *unique* values instead of a running total of rows, we need to be aware that in T-SQL, `DISTINCT` cannot be used inside a window function, so `COUNT(DISTINCT customer_id) OVER (...)` will not run. So we have to find a way around that.

Here we see how to calculate how many separate customers a salesperson has served as their orders come in over time, where a customer who returns is still counted only once. At the end I'll also explain a scenario in which two orders fall on the same day.

## Create a table with artificial data in it

Make sure to run this in a test database. The table I used here is small so I didn't create the necessary indexes on it.

```sql
DROP TABLE IF EXISTS sales_orders_test;

CREATE TABLE sales_orders_test (
    order_id     INT,
    order_date   DATE,
    staff_id     INT,            -- the salesperson who handled the order
    customer_id  INT,
    order_total  DECIMAL(10,2)
);

INSERT INTO sales_orders_test (order_id, order_date, staff_id, customer_id, order_total) VALUES
(1001, '2025-03-03', 1, 100, 240.00),
(1002, '2025-03-05', 1, 101,  80.00),
(1003, '2025-03-09', 1, 100,  55.00),   -- customer 100 again
(1004, '2025-03-14', 1, 102, 320.00),
(1005, '2025-03-20', 1, 101,  60.00),   -- customer 101 again
(1006, '2025-03-04', 2, 200, 410.00),
(1007, '2025-03-08', 2, 201,  90.00),
(1008, '2025-03-12', 2, 200, 130.00),   -- customer 200 again
(1009, '2025-03-18', 2, 202,  75.00);
```

## First approach: mark each customer's first order, then count the marks

In the first approach we give every customer's first order with a salesperson a mark, and leave their later orders blank. The `ROW_NUMBER` function numbers a customer's orders per salesperson, so the row numbered 1 is the order where that customer shows up the first time. A plain `COUNT` over the marks then gives the answer, because `COUNT` ignores NULLs and the repeated orders hold nothing.

```sql
WITH flagged AS (
    SELECT
        order_id,
        order_date,
        staff_id,
        customer_id,
        order_total,
        CASE
            WHEN ROW_NUMBER() OVER (PARTITION BY staff_id, customer_id
                                    ORDER BY order_date) = 1
            THEN customer_id
        END AS first_time_customer
    FROM sales_orders_test
)
SELECT
    order_id,
    order_date,
    staff_id,
    customer_id,
    order_total,
    COUNT(first_time_customer) OVER (PARTITION BY staff_id
                                     ORDER BY order_date) AS unique_customers_so_far
FROM flagged
ORDER BY
    staff_id,
    order_date;
```
## Second approach: count only the first occurrence

This approach is a slight variation of the previous one, here we assign a row number to each customer per staff_id, ordered by the order_date. Then in the outer query, we count only the rows where the row_num = 1 .

```sql
WITH flagged AS (
    SELECT
        order_id,
        order_date,
        staff_id,
        customer_id,
        order_total,
        ROW_NUMBER() OVER (PARTITION BY staff_id, customer_id
                            ORDER BY order_date) AS row_num
    FROM sales_orders_test
)
SELECT
    order_id,
    order_date,
    staff_id,
    customer_id,
    order_total,
    COUNT(CASE WHEN row_num = 1 THEN 1 END) OVER (PARTITION BY staff_id
                                                   ORDER BY order_date) AS unique_customers_so_far
FROM flagged
ORDER BY
    staff_id,
    order_date;
```

Both queries return the same result:

```text
order_id  order_date  staff_id  customer_id  order_total  unique_customers_so_far
--------  ----------  --------  -----------  -----------  -----------------------
1001      2025-03-03  1         100          240.00       1
1002      2025-03-05  1         101           80.00       2
1003      2025-03-09  1         100           55.00       2
1004      2025-03-14  1         102          320.00       3
1005      2025-03-20  1         101           60.00       3
1006      2025-03-04  2         200          410.00       1
1007      2025-03-08  2         201           90.00       2
1008      2025-03-12  2         200          130.00       2
1009      2025-03-18  2         202           75.00       3
```

Orders 1003, 1005 and 1008 are for the repeated customers, so the number does not move on those rows.

## What happens when two orders fall on the same day?

The running count is ordered by `order_date` only, and we didn't specify the window frame. 
If we don't specify the window frame, T-SQL fills in a default: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. 

The part that matters here is that with `RANGE`, rows that share the same `ORDER BY` value (in our case, the same order_date) are treated as one peer group and counted together, so two orders on the same day both get the count as of that whole day.

Reset the table with data that has two orders on the same day:

```sql
DROP TABLE IF EXISTS sales_orders_test;

CREATE TABLE sales_orders_test (
    order_id     INT,
    order_date   DATE,
    staff_id     INT,
    customer_id  INT,
    order_total  DECIMAL(10,2)
);

INSERT INTO sales_orders_test (order_id, order_date, staff_id, customer_id, order_total) VALUES
(2001, '2025-04-01', 1, 100, 150.00),
(2002, '2025-04-03', 1, 101, 200.00),
(2003, '2025-04-03', 1, 102,  90.00),   -- same day as 2002, different customer
(2004, '2025-04-05', 1, 100,  60.00);   -- customer 100 again
```

Run one of the queries above against this table. Orders 2002 and 2003 both fall on 3 April, and both come back with 3:

```sql
WITH flagged AS (
    SELECT
        order_id,
        order_date,
        staff_id,
        customer_id,
        order_total,
        ROW_NUMBER() OVER (PARTITION BY staff_id, customer_id
                            ORDER BY order_date) AS row_num
    FROM sales_orders_test
)
SELECT
    order_id,
    order_date,
    staff_id,
    customer_id,
    order_total,
    COUNT(CASE WHEN row_num = 1 THEN 1 END) OVER (PARTITION BY staff_id
                                                   ORDER BY order_date) AS unique_customers_so_far
FROM flagged
ORDER BY
    staff_id,
    order_date;
```
The output is the same for both approaches:

```text
order_id  order_date  staff_id  customer_id  order_total  unique_customers_so_far
--------  ----------  --------  -----------  -----------  -----------------------
2001      2025-04-01  1         100          150.00       1
2002      2025-04-03  1         101          200.00       3
2003      2025-04-03  1         102           90.00       3
2004      2025-04-05  1         100           60.00       3
```

Both same-day orders show 3, which is the total unique customers as of 3rd of April. If that "as of this date" meaning is the one we want, this is correct and there is nothing more to do about it, but if each order should carry its own step (2, then 3), we need to add `order_id` as a tiebreaker so the ordering is unique. 

With a unique order, there are no peers left, so the count moves one row at a time and at that point `RANGE` and `ROWS` give us the same result. (Switching the frame to `ROWS` on its own is not enough because without a unique tiebreaker the order among the rows with the same order_date is not guaranteed, so the result would not be deterministic.)

```sql
WITH flagged AS (
    SELECT
        order_id,
        order_date,
        staff_id,
        customer_id,
        order_total,
        CASE
            WHEN ROW_NUMBER() OVER (PARTITION BY staff_id, customer_id
                                    ORDER BY order_date, order_id) = 1
            THEN customer_id
        END AS first_time_customer
    FROM sales_orders_test
)
SELECT
    order_id,
    order_date,
    staff_id,
    customer_id,
    order_total,
    COUNT(first_time_customer) OVER (PARTITION BY staff_id
                                     ORDER BY order_date, order_id) AS unique_customers_so_far
FROM flagged
ORDER BY
    staff_id,
    order_date,
    order_id;
```

```text
order_id  order_date  staff_id  customer_id  order_total  unique_customers_so_far
--------  ----------  --------  -----------  -----------  -----------------------
2001      2025-04-01  1         100          150.00       1
2002      2025-04-03  1         101          200.00       2
2003      2025-04-03  1         102           90.00       3
2004      2025-04-05  1         100           60.00       3
```

Now 2002 is 2 and 2003 is 3.

## Cleanup

Run this code if you executed the queries above.
```sql
DROP TABLE IF EXISTS sales_orders_test;
```
