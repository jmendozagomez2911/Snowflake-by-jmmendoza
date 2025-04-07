# Section 8: Storage, Data Protection & Data Sharing

This final section focuses on **how Snowflake stores data** and provides **data protection** features (Time Travel, Fail-safe) alongside **secure data sharing** mechanisms (secure shares, data marketplace, data exchange). We’ll also see how zero-copy cloning, replication, and storage billing fit into the bigger picture.

---

## 1. Introduction

In Snowflake, data is stored in **cloud provider blob storage** (AWS S3, Azure Blob, or GCS) but managed by Snowflake. When you load data into tables, Snowflake reorganizes it into a **columnar format** and divides it into **micro-partitions**. Key topics in this section:

1. **Micro-partitions**: Sub-file structures storing chunks of table data.
2. **Time Travel & Fail-safe**: Recover or revert table data over a certain timeframe.
3. **Cloning & Replication**: Zero-copy clones inside an account; replication across accounts in an organization.
4. **Secure Data Sharing**: Provide read-only data access across Snowflake accounts (and to non-Snowflake users via reader accounts).
5. **Data Marketplace** & **Data Exchange**: Mechanisms to share or monetize datasets.
6. **Storage Billing**: How data storage usage and protection copies (Time Travel, Fail-safe) factor into monthly costs.

---

## 2. Storage Layer Overview

### 2.1 Conceptual Structure

- **Centralized Data Storage**
    - All table data is stored in cloud provider blob storage, fully managed by Snowflake.
    - Tables and schemas are just logical abstractions referencing these columnar files.
- **Micro-partitions**: Automatic partitioning based on the “natural order” of data as loaded; typically 50–500 MB each.
- **Stages**: Temporary holding areas for loading/unloading (internal or external).

### 2.2 Data Lifecycle

1. **Active Data** (current table state, up-to-date micro-partitions).
2. **Time Travel** (historical micro-partitions for a defined retention period).
3. **Fail-safe** (non-configurable 7-day retention for permanent tables after Time Travel expires).

### 2.3 Storage Billing

- **Flat monthly rate** per TB, computed via average daily usage over the month.
- **Cost** depends on region, cloud provider, and whether you are On-Demand or Capacity.

---

## 3. Micro-partitions

When loading data (e.g. via `COPY INTO <table>`), Snowflake automatically:

1. **Transforms** the file into columnar format, compressing & encrypting.
2. **Divides** data into micro-partitions (50–500 MB each).
3. **Stores** each partition in immutable files.

#### Metadata & Partition Pruning

- **Metadata Store** tracks min/max values per column for each micro-partition (plus distinct counts, etc.).
- **Pruning**: Queries with a filter or predicate can skip partitions not relevant to the query, reducing scans and boosting performance.

---

## 4. Time Travel & Fail-safe

### 4.1 Time Travel

A *configurable period* (0–1 days on Standard, up to 90 days on Enterprise) during which historical data can be viewed or restored:

- **UNDROP**: Restore a dropped table, schema, or database if within retention.
- **AT | BEFORE**: Query a table “as of” a past time/statement ID.
- **Cloning in the Past**: Create a zero-copy clone at a prior time point using `CLONE ... AT (...)`. 
This creates a **logical snapshot** of the table (or database/schema) as it existed at that time **without copying physical data**. Snowflake simply references the same micro-partitions as the original object, making it fast and space-efficient. If the clone is later modified, only the changed parts are stored separately (copy-on-write).

**Data Retention** is set via `DATA_RETENTION_TIME_IN_DAYS` at the account, database, schema, or table level. Child settings override parent.


### 4.2 Fail-safe

  Fail-safe is Snowflake’s **last-resort recovery mechanism**, designed to recover data after Time Travel has expired — typically in the event of a **catastrophic failure** or accidental data loss that cannot be resolved by the user.
- **7-day non-configurable** retention period after Time Travel expires (applies to **permanent tables only**).
- Recovery requires contacting **Snowflake Support** — users cannot access Fail-safe data directly.
- Data in **Fail-safe** still **incurs storage costs**.
- Fail-safe begins **only after Time Travel ends**. For example, with **Enterprise Edition** and 90-day Time Travel, Fail-safe covers **days 91 to 97** after a table was dropped or modified.
- After Fail-safe expires, the data is **permanently unrecoverable**.
---

## 5. Cloning

**Zero-copy cloning** creates a *logical* copy of an object (table, schema, or database) **without duplicating data**.

- **Syntax**:
  ```sql
  CREATE [ OR REPLACE ] TABLE new_table_name CLONE source_table_name;
  ```
- **Zero-copy**: The cloned table initially references the same micro-partitions as the source.
- **Changes** in the clone or source do not affect each other—new/updated partitions are created as needed.
- **Time Travel** can be combined, e.g.
  ```sql
  CREATE TABLE new_table_name CLONE source_table_name AT (TIMESTAMP => '...') 
  ```
- **Cloning** is recursive for databases/schemas, but some objects (e.g., external tables) are never cloned, and privileges are not copied unless `COPY GRANTS` is specified.

---

## 6. Replication

**Replication** is used for cross-account data redundancy and disaster recovery:

- **Primary** database in one account, **secondary** read-only replica in another.
- **Incremental refresh** pushes changes from primary to secondary.
- **Requires** an Organization feature + `ENABLE ACCOUNT DATABASE REPLICATION` to be set in both source & target accounts.
- **Billing** includes data transfer across regions and compute for the replication operation.

---

## 7. Secure Data Sharing

Allows *read-only* data sharing between Snowflake accounts with **no data copying**:

1. **Data Provider** sets up a **share** object:
    - `CREATE SHARE myshare;`
    - Grants objects (tables/views, etc. – must be secure for views or UDFs).
    - Adds consumer accounts: `ALTER SHARE myshare ADD ACCOUNTS = ( 'account_name' );`

   > 🧠 Think of the `SHARE` as a container or intermediate layer. The provider "uploads" (exposes) specific objects into this container, and the consumer later "retrieves" them by creating a virtual database from the share.

2. **Data Consumer** creates a *database* from that share:
    - `CREATE DATABASE mydb FROM SHARE provider_acct.myshare;`

   > ❗ This does not create a copy of the data. The consumer creates a virtual, read-only database that points to the shared data residing in the provider's account.

3. **Consumption**: The data is queryable in the consumer’s environment.
4. **Limitations**:
    - ✅ The shared data is **read-only** for the consumer.
    - ❌ **Only one database can be created per share** in the consumer account.
        - For example:
          ```sql
          CREATE DATABASE shared_sales FROM SHARE provider_acct.sales_share; -- ✅ works
          CREATE DATABASE sales_copy FROM SHARE provider_acct.sales_share;   -- ❌ error
          ```
          A given `SHARE` can only be used to create one database in a consumer account. If another is needed, the provider must create a new share.
    - ❌ **No Time Travel**: Consumers cannot use `AT` or `BEFORE` clauses on shared data.
    - ❌ **No modification**: Consumers cannot insert, update, or delete data from shared objects.
    - ❌ **No re-sharing**: A consumer cannot create a new share from the shared data.
    - ⚠️ **Cross-region or cross-cloud sharing requires replication**: Data must be replicated to the target region before it can be shared across clouds or regions.



### 7.1 Reader Accounts

- For consumers *without* a Snowflake account.
- Managed by the data provider; the provider pays for usage.
- Very limited feature set (read-only, no data load, etc.).
- Limit of 20 by default (can be raised by Support).

---

## 8. Data Marketplace & Data Exchange

Snowflake also provides **Data Marketplace** for publicly listing datasets and a **Data Exchange** feature for private exchanges within a specific group/organization:

1. **Data Marketplace**: Publicly accessible listings (some free, some monetized).
2. **Data Exchange**: Private, invite-only environment (like a private marketplace).
    - Requires contacting Snowflake Support to set it up.
    - Similar to marketplace but restricted to selected accounts (e.g., internal departments, trusted vendors).

---

## 9. Conclusion

**Section 8** covers crucial final pieces of the Snowflake puzzle:

- **Storage** model (micro-partitions, metadata, no user-managed partitioning).
- **Data Protection** via **Time Travel** and **Fail-safe** (restore, undrop, historical queries).
- **Cloning** (zero-copy, quick test or dev environments).
- **Replication** for multi-account DR scenarios.
- **Secure Data Sharing** for read-only data sharing with other Snowflake accounts or *reader accounts*.
- **Data Marketplace** & **Data Exchange**: Options for broader or private data distribution/monetization.

This completes the core Snowflake capabilities for data lifecycle management and collaboration, rounding out the knowledge needed for the SnowPro Core Certification.