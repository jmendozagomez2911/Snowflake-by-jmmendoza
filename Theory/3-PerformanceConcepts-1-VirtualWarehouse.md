# Section 4: Performance Concepts: Virtual Warehouses

This section explores **performance** in Snowflake, focusing on **virtual warehouses**—the compute layer of Snowflake’s architecture. We’ll see:

- Virtual warehouse basics (start/stop, states, properties)
- Sizing and billing
- Resource monitors to control credit usage
- Multi-cluster warehouses (scale-out)
- Query Acceleration Service (QAS)

---

## 1. Introduction to Virtual Warehouses

- **Definition**: A *virtual warehouse* is a *named abstraction* for a cluster of compute nodes (e.g., AWS EC2 instances).
- **Key Role**:
    - Performs the *execution* of SELECT, DML (INSERT, UPDATE, DELETE), and data loading operations.
    - Maintains a **local SSD cache** for faster query performance.
- **Create & Destroy**: Instantly managed via SQL or UI. You **do not** manage the underlying servers directly.
- **Independent**: You can spin up as many warehouses as needed (e.g., separate for ingestion vs. reporting).

### 1.1 Key Warehouse Properties

1. **Size** (T-shirt sizing: XS, S, M, L, etc.)
2. **State** (started, suspended, or resizing)
3. **Auto-Suspend / Auto-Resume** for cost optimization
4. **Scaling** (multi-cluster or Query Acceleration Service)

---

## 2. Warehouse States & Properties

### 2.1 States

- **Started**: Warehouse is actively running and accruing credits.
- **Suspended**: Warehouse is shut down, *not* accruing credits; any data cache is lost on suspend.
- **Resizing**: The warehouse is **scaling up or down** — that is, changing its size (e.g., `SMALL` → `MEDIUM` or `LARGE` → `SMALL`).
  - This is also called **vertical scaling**.
  - **Running queries continue uninterrupted** using the original size.
  - The new size applies only to **queued and future queries** after the resize completes.
  - Resizing is always a **manual operation** and affects a **single cluster** — it is **not the same** as scaling out with multiple clusters.
  - This mechanism ensures **zero downtime** and a smooth transition between resource levels.

### 2.2 Commands

```sql
-- Suspend
ALTER WAREHOUSE my_wh SUSPEND;

-- Resume
ALTER WAREHOUSE my_wh RESUME;
```

### 2.3 Automated Suspend/Resume

- **AUTO_SUSPEND**: After X seconds of no activity, warehouse suspends automatically.
    - Default is `600` seconds (10 mins).
    - Minimum value: `60` seconds (due to 1-minute billing granularity).
    - ⚠️ Setting it too low can cause frequent suspends/resumes, adding latency and slight cost overhead.
- **AUTO_RESUME**: If `true`, any new query against a suspended warehouse *wakes* it automatically.
    - Ideal for BI tools, ad-hoc users, or background services that don't need manual control.
    - Prevents query failures due to inactive compute.
- **INITIALLY_SUSPENDED**: If set to `TRUE`, the warehouse is **created in a suspended state**.
    - Useful for dev/test environments or scheduled workloads.
    - Often combined with `AUTO_RESUME = TRUE` so that it activates **only when needed**.
    - Helps **save credits** by avoiding startup on creation.
---

## 3. Warehouse Sizing & Billing

### 3.1 T-Shirt Sizing (XS → 6X)

- **Sizes**: XS, S, M, L, XL, 2X, 3X, 4X, 5X, 6X.
- Each size ~doubles CPU/memory resources.
- Larger size generally → faster queries on large/complex workloads, but higher cost.

### 3.2 Cost Model

- **What is a Credit?**  
  A **credit** is Snowflake’s unit for measuring compute usage.  
  You are charged credits based on **how long a virtual warehouse is running**,  
  regardless of how many queries you run or how complex they are.

- **Credits per Hour**  
  Each warehouse size consumes a fixed number of credits per hour:
  - **XS = 1**, **S = 2**, **M = 4**, **L = 8**, etc.
  - Billing is **per second**, with a **60-second minimum** each time the warehouse starts or resumes.

- **Dollar Cost**  
  The cost in dollars is calculated as: `credits_used * cost_per_credit`  
  The price per credit depends on your Snowflake edition and cloud provider — typically between **$2 and $4 per credit**.

- **Key Point**  
  You pay for **runtime**, not query volume.
  - A warehouse running for 15 minutes (even with no queries) costs the same as if it ran one heavy query in that time.
  - ⏱️ **Time = cost** — use **auto-suspend** and **auto-resume** to control usage efficiently.


### 3.3 Resizing Implications

- **Scaling Up**:
    - Increases performance for large or complex queries but also costs more credits/hour.
    - Only applies to *new/queued* queries; running queries keep their initial resources.
- **Scaling Down**:
    - Reduces cost but flushes any cache in the removed nodes.
    - Could degrade performance on next queries.

---

## 4. Resource Monitors

- **Definition**: *Standalone objects* that set credit consumption thresholds over a time interval (daily, weekly, monthly, etc.).
- **Purpose**: Prevent runaway costs by suspending or alerting when a threshold is reached.
- **Usage**:
    - Created by `ACCOUNTADMIN` (or assigned roles).
    - Can be assigned to one or more warehouses or at the account level.

### 4.1 Basic Properties

1. **Credit Quota**: e.g. `100` credits.
2. **Frequency**: `DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY`, or `NEVER` for resets.
3. **Start Timestamp**: Control when monitoring begins.
4. **Trigger**: Condition (percentage or absolute consumption) + Action (NOTIFY, SUSPEND, SUSPEND IMMEDIATE).

```sql
CREATE RESOURCE MONITOR my_rm
WITH CREDIT_QUOTA = 100
FREQUENCY = MONTHLY
TRIGGERS
ON 80 PERCENT DO NOTIFY
ON 100 PERCENT DO SUSPEND;
```

```sql
-- Apply to account or a specific warehouse:
ALTER ACCOUNT SET RESOURCE_MONITOR = my_rm;
ALTER WAREHOUSE my_wh SET RESOURCE_MONITOR = my_rm;
```

---

## 5. Multi-Cluster Warehouses (Scale-Out)

A **multi-cluster warehouse** allows Snowflake to launch multiple *identical compute clusters* under the same warehouse name to handle **high concurrency** (i.e. many users or queries at the same time).

> 🧠 Important: Each **query runs in a single cluster**. Snowflake does **not** split one query across multiple clusters like Spark does.


### 5.1 Configuration

- **MIN_CLUSTER_COUNT** and **MAX_CLUSTER_COUNT**

  These parameters define how many **compute clusters** Snowflake is allowed to run **simultaneously** under a single multi-cluster warehouse:

  - **MIN_CLUSTER_COUNT**: The **minimum number of clusters** that will always be active when the warehouse is running.
    - Snowflake **never runs fewer than this** (even if the load is low).
    - Acts like the "base capacity" of the warehouse.

  - **MAX_CLUSTER_COUNT**: The **maximum number of clusters** that Snowflake can launch to handle spikes in query load.
    - Snowflake **never exceeds this limit**, even during heavy concurrency.

  Together, these control the **scale-out range** of your multi-cluster warehouse.

---

#### ⚙️ Modes of Operation

- **Maximized Mode** (`MIN_CLUSTER_COUNT == MAX_CLUSTER_COUNT`)
  - A fixed number of clusters always run — scaling is **disabled**.
  - Example: `MIN = 3`, `MAX = 3` → 3 clusters run all the time.
  - Good for workloads with **predictable high concurrency** and consistent performance needs.

- **Auto-Scale Mode** (`MIN_CLUSTER_COUNT < MAX_CLUSTER_COUNT`)
  - Snowflake can **dynamically scale out** by adding clusters when queries are queued.
  - Also scales **back in** by removing clusters during idle periods (based on scaling policy).
  - Example: `MIN = 1`, `MAX = 5` → Snowflake runs at least 1 cluster and scales up to 5 as needed.
  - Ideal for **variable workloads**, such as interactive dashboards, user traffic peaks, or batch jobs.


### 5.2 Scaling Policy

1. **Standard**
  - Spins up a new cluster **immediately** when queries are queued.
  - Monitors usage every minute and scales down after ~2–3 minutes of low activity.
  - Reduces wait time for users, but may increase credit consumption.

2. **Economy**
  - More cost-aware: only adds a new cluster if high load is expected to persist for ~6 minutes.
  - Waits ~5–6 minutes before scaling down during light usage.
  - Lower credit usage, but may allow short-lived queues to build up.


### 5.3 Billing for Multi-Cluster

- **Total cost = credits used per cluster × number of active clusters × time**
- Example: 3 clusters running `MEDIUM` size = `3 × 4 credits/hour = 12 credits/hour`
- Auto-scaling ensures clusters only run **when needed**, saving costs during light usage.


### 5.4 Concurrency Properties

- **MAX_CONCURRENCY_LEVEL**: Max number of concurrent SQL statements per cluster. Default = 8.
- **STATEMENT_QUEUED_TIMEOUT_IN_SECONDS**: Max time a query can sit in the queue before it is automatically cancelled.
- **STATEMENT_TIMEOUT_IN_SECONDS**: Max time a **running** query is allowed to execute before being aborted.


> 💡 **Real concurrency happens when multiple queries are submitted at the same time from different sessions**.  
> For example, if you run one query per tab in the Snowflake Web UI, each tab uses a separate session — so queries may run in parallel and cause queuing if the warehouse is fully loaded.  
> In contrast, multiple queries separated by semicolons (`;`) in a single query editor tab are executed sequentially and do **not** cause concurrency or queuing.


### ✅ True/False Review

-   **"A virtual warehouse can only be scaled up manually."** → ✅ **True**
  -   Snowflake does **not** auto-increase warehouse size (`WAREHOUSE_SIZE`).
  -   It can **scale out automatically** using multiple clusters, but **scaling up** is always a **manual operation**.


⚠️ **Common confusion:**

-   ❌ **Scaling up** = increasing size (e.g. `SMALL` → `LARGE`) → **manual only**
-   ✅ **Scaling out** = adding clusters (multi-cluster) → **can be automatic**
 
## 6. Query Acceleration Service (QAS)

- **Definition**: A *serverless* feature that *dynamically* adds compute power to *complex, parallelizable* queries.
- **Behavior**:
    - If part of a query can be parallelized, Snowflake may spin up ephemeral compute to assist.
    - Once the query completes, the extra compute is relinquished.
    - Only certain queries qualify (large data scans, parallelizable segments).

### 6.1 Enabling QAS

```sql
CREATE WAREHOUSE my_wh
  WAREHOUSE_SIZE = 'LARGE'
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 3
  ENABLE_QUERY_ACCELERATION = TRUE;
```

- **`ENABLE_QUERY_ACCELERATION = TRUE`** toggles the feature on.
- **`QUERY_ACCELERATION_MAX_SCALE_FACTOR`** sets the upper bound multiplier relative to warehouse size. (Default = 8, Range = 0–100).
    - E.g., if your warehouse is 8 credits/hour and you set scale factor = 3, the *maximum* extra cost is up to 24 credits/hour *while queries are accelerating*.

### 6.2 Cost Model

- **Serverless** usage → billed *per second* of offloaded compute.
- If the service isn’t actively accelerating queries, you don’t pay for QAS.
- Potential to significantly reduce query times for sporadic complex queries without permanently sizing up the warehouse.

### 6.3 Determining Eligibility

- **`ESTIMATE_QUERY_ACCELERATION(<query_id>)`** returns JSON with:
    - `ELIGIBLE` or `INELIGIBLE` status
    - Potential performance improvements across various scale factors
- Use **Query Profile** in UI to see actual “Scan selected for acceleration” metrics.

---

## 7. Conclusion & Next Steps

**Section 4** introduced how Snowflake’s virtual warehouses drive **performance and cost**:

1. **States & Automation**: Start/suspend warehouses on demand to save credits.
2. **Sizing**: Match the T-shirt size to your workload.
3. **Resource Monitors**: Cap or monitor credit usage.
4. **Multi-Cluster**: Scale-out for concurrency.
5. **Query Acceleration**: Dynamically add serverless compute to large/complex queries.

Together, these features provide powerful levers for balancing **performance**, **throughput**, and **cost** in Snowflake. In practice, you’ll combine them with query optimization techniques (caching, partition pruning, etc.) to efficiently handle a wide range of data workloads.

**End of Documentation**