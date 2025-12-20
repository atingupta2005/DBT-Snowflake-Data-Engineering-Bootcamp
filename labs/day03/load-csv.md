# Load CSVs in Snowflake

## High-Level Flow (What You’re Doing)

1. Create a database + schema
2. Create tables with explicit columns
3. Create a file format for CSVs
4. Upload files to a Snowflake stage
5. `COPY INTO` tables
6. Validate row counts

---

## 1. Create Database & Schema

```sql
CREATE DATABASE IF NOT EXISTS olist;
CREATE SCHEMA IF NOT EXISTS olist.raw;

USE DATABASE olist;
USE SCHEMA raw;
```

This mirrors a **raw → staging → marts** pattern you’ll use later in dbt.

---

## 2. Create Tables (Explicit Schemas)

### customers

```sql
CREATE OR REPLACE TABLE customers (
    customer_id STRING,
    customer_unique_id STRING,
    customer_zip_code_prefix STRING,
    customer_city STRING,
    customer_state STRING
);
```

### orders

```sql
CREATE OR REPLACE TABLE orders (
    order_id STRING,
    customer_id STRING,
    order_status STRING,
    order_purchase_timestamp TIMESTAMP,
    order_approved_at TIMESTAMP,
    order_delivered_carrier_date TIMESTAMP,
    order_delivered_customer_date TIMESTAMP,
    order_estimated_delivery_date TIMESTAMP
);
```

### order_items

```sql
CREATE OR REPLACE TABLE order_items (
    order_id STRING,
    order_item_id INTEGER,
    product_id STRING,
    seller_id STRING,
    shipping_limit_date TIMESTAMP,
    price NUMBER(10,2),
    freight_value NUMBER(10,2)
);
```

### payments

```sql
CREATE OR REPLACE TABLE payments (
    order_id STRING,
    payment_sequential INTEGER,
    payment_type STRING,
    payment_installments INTEGER,
    payment_value NUMBER(10,2)
);
```

---

## 3. Create a CSV File Format

```sql
CREATE OR REPLACE FILE FORMAT csv_format
TYPE = 'CSV'
FIELD_DELIMITER = ','
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"'
NULL_IF = ('');
```

This prevents:

* Header row ingestion
* Quote issues
* Silent null bugs

---

## 4. Create an Internal Stage

```sql
CREATE OR REPLACE STAGE olist_stage
FILE_FORMAT = csv_format;
```

---

## 5. Upload Files to Snowflake

## 6. Load Data with COPY INTO

### customers

```sql
COPY INTO customers
FROM @olist_stage/customers.csv
```

### orders

```sql
COPY INTO orders
FROM @olist_stage/orders.csv
```

### order_items

```sql
COPY INTO order_items
FROM @olist_stage/order_items.csv
```

### payments

```sql
COPY INTO payments
FROM @olist_stage/payments.csv
```

---

## 7. Validate Loads (Critical Habit)

```sql
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM order_items;
SELECT COUNT(*) FROM payments;
```

Optional sanity checks:

```sql
SELECT * FROM orders LIMIT 10;
SELECT * FROM order_items WHERE order_id IS NULL;
```

