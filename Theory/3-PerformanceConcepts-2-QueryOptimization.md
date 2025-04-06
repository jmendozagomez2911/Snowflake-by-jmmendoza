# Section 5: Performance Concepts – Query Optimization

This section focuses on **query-level** performance in Snowflake—optimizing SQL, leveraging caching, materialized views, clustering, and specialized services like the Search Optimization Service. While Snowflake automates many tasks, writing efficient queries and managing your data layout can still have a **major** impact on performance and costs.

---

## 1. Query Performance Analysis Tools

### 1.1 Query History

- **Location**:
    - Snowflake UI (Snowsight or Classic Console) → “History”
    - Retains **14 days** of query logs by default.
- **Visibility**:
  - Users can see each other’s query text if they have either the `OPERATE` or `MONITOR` privilege on the warehouse.
    - `MONITOR`: Allows viewing query history and performance stats.
    - `OPERATE`: Includes `MONITOR` privileges plus the ability to start, stop, or resize the warehouse.
  - Users cannot view each other’s query **results**, only metadata.
- **Information**:
    - Query status (success/failure), duration, bytes scanned, compilation vs. execution time, etc.
    - Filter queries by user, duration, error, etc.

### 1.2 Query Profile

- **Definition**: A visual and interactive breakdown of an *executed* SQL query, showing how Snowflake processed it step by step.
  - It displays an **operator tree** (similar to a DAG) that shows the query plan as executed.
  - **Operator Nodes**:
    - E.g., “TABLE SCAN,” “JOIN,” “SORT,” etc.
    - Shows row counts, time spent, and relationships between operators.
- **Diagnostics**:
    - Identify heavy or “expensive” operations (lots of processing time or data spill).
    - See if queries suffer from excessive *sorting*, *joining*, or *spilling to local/remote disk*.

### 1.3 Account Usage & Information Schema

Snowflake provides two programmatic ways to analyze query history using SQL — each with different scope and use cases:

- **ACCOUNT_USAGE Views** (in the **SNOWFLAKE** database):
  - Long-term metrics (up to 1 year).
  - ~45-minute latency.
  - Best for **auditing**, **historical reporting**, and tracking activity across the entire account.
  - ✅ You are querying views such as `QUERY_HISTORY`, `LOGIN_HISTORY`, `WAREHOUSE_METERING_HISTORY`, and more — all scoped to your full account.
    - For example, `QUERY_HISTORY` includes: `query_text`, `user_name`, `warehouse_name`, `execution_status`, `start_time`, `total_elapsed_time`, `bytes_scanned`, etc.
    - Example use:
      ```sql
      SELECT query_text, user_name, total_elapsed_time
      FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
      WHERE start_time >= DATEADD(DAY, -1, CURRENT_TIMESTAMP());
      ```

- **INFORMATION_SCHEMA Table Functions** (in each database):
  - 7 days of query history, nearly real-time.
  - Best for **real-time monitoring**, **troubleshooting**, or **per-database dashboards**.
  - ✅ You are calling table functions like `QUERY_HISTORY()`, `TABLE_STORAGE_METRICS()`, and others — limited to **only that specific database**.
    - Includes similar fields: `query_text`, `execution_status`, `start_time`, etc.
    - Example use:
      ```sql
      SELECT *
      FROM TABLE(MY_DB.INFORMATION_SCHEMA.QUERY_HISTORY(DATE_RANGE_START => DATEADD(HOUR, -1, CURRENT_TIMESTAMP())));
      ```

Use these programmatic options depending on your needs:
- Use `ACCOUNT_USAGE` for **long-term, account-wide auditing** and reporting.
- Use `INFORMATION_SCHEMA` for **short-term, real-time diagnostics** and **database-specific monitoring**.

---

## 2. SQL Tuning

Snowflake’s engine handles many optimizations automatically. However, **poorly structured SQL** can still degrade performance significantly. Key considerations:

### 2.1 Order of Execution

1. **FROM / JOIN / WHERE**
2. **GROUP BY / HAVING**
3. **SELECT / DISTINCT**
4. **ORDER BY**
5. **LIMIT**

Putting filters *early* (e.g., in subqueries) helps reduce data before more expensive operations like GROUP BY and ORDER BY.

### 2.2 Joins

- **Unique Join Keys**:
    - Avoid unintentional row explosion (i.e., many-to-many joins).
    - Validate that columns used in the join are unique or carefully handle duplicates.
- **Query Profile** can show if the join operator outputs far more rows than expected.

### 2.3 ORDER BY & LIMIT

- **Sorting Large Data** can be expensive if it spills to disk.
- **LIMIT** can reduce the number of rows that must be sorted.
- Placing multiple ORDER BY clauses (e.g., in subqueries) is usually redundant unless logically required—prefer ordering once at the top level if possible.

### 2.4 GROUP BY & Cardinality

- **Grouping Low/Medium Cardinality** columns is usually fine.
- **Grouping Very High Cardinality** columns (e.g., unique IDs) can cause significant overhead:
    - May spill to local or remote storage.
    - Check the operator tree for large “AGGREGATE” times.

### 2.5 Memory Spilling

- If you see **spilling to local disk** or **spilling to remote disk** in the Query Profile, it means your warehouse ran out of memory for intermediate results.
- **Solutions**: Filter out more data (WHERE), or **increase** the warehouse size so there is more memory available.

---

## 3. Caching

Snowflake has **three** relevant “caches,” each at different layers.

### 3.1 Metadata Cache (Services Layer)

- **Definition**: Stores metadata about database objects, such as table structure, row counts, and micro-partition info.
- **Purpose**: Enables metadata-related queries to execute without using compute resources.
- **Duration**: Metadata is typically refreshed automatically, but may persist for **minutes to hours** depending on system activity.

### 3.2 Results Cache (Services Layer)

- **Definition**: Stores the final result of a query for fast reuse.
- **Purpose**: Returns results instantly if the same query is rerun and the data hasn’t changed.
- **Duration**: Valid for up to **24 hours** (can be extended to **up to 31 days** if the result is reused regularly).
- **Note**: The underlying data must remain unchanged, and the query must be **textually identical**.

### 3.3 Local Disk Cache (Virtual Warehouse)

- **Definition**: Uses the warehouse’s local SSD to store raw data files accessed during query execution.
- **Purpose**: Reduces the need to re-read data from cloud storage for repeated queries.
- **Duration**: Cache persists only **while the warehouse is running**. It is cleared when the warehouse is **suspended, resized, or dropped**.


## 4. Materialized Views

- **Definition**: *Pre-computed*, *persisted* results of a SELECT on *one table*.
- **Minimum Edition**: Requires **Enterprise Edition** or higher.

- **Background Maintenance**:
    - Uses **serverless** compute (billed at a separate rate) to keep the view’s data in sync with base table.
    - If the view is outdated, Snowflake can read base table data at query time to ensure consistent results.
- **Cost**:
    1. **Storage** of the pre-computed results.
    2. **Serverless compute** for automatic refresh (10 credits/hr + 5 credits/hr for cloud services at time of recording).
- **Limitations**:
    - **Single** base table only (no joins).
    - No `ORDER BY`, `LIMIT`, `HAVING`, `WINDOW FUNCTIONS`, or UDFs in the definition.
- **Syntax**:
  ```sql
  CREATE MATERIALIZED VIEW mv_name AS
    SELECT col1, col2 FROM base_table WHERE ...
    -- no joins, no order by, etc.
  ```

---

## 5. Clustering

### 5.1 Conceptual Basics

- **Definition**: Organizing column values within micro-partitions so that related data is stored together.
- **Purpose**: Improves query performance by enabling **micro-partition pruning** (skipping irrelevant partitions).
- **How It's Measured**:
  - `CLUSTERING_INFORMATION(<table>, <columns>)` – shows how well data is clustered based on specific columns ( returns a row with multiple clustering metrics (avg overlaps, depth, total partitions).).
  - `CLUSTERING_DEPTH('<table>', '(<columns>)')` – returns a single float number representing the average number of micro-partitions that contain the same values (i.e., overlap depth).
- **Key Metrics**:
  - **Overlapping micro-partitions** – Measures how many micro-partitions contain overlapping values in the clustering column(s).
    - This is a **table-level metric**.
    - The more micro-partitions share the same values, the less effective pruning becomes.
    - Fewer overlaps = better clustering performance.
    - Example: If `region_id = 2` appears in micro-partitions MP1, MP2, and MP4, then multiple partitions need to be scanned to find that value, reducing efficiency.
  - **Overlap depth** – Measures how many micro-partitions a **specific value** appears in.
    - This is a **value-level metric**.
    - Lower depth = better clustering and more effective pruning.
    - Example: If `customer_id = 123` appears in only 1–2 micro-partitions, queries filtering on that ID will scan less data. But if it appears in 10+ partitions, performance degrades.
  - **Summary of Differences**:
    - *Overlapping micro-partitions* shows how **spread out all values are** across partitions.
    - *Overlap depth* shows how **spread out each individual value is**.
    
- **Analogy**: Think of micro-partitions like boxes in a warehouse:
  - High overlapping micro-partitions = many boxes contain a mix of the same types of items (e.g., multiple boxes all contain bananas, apples, and oranges), making it inefficient to scan.
  - High overlap depth = a specific item (e.g., banana) is stored in many boxes — so to collect all bananas, you must open many boxes.

### 5.2 Natural Clustering

- **Definition**: The *default physical ordering* of data in Snowflake, based on how it was loaded or inserted.
- **How it works**:
  - Snowflake automatically organizes rows into micro-partitions based on load order.
  - Often sufficient for smaller or moderately sized tables.
  - For example, if you load a CSV sorted by `event_date`, the table may naturally be clustered by that column.
- **When it's enough**:
  - If your queries benefit from partition pruning and clustering metrics are acceptable, no need to define a clustering key.
  - Good for low-to-moderate size tables or append-only workloads.
- **Limitations**:
  - You have no control over clustering beyond how you insert/load the data.
  - Over time, natural clustering can degrade if data is inserted in random order.

### 5.3 Automatic Clustering

- **Definition**: A background process that keeps data organized based on a specified *clustering key* (columns or expressions).
- **When to Use**:
  - Recommended for **tables larger than 1 TB**, especially when queries frequently filter on the same columns.
  - Useful when **natural clustering is insufficient** (e.g., due to frequent DML or unsorted data loads).
  - Helps maintain query performance over time as data grows and becomes more fragmented.
- **Cost**:
  - Charged separately as a **serverless** feature.
  - Frequent data changes can trigger more re-clustering and lead to higher costs.


  #### 5.3.1 Best Practices for Automatic Clustering

  - Pick columns with *medium cardinality* frequently used in WHERE, GROUP BY, or ORDER BY.
  - Limit to ~3–4 columns max.
  - Large, frequently queried, *relatively* stable tables get the most benefit.

---

## 6. Search Optimization Service

- **Definition**: A specialized service to speed up “selective point lookup queries,” e.g. `WHERE col = 42` or `WHERE col IN (...)`.
- **Mechanism**:
    - Builds a **search access path** data structure that quickly locates specific values in the micro-partitions.
    - Another serverless feature with ongoing cost.
- **Applicable**:
    - Queries returning few rows from very large tables.
    - Data types: integer/decimal, date/time/timestamp, string/binary.
- **Costs**:
    - 10 credits/hour for compute + 5 for cloud services, plus storage overhead for the access path structure.

---

## 7. Conclusion & Next Steps

**Section 5** presented essential **query optimization** strategies and features:

1. **SQL Tuning**: Good join logic, early filters, mindful GROUP BY & ORDER BY.
2. **Caching**:
    - Results Cache for query re-use,
    - Local Disk Cache for faster repeated scans,
    - Metadata Cache for quick object info.
3. **Materialized Views**: Pre-computed results for a single table to speed up repeated complex queries.
4. **Clustering** & **Automatic Clustering**: Improves partition pruning by reordering data.
5. **Search Optimization Service**: Speeds up selective, point-lookup queries on large tables.

Combining thoughtful SQL practices, caching awareness, and advanced features (materialized views, clustering, or search optimization) can significantly improve Snowflake performance and lower costs.

**End of Documentation**