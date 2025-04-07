# Section 3: Account Access & Security

This section covers **20-25%** of the SnowPro Core Certification exam and focuses on *who* can do *what* in Snowflake, *how* they authenticate, and *how* data is protected. Key themes include:

- **Access Control** (RBAC, DAC)
- **Roles & Privileges**
- **User Authentication** (passwords, MFA, federated auth, etc.)
- **Network Policies**
- **Data Encryption**
- **Column & Row-Level Security**
- **Secure Views**
- **Account Usage & the Information Schema**

---

## 1. Introduction to Account Access & Security

Snowflake’s security model combines:
1. **Role-Based Access Control (RBAC)** – Privileges (SELECT, CREATE, etc.) are assigned to roles rather than directly to users. Users get permissions by having roles granted to them.
2. **Discretionary Access Control (DAC)** – Every object (table, view, etc.) has an *owner* (the role that created it). That owner can grant or revoke privileges on the object to other roles.

Exam-wise, you’ll see many questions about creating/managing roles, understanding ownership, granting/revoking privileges, using multi-factor authentication, controlling network access, and so forth.

---

## 2. Access Control Overview

- **Scope**: Defines *who* can perform *what operations* on *which objects*.
- **Two Major Schemes**
    1. **Role-Based Access Control (RBAC)**:
        - Privileges are assigned to *roles*.
        - Roles are then granted to *users*.
        - A user can have multiple roles and can switch the *active* role in a session.
    2. **Discretionary Access Control (DAC)**:
        - Each object in Snowflake has exactly one *owner role*.
        - The owner can grant/revoke privileges on that object to other roles.

- **Object Hierarchy & Ownership**
    - Securable objects include databases, warehouses, schemas, tables, stages, etc.
    - The role that creates an object is its owner and has *all privileges* on that object.
    - Owners can transfer ownership to another role or manage privileges with `GRANT` / `REVOKE`.
    - A global privilege called **MANAGE GRANTS** also lets a role grant/revoke privileges on objects it does *not* own.

---

### 🧠 Account vs User

To clarify Snowflake's structure:

- An **account** is a complete Snowflake environment — it contains its own warehouses, databases, roles, and users. Each organization may have multiple accounts (e.g. dev, test, prod).
- A **user** is an identity that connects to a specific account. Users exist **inside an account**, and each account manages its own user list.

You filter by `user_name` in views like `ACCOUNT_USAGE.QUERY_HISTORY` to see what each individual user did **within that account**. You are not querying users from other accounts — only from the one you're logged into.

Example:
```sql
SELECT user_name, query_text, execution_status
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD(DAY, -1, CURRENT_TIMESTAMP());
```


## 3. Roles

- **Definition**: A *role* is a *securable object* that can hold privileges on other objects. Roles can also be granted to other roles, forming a hierarchy.

  - **System-Defined Roles**
      1. **ORGADMIN**: Top-level *organization* role (manages multiple Snowflake accounts, usage across org, etc.).
      2. **ACCOUNTADMIN**: Top-level *account* role; can do nearly everything in the account (billing, all object access).
      3. **SECURITYADMIN**: Manages **access control and privileges** across the account. Can create, alter, and drop both **users and roles**, and **grant or revoke roles and object-level privileges**. Also responsible for managing **role hierarchies**, RBAC enforcement, and **enterprise-level security policies**.  
         → Use this role when enforcing security, assigning roles to other roles, or managing access to objects like tables, warehouses, and databases.
      4. **USERADMIN**: Manages **user accounts** and **custom roles**. Can create, alter, and drop users, reset passwords, assign default roles and warehouses (only within its role hierarchy), and manage login policies.  
       → Use this role for onboarding users, managing identity, and setting up default access, but **not** for granting access to objects or managing roles outside its scope.
      5. **SYSADMIN**: Manages *objects* (warehouses, databases, schemas, etc.).
      6. **PUBLIC**: A pseudo-role granted to *all* users by default (objects owned by PUBLIC are open to everyone).

      ### USERADMIN vs SECURITYADMIN
    
        | Feature / Capability                              | `USERADMIN`          | `SECURITYADMIN`        |
        |---------------------------------------------------|----------------------|------------------------|
        | Create/Alter/Drop Users                           | ✅ Yes               | ✅ Yes                |
        | Create/Alter/Drop Roles                           | ✅ Yes               | ✅ Yes                |
        | Assign Roles to Users                             | ✅ (limited scope)   | ✅ (full scope)       |
        | Grant/Revoke Privileges on Objects                | ❌ No                | ✅ Yes                |
        | Manage Role Hierarchy (grant role to role)        | ❌ No                | ✅ Yes                |
        | Manage Login Policies / Password Resets           | ✅ Yes               | ✅ Yes                |
        | Used for Role-Based Access Control (RBAC)         | ❌ No                | ✅ Yes


- **Custom Roles**
    - Created to implement the principle of *least privilege*.
    - Typically aligned with business roles, e.g., `DATA_ANALYST`, `ETL_ENGINEER`, etc.
    - Created by roles that have the `CREATE ROLE` privilege, such as `USERADMIN`, `SECURITYADMIN`, or `ACCOUNTADMIN`.
    - However, **creating a custom role is not the same as assigning it**:
        - `USERADMIN` can create and assign roles, but **only within its own hierarchy**.
        - To assign a custom role to a **system-defined role** (e.g., `SYSADMIN`) or across role trees, a higher-level role such as `SECURITYADMIN` is required.
    - Example:
      ```sql
      CREATE ROLE data_engineer;
      GRANT ROLE data_engineer TO ROLE sysadmin; -- requires SECURITYADMIN
      ```
    - This separation ensures that role delegation respects Snowflake’s role hierarchy and prevents unauthorized privilege elevation.

- **Role Hierarchy**
    - If Role A is granted to Role B, then Role B inherits all privileges of Role A.
    - System roles already form a hierarchy: `ACCOUNTADMIN` > `SECURITYADMIN` > `USERADMIN` > `SYSADMIN` > `PUBLIC`.

---

## 4. Privileges

- **Definition**: Security privileges determine what actions can be performed on a securable object (e.g., `SELECT` on a table, `USAGE` on a warehouse).
- **Categories**:
    1. **Global Privileges**: e.g. `MANAGE GRANTS` (let you manage privileges on any object).
    2. **Account-Level Privileges**: e.g. `CREATE DATABASE`, `MONITOR USAGE`.
    3. **Schema-Level Privileges**: e.g. `USAGE` on a schema, `CREATE TABLE` in a schema.
    4. **Object-Level Privileges**: e.g. `SELECT`, `INSERT`, `MODIFY`, `TRUNCATE` on tables or views.

- **Granting & Revoking**
    - Use `GRANT <privilege> ON <object> TO ROLE <role_name>`.
    - Use `REVOKE <privilege> ON <object> FROM ROLE <role_name>`.
    - Only owners or roles with `MANAGE GRANTS` can grant/revoke.

- **Future Grants**
    - Automate privileges on objects *yet to be created*, e.g.
      ```sql
      GRANT SELECT ON FUTURE TABLES IN SCHEMA my_db.my_schema TO ROLE my_role;
      ```
    - Not supported with certain features (e.g. data sharing, replication, masking/row policies).

---

## 5. User Authentication

- **Username/Password** (default)
    - A user is created with `CREATE USER ... PASSWORD = '...'`.
    - Password must be ≥8 characters, at least 1 digit, 1 uppercase, 1 lowercase.
    - Optionally set `MUST_CHANGE_PASSWORD = TRUE` so user must reset on first login.
    - If no default role is specified, user inherits the `PUBLIC` role on login.

---

## 6. Multi-Factor Authentication (MFA)

- **Overview**
    - Adds a second factor to the username+password login.
    - Snowflake uses Duo Security behind the scenes.
    - Enabled per *user* (via UI > Preferences > MFA).

- **Login Flow**
    1. User enters Snowflake username/password.
    2. Receives a DUO prompt (mobile push, phone call, or code).
    3. Approves or enters passcode to finalize login.

- **Admin Options**
    - **Bypass MFA**: Temporarily allow login without MFA (for X minutes).
    - **Disable MFA**: Cancels user’s enrollment; user must re-enroll if needed later.
    - **Allow Client MFA Caching**: Reduces repeated prompts for short successive logins.

---

## 7. Federated Authentication

- **Definition**: Delegates user authentication to an external **Identity Provider (IdP)** supporting **SAML 2.0**.
- **Supported IdPs**: Okta, ADFS, or custom SAML.
- **Single Sign-On (SSO)**: Allows users to log into multiple services (including Snowflake) with one set of IdP credentials.

- **Configuration**
    - Requires setting account properties like `SAML_IDENTITY_PROVIDER` (certificate, SSO URL, etc.).
    - An “SSO” button can appear on Snowflake’s login page, redirecting users to the IdP.

---

## 8. Key Pair Authentication, OAuth & SCIM

- **Key Pair Auth**
    - Replaces username/password with a *public/private key* pair.
    - Public key stored in Snowflake (`ALTER USER` to set `RSA_PUBLIC_KEY`).
    - Private key stays on client side, used by connectors (Python, Node, etc.) to authenticate.
    - Supports rotation with two concurrent keys.

- **OAuth 2.0**
    - **Delegated Authorization**: e.g. let a BI tool like Tableau access Snowflake without exposing user credentials.
    - Supported in two ways: Snowflake-hosted OAuth and External OAuth.

- **SCIM** (System for Cross-domain Identity Management)
    - Automates user provisioning and role assignments.
    - An IdP can push group updates into Snowflake via the SCIM API.

---

## 9. Network Policies

- **Purpose**: Restrict which IPs can connect to Snowflake, adding a layer of security beyond user credentials.
- **Creation**
  ```sql
  CREATE NETWORK POLICY my_policy
    ALLOWED_IP_LIST = ('192.168.1.0/24')
    BLOCKED_IP_LIST = ('192.168.1.10');
  ```
- **Application**
    - **Account-Level** (`ALTER ACCOUNT SET NETWORK_POLICY = my_policy;`)
    - **User-Level** (`ALTER USER username SET NETWORK_POLICY = my_policy;`)
    - Only one policy can be active per account/user.
    - If both are set, the user-level policy takes precedence.

---

## 10. Data Encryption

Snowflake handles **encryption at rest** and **in transit** automatically:

1. **Encryption at Rest**
    - **AES-256** on all table data, internal stages, result caches.
    - Uses a **hierarchical key model** (file keys < table keys < account key < Snowflake root key).
    - **Key Rotation** every 30 days, with an optional **periodic re-keying** that re-encrypts data (Enterprise edition feature).

2. **Encryption in Transit**
    - **TLS 1.2** over HTTPS for client connections (UI, JDBC, ODBC, etc.).
    - Ensures end-to-end encryption between the user and Snowflake.

3. **Tri-Secret Secure**
    - An optional approach to combine **Snowflake-managed key** + **customer-managed key**.
    - Customer can revoke their part of the key if needed.
    - Requires additional support and is only on Business Critical / higher editions.

---

## 11. Column-Level Security: Dynamic Data Masking

- **Masking Policies**
    - Schema-level objects specifying how to *dynamically mask* certain column data at query time.
    - Example policy:
      ```sql
      CREATE MASKING POLICY email_mask AS (
        val string
      ) RETURNS string ->
      CASE
        WHEN current_role() = 'SUPPORT' THEN val
        ELSE '****'
      END;
      ```
    - Attached to a column with:
      ```sql
      ALTER TABLE my_table
        MODIFY COLUMN email SET MASKING POLICY email_mask;
      ```
- **Behavior**
    - If a user’s role matches policy conditions, they see the *real* value; otherwise, they see a masked value.
    - Data is *not* modified on disk—masking is *dynamic* at query time.

- **External Tokenization**
    - Store data in *tokenized* form, then *detokenize* with an external function if authorized by the masking policy.
    - Ensures even Snowflake cannot read the raw data unless the user is authorized and the external system decrypts.

---

## 12. Row-Level Security (Row Access Policies)

- **Definition**
    - Filter out entire rows from query results based on policy conditions.
    - Like masking policies but return *Boolean* conditions (whether to include the row).

- **Row Access Policy** Example
  ```sql
  CREATE ROW ACCESS POLICY row_access_1 AS
  (val int) RETURNS boolean ->
  CASE
    WHEN current_role() = 'ADMIN' THEN true
    ELSE false
  END;
  ```
    - Applied via:
      ```sql
      ALTER TABLE my_table
        MODIFY COLUMN account_id SET ROW ACCESS POLICY row_access_1;
      ```
- **Behavior**
    - If the condition is false, the user never “sees” those rows in query results.
    - Evaluated *before* any column masking policies.

---

## 13. Secure Views
- **Overview**
    - A **secure view** in Snowflake hides the underlying table metadata and logic from users who lack the necessary privileges.
    - It also disables certain query optimizations that could unintentionally expose restricted data.
    - You can designate both **standard views** and **materialized views** as secure by using the `SECURE` keyword during creation. 
      This allows you to protect both logical views and precomputed results.

    - Created with:
      ```sql
      CREATE SECURE VIEW my_view AS ...
      ```

      ### 🔍 **Example**
      Let’s say you have a sensitive table:
      
      ```sql
      CREATE TABLE employee_data (
        id INT,
        name STRING,
        salary NUMBER,
        department STRING
      );
      ```
      
      You want to expose only names and departments to users without revealing salary info or the base table:
      
      ```sql
      CREATE SECURE VIEW public_view AS
      SELECT name, department
      FROM employee_data
      WHERE department = 'Finance';
      ```
      
      Now you can:
      ```sql
      GRANT SELECT ON VIEW public_view TO ROLE analyst;
      ```
      
      The `analyst` role can query the view but:
      - ❌ Cannot see or access the `employee_data` table
        - ❌ Cannot view the view’s SQL definition
        - ✅ Can only see `name` and `department` for Finance employees


- **Effects**
    1. Definition & underlying object details are hidden from unauthorized users (even `SHOW VIEW` or `GET_DDL`).
    2. Prevents “push-down” optimizations that could accidentally reveal data that the user can’t normally access.

- **Tradeoff**
    - Potentially slower performance due to fewer optimizations.
    - Recommended only if you *must* mask table details from certain roles/users.

---

## 14. Account Usage & Information Schema

Snowflake provides built-in metadata views to support **auditing**, **monitoring**, and **usage analysis** across your account.

These are primarily exposed through two systems:
- The **SNOWFLAKE shared database**
- The **INFORMATION_SCHEMA** views available per database

### 14.1 SNOWFLAKE Database & ACCOUNT_USAGE

#### **SNOWFLAKE Database**
- A shared, **read-only database** (`SNOWFLAKE`) that includes multiple internal schemas for metadata tracking and billing analysis:

    - `ACCOUNT_USAGE` — long-term, **account-level** metadata and activity tracking.
        - Includes details such as executed queries, login history, warehouse usage, table storage, user activity, and more.
        - Allows filtering by `user_name`, making it ideal for **auditing**, **security reviews**, and **technical troubleshooting**.
        - Data retention: up to **1 year**. Latency: around **2 hours**.

    - `ORGANIZATION_USAGE` — **organization-wide** usage and billing metrics.
        - Aggregates credit usage and storage consumption across **all Snowflake accounts** in the same organization.
        - Does **not** expose per-user details — only at the account level (`account_name`, `account_locator`).
        - Useful for **financial reporting**, **cost optimization**, and comparing environments (e.g., dev vs prod).

    - Other internal schemas:
        - `REPLICATION` — metadata on **replication groups** and **failover configurations** (for high availability and DR).
        - `DATA_SHARING` — tracks **inbound/outbound shares**, reader accounts, and usage of shared data.

- Typically, only the `ACCOUNTADMIN` role has access by default (but access can be granted to others).


#### **ACCOUNT_USAGE Schema**
- Contains views that provide **historical metadata and usage data**, such as:
    - Query history
    - Credit consumption
    - Role and user activity
    - Object DDL and access patterns
- Data is retained for **up to 1 year**
- Updates have a **~2-hour latency**
- Dropped objects are retained with a `DELETED` timestamp

    #### 🔍 Real-world usage difference

    - Want to know **who** used **how many credits** on **what warehouse** yesterday?  
    → Use `ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY`

    - Want to know **how many total credits** your **QA account** spent last month?  
    → Use `ORGANIZATION_USAGE.CREDITS_USED_BY_ACCOUNT_DAILY`

### 14.2 INFORMATION_SCHEMA

- **Definition**
    - A read-only schema **automatically created in every database**.
    - Contains views and table functions for metadata about objects in that **database** (some views provide info at the account level).

- **Usage & Retention**
    - Real-time or near real-time metadata, but with **shorter history** (typically from 7 days to 6 months, depending on the object type).
    - Does **not** track dropped objects.

- **Scope**
    - `INFORMATION_SCHEMA` exists at the **database level**, not inside individual schemas.
    - It provides metadata for **all schemas and all objects** within that specific database (tables, views, procedures, privileges, etc.).

---

## 15. Conclusion & Next Steps

In **Section 3: Account Access & Security**, you learned about:

- **RBAC & DAC** (roles, privileges, object ownership)
- **User Authentication** (password policy, MFA, federated auth, key pairs, OAuth, SCIM)
- **Network Policies** (allow/block IP ranges)
- **Data Encryption** (hierarchical key model, tri-secret secure)
- **Column- and Row-Level Security** (masking, row access policies)
- **Secure Views** (hiding underlying table details)
- **Account Usage & Information Schema** (monitor usage, queries, objects)

This completes the fundamental security concepts essential for the SnowPro Core exam. In practice, combine these concepts to enforce *least privilege*, secure data in transit and at rest, and maintain strong governance over your Snowflake environment.
