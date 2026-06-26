# Finding the islands (consecutive runs)

Islands are the unbroken runs in a sequence. Here we'll see the quick trick for clean integer sequences, and a general method that also works for dates, and at the end how the same idea finds runs of the same *value*, not just consecutive numbers.

## Create a table with artificial data in it

Run this in a test database.

```sql
DROP TABLE IF EXISTS orders_test;

CREATE TABLE orders_test (
    order_id INT PRIMARY KEY
);

INSERT INTO orders_test (order_id) VALUES
(1), (2), (3),        -- island 1 to 3
(7), (8),             -- island 7 to 8
(10),                 -- island 10
(14), (15);           -- island 14 to 15
```

## First approach: the row-number trick

If we are dealing with integer sequences, we can subtract a row number from each value, every row inside the same unbroken sequence, gives the *same* result, because both sides go up by one in step. So, the moment there is a gap, the value jumps ahead of the row number and a new sequence begins. So that difference becomes a group key for the island.

```sql
WITH grouped AS (
    SELECT
        order_id,
        order_id - ROW_NUMBER() OVER (ORDER BY order_id) AS island_key
    FROM orders_test
)
SELECT
    MIN(order_id) AS island_start,
    MAX(order_id) AS island_end,
    COUNT(*)      AS length
FROM grouped
GROUP BY island_key
ORDER BY island_start;
```

It helps to see the intermediate `island_key`: rows 1, 2, 3 give `1-1`, `2-2`, `3-3`, all `0`; then 7, 8 give `7-4`, `8-5`, both `3`; and so on. Grouping by that key collapses each run:

```text
island_start  island_end  length
------------  ----------  ------
1             3           3
7             8           2
10            10          1
14            15          2
```

This approach is handy, but it only works when the step is a fixed 1 and the values are unique. For dates, for decimals, or for "a gap means more than 7 days," it won't work.

## Second approach: flag where each island starts, then run a total

In this approach we look back one row with `LAG`. If the distance to the previous value is bigger than the allowed step (or there is no previous row), this row starts a new island, so we mark it with a 1. 

A running total of those marks then numbers the islands, and we group by that number.

```sql
WITH flagged AS (
    SELECT
        order_id,
        CASE
            WHEN LAG(order_id) OVER (ORDER BY order_id) IS NULL
              OR order_id - LAG(order_id) OVER (ORDER BY order_id) > 1
            THEN 1 ELSE 0
        END AS is_new_island
    FROM orders_test
),
islands AS (
    SELECT
        order_id,
        SUM(is_new_island) OVER (ORDER BY order_id
                                 ROWS UNBOUNDED PRECEDING) AS island_id
    FROM flagged
)
SELECT
    MIN(order_id) AS island_start,
    MAX(order_id) AS island_end,
    COUNT(*)      AS length
FROM islands
GROUP BY island_id
ORDER BY island_start;
```

It gives us the same result as the first approach, but now the rule for "what breaks a run" is in one place: the `> 1` . So we can change it to `> 7` for a one-week tolerance, or swap the arithmetic for dates with `DATEDIFF`, without touching the rest.

## What if an island is a repetition of the same value, not a number sequence?

Here we are dealing with things like a status log, with values like "active", "idle", "error".

```sql
DROP TABLE IF EXISTS status_log_test;

CREATE TABLE status_log_test (
    reading_id INT PRIMARY KEY,
    status     VARCHAR(10)
);

INSERT INTO status_log_test (reading_id, status) VALUES
(1, 'active'),
(2, 'active'),
(3, 'active'),
(4, 'idle'),
(5, 'idle'),
(6, 'active'),
(7, 'error'),
(8, 'error');
```

The point here is to have two row numbers: one over the whole set, and one restarted per status.
In a consecutive run of the same status, both increase by one together, so their difference is constant, just like the same idea as the first approach, but now it isolates runs of equal values.

```sql
WITH grouped AS (
    SELECT
        reading_id,
        status,
        ROW_NUMBER() OVER (ORDER BY reading_id)
      - ROW_NUMBER() OVER (PARTITION BY status ORDER BY reading_id) AS island_key
    FROM status_log_test
)
SELECT
    status,
    MIN(reading_id) AS start_reading,
    MAX(reading_id) AS end_reading,
    COUNT(*)        AS readings
FROM grouped
GROUP BY status, island_key
ORDER BY start_reading;
```

```text
status  start_reading  end_reading  readings
------  -------------  -----------  --------
active  1              3            3
idle    4              5            2
active  6              6            1
error   7              8            2
```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS orders_test;
DROP TABLE IF EXISTS status_log_test;
```
