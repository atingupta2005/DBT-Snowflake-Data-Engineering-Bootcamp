### 0.1 Create Database & Schemas

```sql
CREATE DATABASE IF NOT EXISTS OLIST;
```

```sql
CREATE SCHEMA IF NOT EXISTS OLIST.RAW;
CREATE SCHEMA IF NOT EXISTS OLIST.ANALYTICS;
```

---

### 0.2 Create Role

```sql
CREATE ROLE IF NOT EXISTS OLIST_ROLE;
```

---

### 0.3 Grant Permissions (Full Access)

```sql
GRANT USAGE ON DATABASE OLIST TO ROLE OLIST_ROLE;
```

```sql
GRANT USAGE ON ALL SCHEMAS IN DATABASE OLIST TO ROLE OLIST_ROLE;
```

```sql
GRANT ALL PRIVILEGES
ON ALL SCHEMAS IN DATABASE OLIST
TO ROLE OLIST_ROLE;
```

```sql
GRANT ALL PRIVILEGES
ON ALL TABLES IN DATABASE OLIST
TO ROLE OLIST_ROLE;
```

```sql
GRANT ALL PRIVILEGES
ON ALL VIEWS IN DATABASE OLIST
TO ROLE OLIST_ROLE;
```

Future-proof grants:

```sql
GRANT ALL PRIVILEGES
ON FUTURE TABLES IN DATABASE OLIST
TO ROLE OLIST_ROLE;
```

```sql
GRANT ALL PRIVILEGES
ON FUTURE VIEWS IN DATABASE OLIST
TO ROLE OLIST_ROLE;
```

---

### 0.4 Create User

```sql
CREATE USER IF NOT EXISTS OLIST_USER
PASSWORD = 'StrongPassword123'
DEFAULT_ROLE = OLIST_ROLE
DEFAULT_WAREHOUSE = COMPUTE_WH;
```

---


### 0.5 Assign Warehouse Access
```sql
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE DBT_ROLE;
```

### 0.6 Assign Role to User

```sql
GRANT ROLE OLIST_ROLE TO USER OLIST_USER;
```



---

## 2. Create dbt Project

```bash
source ~/.venv/bin/activate
cd ~
dbt init olist_dbt
cd olist_dbt
ls
rm -rf models/example
nano dbt_project.yml
```

```yaml
name: 'olist_dbt'
version: '1.0'
config-version: 2

profile: 'olist_dbt'

models:
  olist_dbt:
    +materialized: table
```

---

## 3. Verify Connection

```bash
dbt debug
```

---

## 4. Materialization Layout

```bash
nano dbt_project.yml
```

```yaml
models:
  olist_dbt:
    staging:
      +materialized: view
    intermediate:
      +materialized: table
    marts:
      +materialized: table
```

---

## 5. Staging Models

```bash
mkdir -p models/staging
nano models/staging/stg_customers.sql
```

```sql
SELECT
    customer_id,
    customer_unique_id,
    customer_city,
    customer_state
FROM olist.raw.customers
```

```bash
dbt run --select stg_customers
```

```sql
SELECT COUNT(*) FROM olist.analytics.stg_customers;
```

---

```bash
nano models/staging/stg_orders.sql
```

```sql
SELECT
    order_id,
    customer_id,
    order_status,
    order_purchase_timestamp
FROM olist.raw.orders
```

```bash
dbt run --select stg_orders
```

---

## 6. `dbt run` vs `dbt run --select`

```bash
dbt run
```

Runs all models in DAG order.

```bash
dbt run --select stg_orders
```

Runs only selected model.

---

## 7. Intermediate Model

```bash
mkdir -p models/intermediate
nano models/intermediate/int_customer_orders.sql
```

```sql
WITH customers AS (
    SELECT
        customer_id,
        customer_city,
        customer_state
    FROM {{ ref('stg_customers') }}
),
orders AS (
    SELECT
        order_id,
        customer_id,
        order_status
    FROM {{ ref('stg_orders') }}
)
SELECT
    o.order_id,
    c.customer_city,
    c.customer_state,
    o.order_status
FROM orders o
JOIN customers c
  ON o.customer_id = c.customer_id
```

```bash
dbt run --select int_customer_orders
```

---

## 8. Mart Model

```bash
mkdir -p models/marts
nano models/marts/fct_orders_by_state.sql
```

```sql
SELECT
    customer_state,
    COUNT(DISTINCT order_id) AS total_orders
FROM {{ ref('int_customer_orders') }}
WHERE order_status != 'canceled'
GROUP BY customer_state
```

```bash
dbt run --select fct_orders_by_state
```

---

## 9. Packages

```bash
nano packages.yml
```

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
```

```bash
dbt deps
ls dbt_packages
```

---

## 10. Utility Commands

```bash
dbt compile
```

```bash
dbt list
```

```bash
dbt list --select marts
```

```bash
dbt clean
```

```bash
dbt --help
```

---

## 11. DAG & Compiled SQL

```bash
ls target
```

```bash
nano target/compiled/olist_dbt/models/marts/fct_orders_by_state.sql
```

---

## 12. Snowflake Validation

```sql
SELECT
    table_name,
    created_on
FROM olist.information_schema.tables
WHERE table_schema = 'ANALYTICS';
```

---

## 13. Final Runs

```bash
dbt run
```

```bash
dbt build
```

```sql
SELECT COUNT(*) FROM olist.analytics.fct_orders_by_state;
```

---


# 14. Model Selection with `dbt run --select`

---

## 14.1 Single Model

```bash
dbt run --select stg_orders
```

Runs only:

* `stg_orders`

---

## 14.2 Multiple Models

```bash
dbt run --select stg_customers stg_orders
```

Runs:

* `stg_customers`
* `stg_orders`

---

## 14.3 Folder-Based Selection

```bash
dbt run --select staging
```

Runs all models under:

* `models/staging`

---

```bash
dbt run --select marts
```

Runs all models under:

* `models/marts`

---

## 14.4 Upstream Dependencies (`+`)

```bash
dbt run --select +int_customer_orders
```

Runs:

* `stg_customers`
* `stg_orders`
* `int_customer_orders`

---

## 14.5 Downstream Dependents (`+`)

```bash
dbt run --select int_customer_orders+
```

Runs:

* `int_customer_orders`
* all downstream models (marts)

---

## 14.6 Full Lineage (Upstream + Downstream)

```bash
dbt run --select +int_customer_orders+
```

Runs:

* all parents
* the model itself
* all children

---

