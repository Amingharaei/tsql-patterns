# Top-N rows per group

We can't use TOP(3) to get the top three orders for *each* customer, here I'll explain how to tackle such a scenario and at the end I'll show what happens when rows tie for the same value at the cut-off point.

## Create a table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id     INT PRIMARY KEY,
    order_date   DATE,
    customer_id  INT,
    order_amount DECIMAL(10,2)
);

INSERT INTO orders_test (order_id, order_date, customer_id, order_amount) VALUES
(1,  '2025-01-05', 1,  100.00),
(2,  '2025-01-10', 1,  200.00),
(3,  '2025-01-15', 2,  300.00),
(4,  '2025-01-20', 2,  400.00),
(5,  '2025-01-25', 3,  500.00),
(6,  '2025-02-10', 3,  800.00),
(7,  '2025-02-15', 1, 1400.00),
(8,  '2025-02-20', 2,  900.00),
(9,  '2025-03-01', 3, 1600.00),
(10, '2025-03-10', 1, 1800.00),
(11, '2025-03-20', 2, 2000.00),
(12, '2025-04-01', 1, 2400.00),
(13, '2025-04-10', 2, 2500.00);

```

## Numbering each customer's orders, then keeping the top three

A simple `TOP (3)` gives us the three biggest orders in the whole table, not three per customer. Instead, we number the orders inside each customer group with `ROW_NUMBER` and then sort them so we get the top 3.

```sql
WITH ranked_orders AS (
    SELECT
        customer_id,
        order_id,
        order_date,
        order_amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id
                           ORDER BY order_amount DESC, order_id DESC) AS rn
    FROM orders_test
)
SELECT
    customer_id,
    order_id,
    order_date,
    order_amount,
    rn
FROM ranked_orders
WHERE rn <= 3
ORDER BY
    customer_id,
    rn;

```
I added `order_id DESC` as a tiebreaker so the ordering is unique and deterministic. Without it, two orders with the same amount could swap numbers between runs. Here is the result of our query:

```text
customer_id  order_id  order_date  order_amount  rn
-----------  --------  ----------  ------------  --
1            12        2025-04-01  2400.00       1
1            10        2025-03-10  1800.00       2
1            7         2025-02-15  1400.00       3
2            13        2025-04-10  2500.00       1
2            11        2025-03-20  2000.00       2
2            8         2025-02-20  900.00        3
3            9         2025-03-01  1600.00       1
3            6         2025-02-10  800.00        2
3            5         2025-01-25  500.00        3

```

Customer 3 only has three orders, so all of them come back. Customers 1 and 2 each have five, so only their top three make it through.

## What happens when orders tie at the cut-off?

`ROW_NUMBER` always generates consecutive, distinct numbers, even when values are equal. If your third and fourth rows share the exact same amount, the tiebreaker forces one row to get number 3 and the other to get number 4.

If the business rule says you must keep every row that tied for third place, you need to swap `ROW_NUMBER` for `RANK` or `DENSE_RANK`.

Reset the table to see how different ranking functions handle ties:

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id     INT PRIMARY KEY,
    order_date   DATE,
    customer_id  INT,
    order_amount DECIMAL(10,2)
);

INSERT INTO orders_test (order_id, order_date, customer_id, order_amount) VALUES
(1, '2025-01-05', 1, 500.00), -- ties for first
(2, '2025-01-10', 1, 500.00), -- ties for first
(3, '2025-01-15', 1, 500.00), -- ties for first
(4, '2025-01-20', 1, 300.00); 

```

Here we have three orders tied at 500.00, and one lower order at 300.00. The point is how `RANK` and `DENSE_RANK` number these rows differently:

```sql
SELECT
    customer_id,
    order_id,
    order_amount,
    RANK() OVER (PARTITION BY customer_id 
                 ORDER BY order_amount DESC) AS rnk,
    DENSE_RANK() OVER (PARTITION BY customer_id 
                       ORDER BY order_amount DESC) AS drnk
FROM orders_test;

```

```text
customer_id  order_id  order_amount  rnk  drnk
-----------  --------  ------------  ---  ----
1            1         500.00        1    1
1            2         500.00        1    1
1            3         500.00        1    1
1            4         300.00        4    2

```

Because three rows are tied for the first place, `RANK` skips numbers and jumps to **4** for the next row. So if you filter `WHERE rnk <= 3`, then the order with the order_amount = 300.00  won't be included. 

However, if the goal is to find the orders with the top 3 *distinct* amounts, we must use `DENSE_RANK`. It does not skip numbers, so the 300.00 order gets rank 2 and will be included in the `<= 3` filter.

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;

```
