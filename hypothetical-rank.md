# Ranking a hypothetical value within each group

Sometimes the question isn't how the existing values rank, but where a new value *would* be placed if we inserted it into each group.

The standard SQL way to ask this is a hypothetical set function, `RANK(@value) WITHIN GROUP (ORDER BY ...)`, but SQL Server doesn't support it yet, so we need to use a workaround.

## Create a table with artificial data in it

Run this in a test database. Customer 1 has two orders of the same amount, which will matter when we compare rank with dense_rank.

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    customer_id  INT,
    order_id     INT,
    order_amount DECIMAL(12,2)
);

INSERT INTO orders_test (customer_id, order_id, order_amount) VALUES
(1, 101, 320.00),
(1, 102, 320.00),   -- same amount as 101
(1, 103, 180.00),
(1, 104,  95.00),
(2, 201, 250.00),
(2, 202, 150.00),
(2, 203, 290.00),
(3, 301, 110.00),
(3, 302, 165.00),
(3, 303, 310.00);
```

## Hypothetical rank

A value's rank in a **descending** way is just "how many values are greater than that, plus one."

```sql
DECLARE @val DECIMAL(12,2) = 300.00;

SELECT
    customer_id,
    COUNT(CASE WHEN order_amount > @val THEN 1 END) + 1 AS hypothetical_rank_desc
FROM orders_test
GROUP BY customer_id;
```

```text
customer_id  hypothetical_rank_desc
-----------  -----------------
1            3
2            1
3            2
```

For customer 1, two of the four values are greater than 300 (320, 320, 180, 95), so a 300 has the rank 3. For customer 2, the 300 is the greatest value so it gets the rank 1, and for customer 3, the last order (310) is greater than 300, so it gets the rank 2.

## Hypothetical dense rank

When there are ties, DENSE_RANK is different from rank. Rank counts the *rows* above the value, DENSE_RANK counts the *distinct values* above it, so repeated values don't increase the number. 

We can achieve that by adding `DISTINCT` inside the count and counting the amounts.

```sql
DECLARE @val2 DECIMAL(12,2) = 300.00;

SELECT
    customer_id,
    COUNT(DISTINCT CASE WHEN order_amount > @val2 THEN order_amount END) + 1 AS hypothetical_dense_rank_desc
FROM orders_test
GROUP BY customer_id;
```

```text
customer_id  hypothetical_dense_rank_desc
-----------  -----------------------
1            2
2            1
3            2
```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;
```
