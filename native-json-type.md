# Working with the native JSON type

Before SQL Server 2025, we had to store JSON in an `NVARCHAR(MAX)` column and parse it on every read. Now, with the native `JSON` type, we can store the document in a parsed binary format.
This means we can read from a property instead of scanning the whole string, and we can also change one value without rewriting the whole document.
The other good point is that the JSON structure is validated on insert, so a malformed document will be rejected at write time rather than blowing up later when we read it.

> **Preview features:** At the time of this document's creation (June 2026), the following capabilities are in preview and are only available on the on-premises engine: the `.modify()`, `JSON_CONTAINS`, JSON indexes, and array wildcard path expressions.

## Create a table with a JSON column

Run this in a test database.

```sql
DROP TABLE IF EXISTS device_events_test;

CREATE TABLE device_events_test (
    event_id   INT IDENTITY(1,1) PRIMARY KEY,
    device_id  VARCHAR(20),
    event_time DATETIME2 DEFAULT SYSDATETIME(),
    payload    JSON
);

INSERT INTO device_events_test (device_id, payload) VALUES
('Sensor-A', N'{"temperature": 22.5, "humidity": 45, "status": "active",  "tags": ["lab", "ground_floor"]}'),
('Sensor-B', N'{"temperature": 24.1, "humidity": 50, "status": "warning", "error_code": 404}'),
('Sensor-A', N'{"temperature": 21.8, "humidity": 44, "status": "active",  "tags": ["lab"]}'),
('Sensor-C', N'{"temperature": null, "humidity": null, "status": "offline", "tags": []}');
```

## Reading JSON values

`JSON_VALUE` pulls a scalar (a string or number) out of the document by path, and always returns text, so we need to cast it to the type we want. `JSON_QUERY` is the one to use when the thing we are pulling is an object or an array, like the `tags` list.

```sql
SELECT
    event_id,
    device_id,
    JSON_VALUE(payload, '$.status') AS current_status,
    CAST(JSON_VALUE(payload, '$.temperature') AS DECIMAL(5,2)) AS temp_celsius,
    JSON_QUERY(payload, '$.tags') AS tags_array
FROM device_events_test;
```

```text
event_id  device_id  current_status  temp_celsius  tags_array
--------  ---------  --------------  ------------  --------------------------
1         Sensor-A   active          22.50         ["lab","ground_floor"]
2         Sensor-B   warning         24.10         NULL
3         Sensor-A   active          21.80         ["lab"]
4         Sensor-C   offline         NULL          []
```

## Checking if a path exists

`JSON_PATH_EXISTS` returns 1 if the given path is present in the document, and 0 if it is not. This checks for the key itself — a key that exists with a `null` value still returns 1.

```sql
SELECT
    event_id,
    device_id,
    JSON_PATH_EXISTS(payload, '$.error_code') AS has_error_code
FROM device_events_test;
```

```text
event_id  device_id  has_error_code
--------  ---------  --------------
1         Sensor-A   0
2         Sensor-B   1
3         Sensor-A   0
4         Sensor-C   0
```

We can also embed it in a `CHECK` constraint on the table definition. This enforces that a required field is always present on insert, which is more reliable than leaving it to the application layer. Any insert that does not include a `status` key in the payload will fail at write time with a constraint violation.

```sql
CREATE TABLE device_events_strict (
    event_id   INT IDENTITY(1,1) PRIMARY KEY,
    device_id  VARCHAR(20),
    event_time DATETIME2 DEFAULT SYSDATETIME(),
    payload    JSON NOT NULL
        CHECK (JSON_PATH_EXISTS(payload, '$.status') = 1)
);


INSERT INTO device_events_strict (device_id, payload) VALUES
('Sensor-D', N'{"temperature": 26.6, "humidity": 49, "tags": ["lab"]}'),
('Sensor-E', N'{"temperature": 23.1, "humidity": 47, "status": "warning"}')
```

we get the following error:

```text
The INSERT statement conflicted with the CHECK constraint "CK__device_ev__paylo__55F4C372". The conflict occurred in database "testdb", table "dbo.device_events_strict", column 'payload'.


The statement has been terminated.
```

## Searching for a value inside JSON

`JSON_CONTAINS` checks whether a given SQL value exists at a specific path inside a document and returns 1 if found, 0 if not, and `NULL` if any argument is null or the path does not exist.

```
JSON_CONTAINS(target_expression, search_value [, path] [, search_mode])
```

One important difference from `JSON_VALUE`: the comparison uses the SQL type of the search value against the stored JSON type. Passing an `INT` compares it as a number rather than as the string `"22"`, so we do not need to cast.

When the path points to an array, we must add the wildcard `[*]` to the path expression. Without it the function will not descend into the individual array elements.

```sql
-- find all events where 'lab' appears in the tags array
SELECT
    event_id,
    device_id,
    JSON_QUERY(payload, '$.tags') AS tags_array
FROM device_events_test
WHERE JSON_CONTAINS(payload, 'lab', '$.tags[*]') = 1;
```

```text
event_id  device_id  tags_array
--------  ---------  ---------
1         Sensor-A   ["lab","ground floor"]
3         Sensor-A   ["lab"]
```

The optional fourth argument `search_mode = 1` switches string comparisons to `LIKE` semantics so we can use wildcard patterns.

```sql
-- find all events with a status that starts with 'act'
SELECT 
    event_id,
    device_id,
    JSON_VALUE(payload, '$.status') AS current_status
FROM device_events_test
WHERE JSON_CONTAINS(payload, 'off%', '$.status', 1) = 1;
```

```text
event_id  device_id  current_status
--------  ---------  --------------
4         Sensor-C   offline
```

## Building JSON from rows

We can do this by using the following two functions:
1- `JSON_OBJECTAGG` that builds a key/value object from two columns.
2- `JSON_ARRAYAGG` that builds an array from one column.

```sql
WITH parsed_source AS (
    SELECT
        device_id,
        CAST(event_id AS VARCHAR(10)) AS event_id_str,
        CAST(JSON_VALUE(payload, '$.temperature') AS DECIMAL(5,2)) AS temp_celsius,
        JSON_VALUE(payload, '$.status') AS current_status
    FROM device_events_test
)
SELECT
    device_id,
    JSON_OBJECTAGG(event_id_str : temp_celsius) AS temperature_by_event,
    JSON_ARRAYAGG(current_status)               AS status_history
FROM parsed_source
GROUP BY
    device_id;
```

```text
device_id  temperature_by_event     status_history
---------  -----------------------  ---------------------
Sensor-A   {"1":22.50,"3":21.80}    ["active","active"]
Sensor-B   {"2":24.10}              ["warning"]
Sensor-C   {"4":null}               ["offline"]
```

## Updating a scalar value

`JSON_MODIFY` changes one property and returns the updated document. The engine reads the full document, builds a new one with the change applied, and writes the whole thing back to the column. For small documents this is not a concern, but for large documents updated frequently it is doing more work than necessary.

```sql
UPDATE device_events_test
SET payload = JSON_MODIFY(payload, '$.status', 'maintenance')
WHERE device_id = 'Sensor-C';

SELECT
    event_id,
    payload
FROM device_events_test
WHERE device_id = 'Sensor-C';
```

```text
event_id  payload
--------  -------------------------------------------------------------
4         {"temperature":null,"humidity":null,"status":"maintenance",...
```

## The .modify() method

In SQL Server 2025 we can use the `.modify()` method on native `JSON` columns. It is the preferred way to update a scalar property when the column is the native type, because the engine can operate on the stored binary representation directly without rewriting the entire document.

The conditions for a true in-place update are: 
for strings, the new value must be no longer than the current one.
for numbers, it must fall within the same type range. 

When those conditions are not met the engine falls back to a full rewrite, but the syntax and the intent are the same either way.


```sql
-- 'active' is shorter than 'maintenance', so this is a true in-place update
UPDATE device_events_test
SET payload.modify('$.status', 'active')
WHERE device_id = 'Sensor-C';

SELECT
    event_id,
    device_id,
    JSON_VALUE(payload, '$.status') AS current_status
FROM device_events_test
WHERE device_id = 'Sensor-C';
```

```text
event_id  device_id  current_status
--------  ---------  --------------
4         Sensor-C   active
```

The `.modify()` method does not support the `append` keyword. For adding a value to an array, `JSON_MODIFY` with `append` is the right approach.

## How to append a value to a JSON array?

Modifying a scalar property like `status` is easy, but arrays require special handling. If we attempt to add a new value to an array property (like `tags`) by passing a standard path to `JSON_MODIFY`, it will overwrite the entire array container with our new string value.

To tell the database engine that we want to insert a value into an array without erasing it, we must explicitly prepend the keyword `append` to the path argument.
For example, in the following query we want to add the value `"calibration"` to the `tags` array for Sensor-A rows.

```sql
UPDATE device_events_test
SET payload = JSON_MODIFY(payload, 'append $.tags', 'calibration')
WHERE device_id = 'Sensor-A';

SELECT
    event_id,
    JSON_QUERY(payload, '$.tags') AS updated_tags
FROM device_events_test
WHERE device_id = 'Sensor-A';
```

```text
event_id  updated_tags
--------  ----------------------------------------
1         ["lab","ground_floor","calibration"]
3         ["lab","calibration"]
```


## Cleanup

Run this code if you executed the queries above.

```sql
DROP TABLE IF EXISTS device_events_test;
DROP TABLE IF EXISTS device_events_strict;
```
