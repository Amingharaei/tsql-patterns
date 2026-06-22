# Finding the gaps
Here, I'll show you three methods to find the gaps in a sequence of values in a column and at the end I'll explain how to handle scenarios when a gap occurs at the start of the sequence.

## Create a table with artificial data in it

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id     INT PRIMARY KEY,
    order_amount DECIMAL(10,2)
);

INSERT INTO orders_test (order_id, order_amount) VALUES
(1,  120.00),
(2,   80.00),
(3,  200.00),
(7,   55.00),   -- gaps: 4, 5, 6
(8,  300.00),
(10, 150.00),   -- gap: 9
(14,  90.00),   -- gaps: 11, 12, 13
(15, 130.00);

```

## First approach: self-join to the next number

The idea is to look at every order ID and find the smallest order ID that comes after it. If that next number is more than one step away, then we have a gap. The number right after the current one is the start of the gap, and the number right before the next existing one is the end.

```sql
SELECT
    cur.order_id + 1        AS gap_start,
    MIN(nxt.order_id) - 1   AS gap_end
FROM orders_test AS cur
INNER JOIN orders_test AS nxt
    ON nxt.order_id > cur.order_id
GROUP BY cur.order_id
HAVING MIN(nxt.order_id) > cur.order_id + 1;

```
This method is the classic way to solve this before window functions existed, and it does not scale well. Matching every row to all the rows above it, is a lot of work as the table grows, and the query gets slow.

## Second approach: LEAD inside a CTE

The `LEAD` function does the same job in a single pass instead of a join. For each row, it looks at the next `order_id` in order. If the distance to that next number is bigger than one, we found a gap.

```sql
WITH ordered_ids AS (
    SELECT
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_order_id
    FROM orders_test
)
SELECT
    order_id + 1        AS gap_start,
    next_order_id - 1   AS gap_end
FROM ordered_ids
WHERE next_order_id - order_id > 1;

```

## Third approach: same idea using a derived table instead of a CTE

```sql
SELECT
    order_id + 1        AS gap_start,
    next_order_id - 1   AS gap_end
FROM (
    SELECT
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_order_id
    FROM orders_test
) AS g
WHERE next_order_id - order_id > 1;

```

All three queries return the same three ranges:

```text
gap_start  gap_end
---------  -------
4          6
9          9
11         13

```

## What happens when a gap occurs at the start of the sequence?

These solutions I showed share a limitation: they only look for gaps *between* rows that already exist in the table. If a business rule says the sequence must start at 1, but the first row in our table is 4, the queries will miss the missing 1 to 3 range because there is no lower row to start the calculation from.

Reset the table with data where the sequence starts late:

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id     INT PRIMARY KEY,
    order_amount DECIMAL(10,2)
);

INSERT INTO orders_test (order_id, order_amount) VALUES
(4,  120.00),   -- sequence starts at 4 instead of 1
(5,   80.00),
(6,  200.00),
(9,   55.00);

```

Running the previous queries against this data returns only the internal gap:

```text
gap_start  gap_end
---------  -------
7          8

```

To fix this and catch the missing numbers at the start, we can inject a **virtual anchor** row using `UNION ALL` before we run the window function. If the sequence must start at 1, we can add a dummy row with an ID of 0.

```sql
WITH anchored_source AS (
    SELECT order_id FROM orders_test
    UNION ALL
    SELECT 0 AS order_id -- sets up 1 as the minimum starting point
),
ordered_ids AS (
    SELECT
        order_id,
        LEAD(order_id) OVER (ORDER BY order_id) AS next_order_id
    FROM anchored_source
)
SELECT
    order_id + 1        AS gap_start,
    next_order_id - 1   AS gap_end
FROM ordered_ids
WHERE next_order_id - order_id > 1;

```

Now the query checks the step between 0 and 4, which finds the missing boundary range alongside the internal gaps:

```text
gap_start  gap_end
---------  -------
1          3
7          8

```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;

```