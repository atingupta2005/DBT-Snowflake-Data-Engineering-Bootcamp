# Snowflake + dbt Quick Start

## 1. What We Are Building

A minimal but complete ELT flow:

```
Sample Data → Snowflake RAW table → dbt model → ANALYTICS table
```

Key understanding:

* Snowflake stores and processes data
* dbt transforms data inside Snowflake
* dbt does NOT load data

---

## 2. Snowflake Initial Setup (Admin / Signup User)

Login to Snowflake Web UI using the **signup (admin) account**.

All steps in this section must be executed using the signup user.

---

## 3. Create Role for dbt

```sql
CREATE ROLE IF NOT EXISTS DBT_ROLE;
```

Validation:

```sql
SHOW ROLES LIKE 'DBT_ROLE';
```

---

## 4. Create Dedicated dbt User (REQUIRED)

Do NOT use the signup account for dbt.

```sql
CREATE USER IF NOT EXISTS DBT_USER
  PASSWORD = 'StrongPassword@123'
  DEFAULT_ROLE = DBT_ROLE
  DEFAULT_WAREHOUSE = COMPUTE_WH
  MUST_CHANGE_PASSWORD = FALSE;
```

Grant role:

```sql
GRANT ROLE DBT_ROLE TO USER DBT_USER;
```

Validation:

```sql
SHOW USERS LIKE 'DBT_USER';
```

---

## 5. Warehouse Access

```sql
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE DBT_ROLE;
```

Validation:

```sql
SHOW GRANTS TO ROLE DBT_ROLE;
```

---

## 6. Create Database and Schemas

```sql
USE ROLE ACCOUNTADMIN;
USE WAREHOUSE COMPUTE_WH;
CREATE DATABASE IF NOT EXISTS DBT_DEMO_DB;
USE DATABASE DBT_DEMO_DB;
CREATE SCHEMA IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS ANALYTICS;
```

Grant permissions:


```sql
GRANT ALL ON DATABASE DBT_DEMO_DB TO ROLE DBT_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE DBT_DEMO_DB TO ROLE DBT_ROLE;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE DBT_DEMO_DB TO ROLE DBT_ROLE;
```

```
GRANT ALL PRIVILEGES ON SCHEMA DBT_DEMO_DB.ANALYTICS TO ROLE DBT_ROLE;
GRANT ALL PRIVILEGES ON SCHEMA DBT_DEMO_DB.RAW TO ROLE DBT_ROLE;

-- read access to all existing raw tables
GRANT SELECT ON ALL TABLES IN SCHEMA DBT_DEMO_DB.RAW TO ROLE DBT_ROLE;

-- write/manage access on analytics tables (explicit)
GRANT INSERT, UPDATE, DELETE, TRUNCATE, SELECT ON ALL TABLES IN SCHEMA DBT_DEMO_DB.ANALYTICS TO ROLE DBT_ROLE;

-- or grant everything:
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA DBT_DEMO_DB.ANALYTICS TO ROLE DBT_ROLE;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA DBT_DEMO_DB.RAW TO ROLE DBT_ROLE;
```

Validation:

```sql
SHOW SCHEMAS IN DATABASE DBT_DEMO_DB;
```

---

## 7. Collect Snowflake Connection Details

Snowflake UI → **Admin → Accounts**

This value is used directly in `profiles.yml`.

---

| Field     | Value                   |
| --------- | ----------------------- |
| Account   | e.g. xy12345.ap-south-1 |
| User      | DBT_USER                |
| Role      | DBT_ROLE                |
| Warehouse | COMPUTE_WH              |
| Database  | DBT_DEMO_DB             |
| Schema    | ANALYTICS               |

Important notes:

* Account ID is case-sensitive

---

## 8. Create dbt Project (CentOS)

Install DBT
```
sudo dnf update -y
sudo yum update
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install "dbt-core==1.9.8" "dbt-snowflake==1.9.4"
dbt --version
```

Activate virtual environment:

Initialize dbt:

```bash
cd ~
dbt init dbt_demo
```

Choose:

* Adapter: snowflake
* Project name: dbt_demo

```bash
cd dbt_demo
```

---

## 9. Configure dbt Profile (REQUIRED FOR SNOWFLAKE CONNECTION)

This step tells **dbt how to connect to Snowflake**. Without this file, dbt cannot run.

Location of profile file (fixed for all users):

```
~/.dbt/profiles.yml
```

Edit the file:

```bash
nano ~/.dbt/profiles.yml
```

Replace the entire file content with:

```yaml
dbt_demo:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: EZUEDIM-AW05172
      user: DBT_USER
      password: StrongPassword@123
      role: DBT_ROLE
      warehouse: COMPUTE_WH
      database: DBT_DEMO_DB
      schema: ANALYTICS
      threads: 4
```

How dbt uses this:

* `dbt_demo` → must match project name created by `dbt init`
* `account` → Snowflake Account Identifier
* `user/password` → Snowflake login for dbt
* `role/warehouse/database/schema` → execution context

Checklist before saving:

* Account ID copied from Snowflake UI
* DBT_USER exists
* DBT_ROLE granted correctly
* Database and schema already created

---

## 10. Validate dbt–Snowflake Connectivity

After project creation and profile configuration, always validate connectivity.

Run:

```bash
dbt debug
```

Expected output:

```
All checks passed!
```

What `dbt debug` validates:

* profiles.yml syntax
* Snowflake authentication
* Role and warehouse access
* Database and schema access

---

## 11. Load Sample Data (RAW Schema)

```sql
USE DATABASE DBT_DEMO_DB;
USE SCHEMA RAW;

CREATE OR REPLACE TABLE customers (
  customer_id INT,
  customer_name STRING,
  country STRING,
  signup_date DATE
);

INSERT INTO customers VALUES
(1,'Amit','India','2023-01-10'),
(2,'John','USA','2023-02-15'),
(3,'Sara','UK','2023-03-01'),
(4,'Neha','India','2023-03-18');
```

Validation:

```sql
SELECT COUNT(*) FROM customers;
```

---

## 12. Create dbt Model

```bash
nano models/customers.sql
```

```sql
SELECT
  customer_id,
  customer_name,
  country,
  signup_date
FROM DBT_DEMO_DB.RAW.customers
```

---

## 13. Configure Materialization

```bash
nano dbt_project.yml
```

Ensure:

```yaml
models:
  dbt_demo:
    +materialized: table
```

---

## 14. Run dbt

```bash
dbt run
```

Expected:

```
1 of 1 OK created table ANALYTICS.customers
```

---

## 15. Validate Output

```sql
SELECT * FROM DBT_DEMO_DB.ANALYTICS.customers ORDER BY customer_id;
```

Row count must match RAW table.

---

## 16. Debugging Checklist

```bash
dbt debug
```

```bash
dbt compile
```

Check compiled SQL:

```
target/compiled/dbt_demo/models/customers.sql
```

Validation commands:

```bash
dbt --version
dbt debug
dbt compile
dbt run
```

Generated SQL validation:

```bash
ls target/compiled/dbt_demo/models/
```

---

```sql
SELECT COUNT(*) FROM DBT_DEMO_DB.RAW.customers;
SELECT COUNT(*) FROM DBT_DEMO_DB.ANALYTICS.customers;
```

