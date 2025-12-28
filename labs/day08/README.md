## Day 08 — dbt Model Development (Staging → Intermediate → Mart)

### Constants

* Raw: `OLIST.RAW`
* dev target schema: `OLIST.ANALYTICS_DEV`
* prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## 0) Init project

```bash
source ~/.venv/bin/activate
cd ~
dbt init dbt_olist_project
cd ~/dbt_olist_project
```

---

## 1) Snowflake profile (`~/.dbt/profiles.yml`)

```bash
nano ~/.dbt/profiles.yml
```

Great — then you **should** use two schemas:

* **dev:** `OLIST.ANALYTICS_DEV`
* **prod:** `OLIST.ANALYTICS`

### 1) Correct `profiles.yml` (dev vs prod schemas)

```yml
dbt_olist_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: LFQYFCV-HN31035
      user: OLIST_USER
      password: StrongPassword123
      role: OLIST_ROLE
      warehouse: COMPUTE_WH
      database: OLIST
      schema: ANALYTICS_DEV
      threads: 2
      client_session_keep_alive: false

    prod:
      type: snowflake
      account: LFQYFCV-HN31035
      user: OLIST_USER
      password: StrongPassword123
      role: OLIST_ROLE
      warehouse: COMPUTE_WH
      database: OLIST
      schema: ANALYTICS
      threads: 2
      client_session_keep_alive: false
```

#### Snowflake SQL to create `ANALYTICS_DEV` + grant permissions

Run as an admin role (e.g., `ACCOUNTADMIN`):

```sql
use role accountadmin;
use warehouse compute_wh;

-- ensure role is assigned
grant role OLIST_ROLE to user OLIST_USER;
grant role OLIST_ROLE to user ATINGUPTA2006;

-- allow using warehouse + database
grant usage on warehouse COMPUTE_WH to role OLIST_ROLE;
grant usage on database OLIST to role OLIST_ROLE;

-- create dev schema
create schema if not exists OLIST.ANALYTICS_DEV;

-- allow dbt to create objects in dev
grant usage on schema OLIST.ANALYTICS_DEV to role OLIST_ROLE;
grant create table on schema OLIST.ANALYTICS_DEV to role OLIST_ROLE;
grant create view  on schema OLIST.ANALYTICS_DEV to role OLIST_ROLE;
grant modify on schema OLIST.ANALYTICS_DEV to role OLIST_ROLE;

-- (recommended) future grants so the role can query what it creates
grant select on future tables in schema OLIST.ANALYTICS_DEV to role OLIST_ROLE;
grant select on future views  in schema OLIST.ANALYTICS_DEV to role OLIST_ROLE;
```

---

## 2) Project config (`dbt_project.yml`)

```bash
cd ~/dbt_olist_project
nano dbt_project.yml
```

Ensure:

```yml
name: 'dbt_olist_project'
profile: 'dbt_olist_project'
```

Models block:

```yml
models:
  dbt_olist_project:
    +persist_docs:
      relation: true
      columns: true
    staging:
      +materialized: view
    intermediate:
      +materialized: view
    marts:
      +materialized: table
```

---

## 3) Quick checks

```bash
dbt --version
dbt debug
dbt debug --target prod
```

---

## 4) Folders

```bash
mkdir -p models/staging models/intermediate models/marts
```

---

## 5) Sources

Create:

```bash
nano models/staging/_sources.yml
```

```yml
version: 2

sources:
  - name: olist
    schema: RAW
    tables:
      - name: customers
        identifier: CUSTOMERS
      - name: orders
        identifier: ORDERS
      - name: order_items
        identifier: ORDER_ITEMS
      - name: payments
        identifier: PAYMENTS
      - name: products
        identifier: PRODUCTS
```

Validate:

```bash
dbt parse
```

---

## 6) Staging models (`models/staging/`)

### `stg_customers.sql`

```bash
nano models/staging/stg_customers.sql
```

```sql
select
  customer_id,
  customer_unique_id,
  cast(customer_zip_code_prefix as number) as customer_zip_code_prefix,
  lower(trim(customer_city)) as customer_city,
  upper(trim(customer_state)) as customer_state
from {{ source('olist', 'customers') }}
```

### `stg_orders.sql`

```bash
nano models/staging/stg_orders.sql
```

```sql
select
  order_id,
  customer_id,
  lower(trim(order_status)) as order_status,
  cast(order_purchase_timestamp as timestamp) as order_purchase_ts,
  cast(order_approved_at as timestamp) as order_approved_ts,
  cast(order_delivered_carrier_date as timestamp) as order_delivered_carrier_ts,
  cast(order_delivered_customer_date as timestamp) as order_delivered_customer_ts,
  cast(order_estimated_delivery_date as date) as order_estimated_delivery_date
from {{ source('olist', 'orders') }}
```

### `stg_order_items.sql`

```bash
nano models/staging/stg_order_items.sql
```

```sql
select
  order_id,
  cast(order_item_id as number) as order_item_id,
  product_id,
  seller_id,
  cast(shipping_limit_date as timestamp) as shipping_limit_ts,
  cast(price as number(18,2)) as item_price,
  cast(freight_value as number(18,2)) as freight_value
from {{ source('olist', 'order_items') }}
```

### `stg_payments.sql`

```bash
nano models/staging/stg_payments.sql
```

```sql
select
  order_id,
  cast(payment_sequential as number) as payment_sequential,
  lower(trim(payment_type)) as payment_type,
  cast(payment_installments as number) as payment_installments,
  cast(payment_value as number(18,2)) as payment_value
from {{ source('olist', 'payments') }}
```

### `stg_products.sql`

```bash
nano models/staging/stg_products.sql
```

```sql
select
  product_id,
  lower(trim(product_category_name)) as product_category_name,
  cast(product_name_lenght as number) as product_name_length,
  cast(product_description_lenght as number) as product_description_length,
  cast(product_photos_qty as number) as product_photos_qty,
  cast(product_weight_g as number) as product_weight_g,
  cast(product_length_cm as number) as product_length_cm,
  cast(product_height_cm as number) as product_height_cm,
  cast(product_width_cm as number) as product_width_cm
from {{ source('olist', 'products') }}
```

Run staging:

```bash
dbt run --select stg_customers stg_orders stg_order_items stg_payments stg_products
```

---

## 7) Intermediate (`models/intermediate/`)

### `int_payments_agg.sql`

```bash
nano models/intermediate/int_payments_agg.sql
```

```sql
with payments as (
  select
    order_id,
    payment_type,
    payment_installments,
    payment_value
  from {{ ref('stg_payments') }}
),

agg as (
  select
    order_id,
    sum(payment_value) as payment_total_value,
    max(payment_installments) as max_payment_installments,
    count(*) as payment_row_count,
    min(payment_type) as payment_type_any
  from payments
  group by order_id
)

select *
from agg
```

### `int_order_items_agg.sql`

```bash
nano models/intermediate/int_order_items_agg.sql
```

```sql
with items as (
  select
    order_id,
    product_id,
    item_price,
    freight_value
  from {{ ref('stg_order_items') }}
),

agg as (
  select
    order_id,
    count(*) as order_item_row_count,
    count(distinct product_id) as distinct_product_count,
    sum(item_price) as items_total_value,
    sum(freight_value) as freight_total_value
  from items
  group by order_id
)

select *
from agg
```

### `int_orders_enriched.sql`

```bash
nano models/intermediate/int_orders_enriched.sql
```

```sql
with orders as (
  select
    order_id,
    customer_id,
    order_status,
    order_purchase_ts,
    order_approved_ts,
    order_delivered_carrier_ts,
    order_delivered_customer_ts,
    order_estimated_delivery_date
  from {{ ref('stg_orders') }}
),

items_agg as (
  select
    order_id,
    order_item_row_count,
    distinct_product_count,
    items_total_value,
    freight_total_value
  from {{ ref('int_order_items_agg') }}
),

payments_agg as (
  select
    order_id,
    payment_total_value,
    max_payment_installments,
    payment_row_count,
    payment_type_any
  from {{ ref('int_payments_agg') }}
),

final as (
  select
    o.order_id,
    o.customer_id,
    o.order_status,
    o.order_purchase_ts,
    o.order_approved_ts,
    o.order_delivered_carrier_ts,
    o.order_delivered_customer_ts,
    o.order_estimated_delivery_date,

    coalesce(i.order_item_row_count, 0) as order_item_row_count,
    coalesce(i.distinct_product_count, 0) as distinct_product_count,
    coalesce(i.items_total_value, 0) as items_total_value,
    coalesce(i.freight_total_value, 0) as freight_total_value,

    coalesce(p.payment_total_value, 0) as payment_total_value,
    p.max_payment_installments,
    coalesce(p.payment_row_count, 0) as payment_row_count,
    p.payment_type_any
  from orders o
  left join items_agg i
    on o.order_id = i.order_id
  left join payments_agg p
    on o.order_id = p.order_id
)

select *
from final
```

Run intermediate:

```bash
dbt run --select int_payments_agg int_order_items_agg int_orders_enriched
```

Validate grain (Snowflake):

```sql
select
  count(*) as nrows,
  count(distinct order_id) as distinct_orders
from OLIST.ANALYTICS_DEV.INT_ORDERS_ENRICHED;
```

---

## 8) Marts (`models/marts/`)

### `dim_customers.sql`

```bash
nano models/marts/dim_customers.sql
```

```sql
select
  customer_id,
  customer_unique_id,
  customer_zip_code_prefix,
  customer_city,
  customer_state
from {{ ref('stg_customers') }}
```

### `fct_orders.sql`

```bash
nano models/marts/fct_orders.sql
```

```sql
select
  order_id,
  customer_id,
  order_status,
  order_purchase_ts,
  order_approved_ts,
  order_delivered_carrier_ts,
  order_delivered_customer_ts,
  order_estimated_delivery_date,

  order_item_row_count,
  distinct_product_count,
  items_total_value,
  freight_total_value,

  payment_total_value,
  max_payment_installments,
  payment_row_count,
  payment_type_any
from {{ ref('int_orders_enriched') }}
```

Run marts (with parents):

```bash
dbt run --select +dim_customers +fct_orders
```

Validate marts (Snowflake):

```sql
select count(*) as nrows, count(distinct customer_id) as distinct_customers
from OLIST.ANALYTICS_DEV.DIM_CUSTOMERS;
```

```sql
select count(*) as nrows, count(distinct order_id) as distinct_orders
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

---

## 9) Docs

```bash
dbt docs generate
```

---

## 10) Prod

```bash
dbt run --target prod --select +dim_customers +fct_orders
```

```bash
dbt docs generate --target prod
dbt docs serve
```

---

## 11) Troubleshooting

```bash
dbt debug
```

```bash
dbt compile --select stg_orders
```

```bash
dbt run --select +int_orders_enriched
```

```bash
dbt clean
```

```bash
dbt compile
```

```bash
dbt run
```
