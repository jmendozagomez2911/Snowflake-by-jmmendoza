# Section 7: Data Transformations

This section delves deeper into **structured** and **unstructured** data transformations, providing an overview of Snowflake’s function families (scaler, aggregate, window, table, and system functions), sampling methods, unstructured file handling with directory tables, and the file support REST API.

---

## 1. Introduction

Snowflake offers a broad range of **built-in functions** for transformations:
- Scalar functions (one value in → one value out)
- Aggregate functions (many rows → one output)
- Window functions (subset/partition of rows → aggregated output per partition)
- Table functions (one input row → multiple output rows)
- System functions (metadata, system info, or control operations)

Additionally, Snowflake provides special functions and features for:
- **Estimation** (approximate calculations for large data sets)
- **Table Sampling** (quick random subsets)
- **Unstructured** data transformations (file URLs, directory tables, REST APIs)

---
## 2. Summary of Snowflake Functions

Snowflake categorizes its *built-in* functions as follows:

1. **Scalar Functions**
    - Return one value per call.
    - Subcategories include math, string, date/time, JSON & semi-structured, data generation, etc.
    - Common for transforming data at the row level.
    - Examples:
        - `UPPER('hello')` → `'HELLO'`
        - `DATEADD(DAY, 7, CURRENT_DATE)` → adds 7 days to today’s date
        - `UUID_STRING()` or `UNIQUEID()` → produce a unique identifier

2. **Aggregate Functions**
    - Return a single output from multiple input rows.
    - Common examples: `SUM()`, `AVG()`, `COUNT()`, `MAX()`, `MIN()`, etc.
    - Also includes statistical and collection-based aggregates like `ARRAY_AGG()` or `CORR()`.
    - Often used in `GROUP BY` queries to summarize data.
    - Examples:
        - `COUNT(*)` → total number of rows
        - `AVG(salary)` → average salary from multiple rows
        - `ARRAY_AGG(department)` → combines multiple values into an array

3. **Window Functions**
    - Also known as *analytic* functions (e.g., `RANK()`, `MAX() OVER (...)`).
    - Operate over a window (subset) of rows defined by `PARTITION BY` and ordered by `ORDER BY`.
    - Useful for calculations like running totals, row numbers, or comparisons within a group.
    - Examples:
        - `RANK() OVER (PARTITION BY department ORDER BY salary DESC)` → rank employees within each department
        - `SUM(sales) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` → running total of sales

4. **Table Functions**
    - Return a table (set of rows) instead of a single value.
    - Can return 0, 1, or many rows per input row.
    - Include both system-provided and user-defined functions.
    - Examples:
        - `GENERATOR(ROWCOUNT => 10)` – produces synthetic rows for testing
        - `FLATTEN(input => data:skills)` – expands arrays into multiple rows
        - `SPLIT_TO_TABLE('A,B,C', ',')` – returns each letter as a separate row

5. **System Functions**
    - Provide system-level information or administrative actions.
    - Useful for monitoring, debugging, or controlling execution.
    - Examples:
        - `SYSTEM$PIPE_STATUS('<pipe_name>')` – returns JSON describing whether a pipe is paused, running, etc.
        - `SYSTEM$CANCEL_QUERY('<query_id>')` – cancels a currently running query
        - `SYSTEM$USER_TASK_DEPENDENTS_ENABLED()` – shows task dependency settings

> These functions are essential for transforming, analyzing, monitoring, and automating workflows in Snowflake.


---

## 3. Estimation Functions

Approximate calculations can be much faster and use less memory, especially for very large data sets, at the cost of some small error. Snowflake’s estimation functions fall into four categories:

1. **Cardinality Estimation**
   - Estimates the number of distinct values in a column using probabilistic techniques (HyperLogLog).

   - **Use `APPROX_COUNT_DISTINCT(column)`** when you need a quick, one-time estimate of unique values. This function runs immediately and returns an approximate count.
     ```sql
     SELECT APPROX_COUNT_DISTINCT(user_id) FROM users;
     ```

   - **Use `HLL(column)`** when you want to store or reuse the result later. It returns a compressed sketch object, which can be stored, combined, and later passed to `APPROX_COUNT_DISTINCT_ESTIMATE(...)`.
     ```sql
     SELECT HLL(user_id) AS user_sketch FROM users;
     -- Later:
     SELECT APPROX_COUNT_DISTINCT_ESTIMATE(user_sketch) FROM sketch_table;
     ```

   - Typical error rate: ~1.6%, but much more efficient than exact `COUNT(DISTINCT ...)`, especially on large datasets.
   - 🔁 Results are generally similar for both approaches, but using HLL allows for aggregation across distributed sources or for deferred computation.


2. **Similarity Estimation**
    - Compares the similarity between two sets of data.
    - Examples:
        - `SELECT MINHASH(ARRAY_AGG(product)) FROM orders WHERE category = 'A';`
        - `SELECT APPROX_SIMILARITY(sigA, sigB);` where sigA and sigB are minhash results
        - Returns value from 0 to 1 (1 = identical, 0 = no overlap)

3. **Frequency Estimation**
    - Identifies most frequent values in a column.
    - Examples:
        - `APPROX_TOP_K(product_id, 5)` → approx top 5 products by frequency
        - `APPROX_TOP_K(product_id, 5, 100)` → same, with memory cap

4. **Percentile Estimation**
    - Approximates percentiles using T-Digest algorithm.
    - Examples:
        - `APPROX_PERCENTILE(salary, 0.5)` → approximate median
        - `APPROX_PERCENTILE(salary, 0.95)` → approximate 95th percentile
    - Great for performance with large numeric datasets where sorting would be expensive

---

## 4. Table Sampling

Snowflake supports **random sampling** of rows from a table.

### Syntax:

```sql
SELECT *
FROM my_table
SAMPLE [ ROW | BLOCK ] (<percent>)
[ SEED <integer> ];
```

### Sampling Types

- **Fraction-based sampling (percentage):**
    - `ROW` / `BERNOULLI`: Each row has a `p%` chance of being selected.
    - `BLOCK` / `SYSTEM`: Applies the sampling probability per data block, not individual rows. Better performance on large tables but less random with small tables.
    - You can use a `SEED` to make the sample deterministic (if data doesn’t change).

  **Example:**
    ```sql
    SELECT *
    FROM sales_data
    SAMPLE ROW (10) -- selects ~10% of the rows
    SEED 99;
    ```

- **Fixed-size sampling:**
    - Request a specific number of rows, chosen randomly.

  **Example:**
    ```sql
    SELECT *
    FROM my_table
    SAMPLE (5 ROWS); -- returns exactly 5 random rows
    ```


---

## 5. Unstructured File Functions

Snowflake allows you to store and manage *unstructured* files (images, PDFs, videos, etc.) in stages—either **internal** or **external**. There are three main functions to generate URLs for staged files:

1. **`BUILD_SCOPED_FILE_URL(stage_name, file_path)`**
    - Returns an *encoded*, short-lived URL (24-hour validity).
    - Only usable by the user who generated it.
    - Can be embedded in a secure view or UDF to share access.
    - ✅ **Use when** you want file access scoped to the current user session, such as in secure dashboards or UDFs with row-level access.

2. **`BUILD_STAGE_FILE_URL(stage_name, file_path)`**
    - Returns a permanent “file URL” with no expiry.
    - Requires *READ* privilege on internal stage or *USAGE* on external stage.
    - ✅ **Use when** you need a consistent, shareable link that doesn’t expire, and you're managing security at the application level.

3. **`GET_PRE_SIGNED_URL(stage_name, file_path, expiration_in_seconds)`**
    - Generates a time-limited pre-signed URL (e.g., 600s) for direct, unauthenticated access.
    - Also requires read/usage privilege on the stage.
    - ✅ **Use when** you need to give **temporary public access** to a file, such as for external systems, APIs, or time-limited downloads.

**Use Case**: You can store images, PDFs, etc. in a stage, and create a table that references them by path. Then generate a file URL that can be used in UIs or BI tools to fetch that file.

---

## 6. Directory Tables

When enabled on a **named stage** (internal or external), a “directory table” gives a **queryable interface** to list the stage’s files as rows:

```sql
CREATE OR REPLACE STAGE mystage
  DIRECTORY = ( ENABLE = TRUE );
```

```sql
SELECT *
FROM DIRECTORY(@mystage);
```

This returns metadata about each file in the stage.

### Columns Returned:
- `RELATIVE_PATH` – file path relative to the stage root
- `FILE_SIZE` – size in bytes
- `LAST_MODIFIED` – timestamp of last modification
- `FILE_URL` – equivalent to using `BUILD_STAGE_FILE_URL`
- `FILE_CHECKSUM` – SHA-256 hash (optional)

### Metadata Refresh:
- **External Stages**:
    - Refresh can be **automatic** (with cloud notifications)
    - Or **manual** via:
      ```sql
      ALTER STAGE mystage REFRESH;
      ```
- **Internal Stages**:
    - Only support **manual refresh**.

### 🔍 Example Query:

```sql
SELECT RELATIVE_PATH, FILE_SIZE, FILE_URL
FROM DIRECTORY(@my_internal_stage)
WHERE FILE_SIZE > 100000
ORDER BY LAST_MODIFIED DESC;
```

This lists files over 100 KB, sorted by most recently modified.


---

## 7. File Support REST API

Snowflake provides a **single** method: `GET /api/files`, letting you download a file from a stage via a *scoped* or *file* URL:

- **Scoped URL**:
    - Only the user who created the URL can download.
    - Valid for 24 hours.
- **File URL**:
    - Lasts indefinitely.
    - Role must have privileges on the stage (READ for internal, USAGE for external).

**Usage** (Python example):
```python
import requests

scoped_or_file_url = "<some URL from BUILD_SCOPED_FILE_URL or BUILD_STAGE_FILE_URL>"
headers = {
  'authorization': 'Bearer <OAuth_token>',
  'Accept': 'application/json'
}

response = requests.get(scoped_or_file_url, headers=headers)
file_contents = response.content
# e.g. save to disk
```
Combines well with custom applications or dashboards that need dynamic file access from Snowflake stages.

---

## 8. Conclusion & Next Steps

**Section 7** focuses on Snowflake’s **transformation functions** and extended handling of **unstructured** data:

1. **Function Families**: Scalar, Aggregate, Window, Table, System.
2. **Estimation Functions**: Approx distinct, top-k, similarity, percentile.
3. **Table Sampling**: Random or fixed-size subsets.
4. **Unstructured Files**:
    - Generating URLs with `BUILD_SCOPED_FILE_URL`, `BUILD_STAGE_FILE_URL`, `GET_PRE_SIGNED_URL`.
    - Directory Tables for a queryable file listing.
    - File Support REST API for direct programmatic download.

From here, you can integrate these techniques into data engineering pipelines or advanced analytics that combine structured, semi-structured, and unstructured data in Snowflake.