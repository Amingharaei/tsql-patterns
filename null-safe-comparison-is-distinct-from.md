# Null-safe comparison with IS DISTINCT FROM

`NULL = NULL` doesn't return true, it returns *unknown*, so the moment we compare two nullable columns to check whether a value changed, ordinary `=` and `<>` give us the wrong answer.

`IS [NOT] DISTINCT FROM` is the null-safe comparison that fixes it. At the end I'll show the workaround we had to use before SQL Server 2022.

## The trap, in two rows

Before any table, this is the whole problem. Run it:

```sql
SELECT
    CASE WHEN NULL = NULL THEN 'equal' ELSE 'not equal' END AS with_equals,
    CASE WHEN NULL IS NOT DISTINCT FROM NULL THEN 'same' ELSE 'different' END AS null_safe;
```

```text
with_equals  null_safe
-----------  ---------
not equal    same
```

`=` says two NULLs are "not equal" (really it evaluated to unknown, which falls through to the `ELSE`), while `IS NOT DISTINCT FROM` treats NULL as a known value and correctly says they're the same. Here is the full truth table for the two operators:

```text
A     B     A = B     A IS NOT DISTINCT FROM B
----  ----  --------  ------------------------
0     0     True      True
0     1     False     False
0     NULL  Unknown   False
NULL  NULL  Unknown   True
```

## Detecting changed rows

Run this in a test database. `target_test` is what we already have; `source_test` is the incoming batch. Some columns are nullable, and that is exactly where ordinary comparison will fail.

```sql
DROP TABLE IF EXISTS target_test;
DROP TABLE IF EXISTS source_test;

CREATE TABLE target_test (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(50),
    phone       VARCHAR(20) NULL,
    region      VARCHAR(20) NULL
);

CREATE TABLE source_test (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(50),
    phone       VARCHAR(20) NULL,
    region      VARCHAR(20) NULL
);

INSERT INTO target_test (customer_id, email, phone, region) VALUES
(1, 'a@x.com', '111',  'Tehran'),
(2, 'b@x.com', NULL,   'Shiraz'),   -- phone is currently unknown
(3, 'c@x.com', '333',  NULL);       -- region is currently unknown

INSERT INTO source_test (customer_id, email, phone, region) VALUES
(1, 'a@x.com', '111',  'Tehran'),
(2, 'b@x.com', '999',  'Shiraz'),   -- phone went from NULL to 999
(3, 'c@x.com', '333',  NULL);       -- still unknown
```

The obvious way to find changed rows will give us wrong results:

```sql
SELECT s.customer_id
FROM source_test AS s
JOIN target_test AS t ON s.customer_id = t.customer_id
WHERE s.email  <> t.email
   OR s.phone  <> t.phone
   OR s.region <> t.region;
```

This returns **no rows**. For customer 2, `'999' <> NULL` is unknown, not true, so the change won't be detected. Now the null-safe version:

```sql
SELECT s.customer_id
FROM source_test AS s
JOIN target_test AS t ON s.customer_id = t.customer_id
WHERE s.email  IS DISTINCT FROM t.email
   OR s.phone  IS DISTINCT FROM t.phone
   OR s.region IS DISTINCT FROM t.region;
```

```text
customer_id
-----------
2
```

Customer 2 is correctly flagged (phone went from `NULL` to `999`), and customer 3 is correctly left alone (`NULL` region on both sides counts as the same).

It works wherever a boolean is expected, including a join condition, so we can also use it to match rows on a nullable key where two `NULL`s should join to each other:

```sql
SELECT s.customer_id
FROM source_test AS s
JOIN target_test AS t
    ON s.customer_id = t.customer_id
   AND s.region IS NOT DISTINCT FROM t.region;
```

```text
customer_id
-----------
1
2
3
```

## What to do before SQL Server 2022?

If we are on an older version, we have to apply the null-safe logic in the code. The fully correct expansion of `s.phone IS DISTINCT FROM t.phone` is this:

```sql
WHERE (s.phone <> t.phone OR s.phone IS NULL OR t.phone IS NULL)
  AND NOT (s.phone IS NULL AND t.phone IS NULL)
   -- ...repeated for every column
```
Another approach which is more readable:

```sql
WHERE (s.phone IS NULL AND t.phone IS NOT NULL)
   OR (s.phone IS NOT NULL AND t.phone IS NULL)
   OR (s.phone <> t.phone)
   -- ...repeated for every column
   ```

## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS target_test;
DROP TABLE IF EXISTS source_test;
```
