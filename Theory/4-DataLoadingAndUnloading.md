

# Section 6: Data Loading & Unloading

This section covers about **10-15%** of the exam and focuses on how to **move data** into and out of Snowflake. You’ll learn about **stages**, the **COPY** commands, **Snowpipe** for continuous ingestion, data **unloading**, and handling **semi-structured** data (JSON, Avro, Parquet, etc.).

---

## 1. Introduction

Snowflake supports multiple ways to load data:
1. **Bulk Loading** (manual execution of `COPY INTO <table>`),
2. **Continuous Ingestion** with **Snowpipe**,
3. **Simple Insert/GUI** (not scalable but useful for quick demos).

Just as you can load data into Snowflake using `COPY INTO <table>`,  
you can also **unload data** from tables into files using the `COPY INTO <location>` command.

In both directions, Snowflake uses **stages** — special file exchange areas — to handle file-based data movement.  
Stages can be:

- **Internal stages**: File storage inside Snowflake, managed by Snowflake.  
  These are used when you upload files from your local machine using the `PUT` command.
- **External stages**: Your own cloud storage (S3, Azure, or GCS) registered with Snowflake using `CREATE STAGE`.  
  You don’t upload files into these — Snowflake reads them directly from the cloud location.

Even though the term “stage” sounds temporary, it just describes the role of this location —  
as an **intermediate file landing zone** before data enters or exits a Snowflake table.  
The data itself may live there temporarily or permanently, especially in external stages.

We also delve into **semi-structured data** (JSON, Avro, Parquet, etc.) loading, unloading, and query methods in Snowflake, which uses specialized data types (`VARIANT`, `OBJECT`, `ARRAY`) and extended SQL functions.

---

## 2. Stages

Stages are *temporary holding areas* for data files (raw, uncompressed CSV/JSON, or compressed, etc.).  
Files in a stage can then be **loaded** into a table or **unloaded** from a table.

### 2.1 Internal vs. External Stages

1. **Internal Stages** (fully managed by Snowflake — no cloud configuration needed):

    - **User Stage**:  
      A personal stage automatically created for each user.  
      ➤ Used for quick testing or uploading small local files with `PUT`.
        - Referenced by `@~`
        - Accessible only to the user
        - Cannot be altered/dropped

    - **Table Stage**:  
      A stage automatically created for every table.  
      ➤ Mainly used for unloading table data into files.
        - Referenced by `@%<table_name>`
        - Accessible to users with privileges on the table
        - Cannot be altered/dropped

    - **Named Internal Stage**:  
      A reusable stage object created with `CREATE STAGE`.  
      ➤ Used in shared pipelines, repeatable jobs, and production workflows.
        - Referenced by `@<stage_name>`
        - Can define file format, encryption, directory options
        - Securable with `GRANT`/`REVOKE`

2. **External Named Stages**:

    - These are backed by user-managed cloud storage (S3, GCS, Azure Blob).
    - Created with `CREATE STAGE`, referencing a cloud URL and security integration or credentials.
    - Snowflake reads/writes files but does **not** store them.
    - Still referenced in SQL as `@<external_stage_name>`.

---


### 📋 Summary Table

| Stage Type           | Reference         | Managed By   | Load | Unload | Typical Use                            | Securable |
|----------------------|-------------------|--------------|------|--------|----------------------------------------|-----------|
| User Stage           | `@~`              | Snowflake    | ✅   | ❌     | Personal file uploads and testing      | ❌        |
| Table Stage          | `@%table`         | Snowflake    | ❌   | ✅     | Unloading from specific tables         | ❌        |
| Named Internal Stage | `@my_stage`       | Snowflake    | ✅   | ✅     | Reusable/shared pipelines              | ✅        |
| External Stage       | `@my_s3_stage`    | Your cloud   | ✅   | ✅     | Production cloud-based data movement   | ✅        |

### 2.2 Useful Commands for Stages

- **PUT**: Upload local files → internal stage (cannot be run from a worksheet, must be via SnowSQL or the CLI).
    - E.g. `PUT file:///my_data.csv @mystage` (Unix) or `PUT file://C:\my_data.csv @mystage` (Windows).
- **LIST**: Lists the contents of a stage.
    - E.g. `LIST @mystage`
- **SELECT** (from stage files):
    - E.g. `SELECT $1, $2 FROM @mystage`
- **REMOVE**: Removes files from stage.
    - E.g. `REMOVE @mystage folder_name/`

---

## 3. Bulk Loading: `COPY INTO <table>`

### 3.1 Overview

- **Mechanics**: `COPY INTO <table>` from a stage moves data into Snowflake’s columnar format.
- **Supported Formats**: CSV/delimited, JSON, Avro, ORC, Parquet, XML.
- **Virtual Warehouse**: Bulk load uses a *user-managed* warehouse for compute.
- **Load History**: Stored in target table metadata for **64 days**, prevents duplicate file loads (checks file name & contents).

### 3.2 Syntax Examples

```sql
COPY INTO my_table
  FROM @my_int_stage
  [ FILES = ( 'file1.csv', 'file2.csv' ) ]
  [ PATTERN = '.*2023.*' ]
  [ <LOAD_TRANSFORMATIONS> ]
  [ FILE_FORMAT = ( TYPE = CSV ... ) ]
  [ COPY_OPTIONS... ];
```

- **Load transformations**: Reordering columns, casting data types, skipping columns, etc. E.g.
  ```sql
  COPY INTO my_table
    FROM (
      SELECT $1::int AS id,
             $2::timestamp AS created_at,
             $3::string AS notes
      FROM @my_stage
    );
  ```
- **On-Error Handling** (`ON_ERROR`): Controls how Snowflake handles bad or invalid rows when loading data with `COPY INTO`.
    - `ABORT_STATEMENT` (default): ❌ Abort the entire load on the **first error**. Best when data must be 100% clean.
    - `CONTINUE`: ✅ Keep loading and **ignore bad rows**. Errors are logged in the load history.
    - `SKIP_FILE_n%`: ⏭️ Skip the **entire file** if more than `n%` of rows are invalid.
    - `SKIP_FILE_<num>`: ⏭️ Skip the file if more than `<num>` bad rows are found.

  🔧 **Example**:
  ```sql
  COPY INTO my_table
  FROM @my_stage/data.csv
  FILE_FORMAT = (TYPE = 'CSV')
  ON_ERROR = CONTINUE;

### 3.3 Validation

Snowflake provides tools to **check data quality before or after** loading using `COPY INTO`.

- **Validation Mode** (in `COPY INTO`)\
  Performs a **dry run** without actually loading data (no data is actually loaded). This helps you identify errors in files before inserting anything into the table.

  ```sql
  COPY INTO my_table
  FROM @my_stage
  VALIDATION_MODE = 'RETURN_n_rows' | 'RETURN_ERRORS' | 'RETURN_ALL_ERRORS';
  ```

  ✅ Options:

    - `'RETURN_n_rows'`: Returns a preview of the first `n` rows.
    - `'RETURN_ERRORS'`: Returns the **first** parsing or formatting error.
    - `'RETURN_ALL_ERRORS'`: Returns **all errors** found during parsing.

  🔧 **Example**:

  ```sql
  COPY INTO my_table
  FROM @my_stage
  VALIDATION_MODE = 'RETURN_ERRORS';
  ```

  This performs a dry validation and reports the first issue encountered.

- **VALIDATE Table Function**\
  Used **after** a load to inspect any errors related to a specific job ID.

  ```sql
  SELECT * FROM TABLE(VALIDATE(job_id => '...'));
  ```

  ➤ Helpful for reviewing what went wrong in a previous load, including skipped rows.


### 3.4 Best Practices

- **Split large files** (~100–250 MB compressed) for parallel ingestion.
- **Avoid tiny files** if possible; combine them before loading.
- **Pre-sort data** to improve micro-partition clustering.
- **Separate warehouse** for loading to avoid resource contention with other queries.

---

## 4. File Formats

- **Purpose**: Tells Snowflake how to parse or produce files (delimiters, compression, etc.).
- **Creating**:
  ```sql
  CREATE FILE FORMAT my_csv_format
    TYPE = CSV
    FIELD_DELIMITER = '|'
    SKIP_HEADER = 1
    COMPRESSION = AUTO;
  ```
- **Usage**:
    - *Attach to stage* or specify in `COPY INTO` or both.
    - If both stage and `COPY INTO` define `FILE_FORMAT`, the `COPY INTO` setting takes precedence.

---

## 5. Continuous Ingestion: Snowpipe

- **Definition**: *Serverless* feature that automatically loads data *within ~1 minute* of file arrival in a stage.

- **Implementing**:
    1. **Create a Pipe** referencing a `COPY INTO <table>` statement.
        - This `COPY INTO` defines the exact logic Snowpipe will run automatically for each new file.
        - Example:
          ```sql
          CREATE OR REPLACE PIPE my_pipe AS
          COPY INTO my_table
          FROM @my_stage
          FILE_FORMAT = (TYPE = 'CSV');
          ```
        - ❗ The `COPY INTO` is not replaced — it's automated by the Pipe when a new file is detected.

    2. **Two modes**:
        - `AUTO_INGEST = TRUE`: Use cloud event notifications (S3 → SQS → Snowflake) for new files in external stages.
        - `AUTO_INGEST = FALSE`: You must call Snowflake’s REST API (`insertFiles`) yourself to inform about new files. Works with internal or external stage.

- **Load History**: 14 days in pipe metadata.

- **Cost**:
    1. Serverless compute credits for each loaded file.
    2. “Utilization Cost” of 0.06 credits / 1000 file notifications.

### 5.1 Snowpipe vs. Bulk Loading

| Feature             | **Bulk Loading**                            | **Snowpipe**                                         |
|---------------------|---------------------------------------------|------------------------------------------------------|
| **Compute**         | User-managed virtual warehouse              | Snowflake-managed serverless compute                 |
| **Trigger**         | Manual `COPY INTO <table>` execution        | Automatic via cloud events or REST API (`insertFiles`) |
| **Latency**         | On-demand (user triggered)                  | Near real-time (~1 min after file arrival)           |
| **Load History**    | 64 days (visible via table metadata)        | 14 days (tracked in pipe metadata)                   |
| **Scalability**     | Depends on warehouse size & concurrency     | Automatically scales to ingest many small files      |
| **Billing**         | Credits for active warehouse time           | Credits per file loaded + notification cost (0.06 credits / 1000 files) |

---

## 6. Data Unloading

- **Command**: `COPY INTO <location>` from [ table | query ].
- **Destinations**:
    1. **Internal Stage** (then `GET` to download),
    2. **External Stage** or *directly* to e.g. `s3://...`,
    3. Must use `GET` for final local copy from an internal stage.
- **Parallel Output**: By default, output is split across multiple files, determined by the warehouse size.
- **Formats**: CSV, TSV, JSON, Parquet for data.
- **Syntax**:
  ```sql
  COPY INTO @mystage/unload_prefix_
  FROM (SELECT col1, col2 FROM my_table)
  FILE_FORMAT = (TYPE = CSV)
  OVERWRITE = TRUE
  SINGLE = FALSE
  MAX_FILE_SIZE = 16777216
  PARTITION BY (date_col)
  INCLUDE_QUERY_ID = TRUE;
  ```
- **GET Command**:
    - E.g. `GET @mystage/unload_prefix_ file:///local/path/`
    - Only works for internal stages (external downloaded via cloud tools).
    - `PARALLEL` and `PATTERN` options available.

---

## 7. Semi-Structured Data Overview

Snowflake **natively** supports semi-structured formats: **JSON**, **Avro**, **ORC**, **Parquet**, **XML**.
- **Extended SQL**: Additional data types `VARIANT`, `ARRAY`, `OBJECT` store hierarchical data.
- **Equivalent to**: `VARIANT` can hold any nested structure.
- **Max size**: 16 MB *per row* for `VARIANT` if storing entire objects.

---

## 8. Loading & Unloading Semi-Structured Data

- **Same `COPY INTO <table>`** approach as structured, but specifying a **semi-structured** `FILE_FORMAT`.
- **Three Approaches**:
    1. **ELT** style: Load entire file into one `VARIANT` column.
    2. **ETL** style: Transform/pick specific elements into separate columns.
    3. **Semi-Automated**: Using `infer_schema` or `MATCH_BY_COLUMN_NAME`.

- **Unloading**:
    - `COPY INTO <location>` supports **JSON** & **Parquet** for semi-structured outputs.

---

## 9. Accessing Semi-Structured Data

### 9.1 Semi-Structured Data Types

Snowflake supports semi-structured data using flexible, schema-less data types. These types allow storage and querying of nested or dynamic data (e.g. JSON, Avro, XML, Parquet).

1. **VARIANT**: A universal container type that can hold any semi-structured content — arrays, objects, strings, numbers, etc.
    - Acts as the default type when you ingest JSON.
    - Can store deeply nested data of mixed types.
    - Example:
      ```json
      {
        "id": 1,
        "name": "Ana",
        "skills": ["SQL", "Python"]
      }
      ```

2. **OBJECT**: Represents key-value pairs (similar to a JSON object).
    - Each value inside can be a scalar, array, or even another object.
    - Example: `{"type": "employee", "active": true}`

3. **ARRAY**: An ordered list of values, accessed by position.
    - Example: `["SQL", "Python", "Spark"]`

---

#### 🆚 VARIANT vs OBJECT

| Feature         | `VARIANT`                                              | `OBJECT`                                             |
|----------------|---------------------------------------------------------|------------------------------------------------------|
| Definition      | Generic container for any semi-structured data         | Key-value pair map (like a JSON object)              |
| Content         | Can store objects, arrays, strings, numbers, etc.      | Only stores key-value pairs                          |
| Use Case        | General ingestion of unknown JSON structure            | When you know you're working with object data only   |
| Access          | `variant_col:key` or `variant_col['key']`              | Same: `object_col:key` or `object_col['key']`        |
| Nesting         | Can contain objects and arrays                         | Can be stored inside a `VARIANT`, but not vice versa |

---

### 9.2 Dot & Bracket Notation

Used to access fields inside VARIANT/OBJECT/ARRAY values.

- **Dot Notation**:
  ```sql
  SELECT data:employee.name
  FROM my_table;
  ```
  ➤ Use when the keys are valid identifiers (no spaces, symbols, etc.).

- **Bracket Notation**:
  ```sql
  SELECT data['employee']['name']
  FROM my_table;
  ```
  ➤ Use when keys contain special characters or are dynamically provided.

- **Casting Values**:
  ```sql
  SELECT data:birthdate::DATE
  FROM my_table;
  ```
  ➤ Required to convert values from VARIANT to usable SQL types (e.g. DATE, NUMBER).

---

### 9.3 Flattening

When your data includes **nested arrays**, use the `FLATTEN()` table function to convert elements into individual rows.

- Returns metadata like:
    - `KEY` (object key),
    - `INDEX` (array position),
    - `VALUE` (actual content),
    - `PATH` (hierarchy within the document)

- Commonly paired with a **LATERAL JOIN**, so that each exploded element can be joined back to its parent row.

#### 🔧 Example: Exploding an array

Suppose you have the following data in a VARIANT column `data`:
```json
{
  "id": 1,
  "name": "Ana",
  "skills": ["SQL", "Python"]
}
```

Query using flatten:
```sql
SELECT t.id, t.data:name AS name, f.value AS skill
FROM my_table t,
     LATERAL FLATTEN(input => t.data:skills) f;
```

**Before Flattening (1 row):**

| id | data                                                              |
|----|-------------------------------------------------------------------|
| 1  | { "id": 1, "name": "Ana", "skills": ["SQL", "Python"] }           |

**After Flattening (2 rows):**

| id | name | skill   |
|----|------|---------|
| 1  | Ana  | "SQL"   |
| 1  | Ana  | "Python"|

This allows you to normalize and query nested arrays as individual records.


---

## 10. Conclusion & Next Steps

In **Section 6**, you learned how to:

1. **Load** data using:
    - *Simple Insert/GUI* (small scale),
    - *Bulk loading* (`COPY INTO <table>` with a user warehouse),
    - **Snowpipe** (serverless, near real-time ingestion).
2. **Unload** data using `COPY INTO <location>` to stages or direct cloud storage, plus the **GET** command for local retrieval.
3. **Handle Semi-Structured** data:
    - Data types (`VARIANT`, `OBJECT`, `ARRAY`),
    - File formats (JSON, Avro, Parquet, etc.),
    - Dot/bracket notation, flatten function, casting.

This completes the fundamental concepts of **Data Loading & Unloading** in Snowflake. Next, you’ll see how this integrates with other platform features (like transformations, tasks & streams, or BI tools).