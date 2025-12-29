# Day 09 Lab — Sources, Source Tests, and Freshness

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## 0) Save a checkpoint (do this before you change anything)

You are about to change YAML that affects how *every* staging model reads raw data.

If something breaks, you want a clean `git diff` that shows exactly what changed.

From your project root:

```bash
cd ~/dbt_olist_project
source ~/.venv/bin/activate
```

Check whether you already have git initialized:

```bash
git status
```

* If you see a status like `On branch main`, you are good.
* If you see `not a git repository`, initialize once:

```bash
git init
```

Now checkpoint Day 08 so you can compare later:

```bash
git add -A
git commit -m "day08 checkpoint before day09 sources"
```

If git says “nothing to commit”, that is fine. It means your working tree is clean.

---

## 1) Replace your sources file with the Day 09 version

### What this change does

* Makes raw tables explicit as `source()` objects.
* Adds test definitions that run with `dbt test`.
* Adds freshness configuration that runs with `dbt source freshness`.

### File to edit

Open your existing Day 08 sources file:

```bash
nano models/staging/_sources.yml
```

Now replace the *entire file* with the content below (copy/paste).

After pasting, save and exit.

```yml
version: 2

sources:
  - name: olist
    database: OLIST
    schema: RAW
    description: "Raw Olist tables loaded into Snowflake. These tables are upstream inputs and are not transformed by dbt."

    tables:
      - name: customers
        identifier: CUSTOMERS
        description: "One row per customer."
        columns:
          - name: customer_id
            description: "Customer primary key."
            tests:
              - not_null
              - unique

          - name: customer_unique_id
            description: "Stable customer identifier across multiple orders."

      - name: orders
        identifier: ORDERS
        description: "One row per order. Contains lifecycle timestamps and status."

        loaded_at_field: order_purchase_timestamp
        freshness:
          warn_after: {count: 4000, period: day}
          error_after: {count: 7000, period: day}

        columns:
          - name: order_id
            description: "Order primary key."
            tests:
              - not_null
              - unique

          - name: customer_id
            description: "Customer placing the order. Joins to customers.customer_id."
            tests:
              - not_null
              - relationships:
                  to: source('olist', 'customers')
                  field: customer_id

          - name: order_status
            description: "Order lifecycle status from the source system."
            tests:
              - not_null
              - accepted_values:
                  values:
                    - created
                    - approved
                    - processing
                    - invoiced
                    - shipped
                    - delivered
                    - canceled
                    - unavailable

          - name: order_purchase_timestamp
            description: "Timestamp when the purchase was placed. Used as a proxy for load time in this training repo."

      - name: order_items
        identifier: ORDER_ITEMS
        description: "Line items for orders. Multiple rows per order_id."

        loaded_at_field: shipping_limit_date
        freshness:
          warn_after: {count: 4000, period: day}
          error_after: {count: 7000, period: day}

        columns:
          - name: order_id
            description: "Order key. Joins to orders.order_id."
            tests:
              - not_null
              - relationships:
                  to: source('olist', 'orders')
                  field: order_id

          - name: order_item_id
            description: "Line number within the order."
            tests:
              - not_null

          - name: product_id
            description: "Product key. Joins to products.product_id."
            tests:
              - not_null
              - relationships:
                  to: source('olist', 'products')
                  field: product_id

      - name: payments
        identifier: PAYMENTS
        description: "Payment records for orders. Multiple rows per order_id when there are multiple payment attempts."
        columns:
          - name: order_id
            description: "Order key. Joins to orders.order_id."
            tests:
              - not_null
              - relationships:
                  to: source('olist', 'orders')
                  field: order_id

          - name: payment_sequential
            description: "Payment sequence number within an order."
            tests:
              - not_null

      - name: products
        identifier: PRODUCTS
        description: "Product catalog table."
        columns:
          - name: product_id
            description: "Product primary key."
            tests:
              - not_null
              - unique

          - name: product_category_name
            description: "Product category label from the source system."
```

---

## 2) Parse (fail fast on YAML mistakes)

`dbt parse` validates that your YAML is valid and that dbt can load the project.

Run:

```bash
dbt parse
```

What to expect:

* It should finish without errors.
* If it fails, do not move on. Fix YAML first.

Most common YAML problems:

* indentation is inconsistent (use 2 spaces only)
* tabs were used instead of spaces
* `tests:` block is mis-indented under `columns:`

---

## 3) Run only the source tests

This command runs tests defined on sources (not model tests):

* `--select source:olist` means “only sources under the `olist` source name”

Run:

```bash
dbt test --select source:olist
```

What to expect:

* You will see each test name and a `PASS`/`FAIL` result.
* A failure means the raw data contract is broken *before* transforms run.

How to interpret failures:

* `not_null` fails: upstream load produced missing keys
* `unique` fails: upstream load duplicated rows
* `relationships` fails: child rows exist without a matching parent key
* `accepted_values` fails: raw status values are outside the allowed list

If a test fails and you suspect the YAML reference is wrong, validate in Snowflake with quick checks.

Example checks:

```sql
-- duplicates for a supposed primary key
select
  customer_id,
  count(*) as nrows
from OLIST.RAW.CUSTOMERS
group by customer_id
having count(*) > 1
order by nrows desc;
```

```sql
-- orphaned child keys
select
  oi.order_id,
  count(*) as nrows
from OLIST.RAW.ORDER_ITEMS oi
left join OLIST.RAW.ORDERS o
  on oi.order_id = o.order_id
where o.order_id is null
group by oi.order_id
order by nrows desc;
```

---

## 4) Run freshness checks

Freshness checks answer one question:

* “How long ago was the most recently loaded record in this source table?”

dbt computes:

* `max(loaded_at_field)` for the table
* compares it to “now”
* returns `pass`, `warn`, or `error` based on your thresholds

Run:

```bash
dbt source freshness --select source:olist
```

What to expect:

* You should see a freshness row for:

  * `orders` (uses `order_purchase_timestamp`)
  * `order_items` (uses `shipping_limit_date`)

Tables without `loaded_at_field` will not appear in freshness results. That is expected.

If freshness fails with an error like “field not found”:

* confirm the column exists in the raw table

```sql
desc table OLIST.RAW.ORDERS;
```

---

## 5) Controlled freshness failure (temporary)

### Why we do this

You should see freshness checks *actually change* from `pass` to `warn`/`error` when thresholds tighten.

If you never test this, freshness stays theoretical.

### What you will do

You will temporarily tighten freshness thresholds for `orders` only.

That makes dbt report staleness for `orders`, because the dataset is historical.

### Step 5A) Tighten thresholds for `orders`

Edit only the `freshness:` block under the `orders` table in `models/staging/_sources.yml`.

Replace it with:

```yml
freshness:
  warn_after: {count: 1, period: day}
  error_after: {count: 2, period: day}
```

Save and exit.

### Step 5B) Run freshness and read the status

Run:

```bash
dbt source freshness --select source:olist
```

What to look for:

* `orders` should show `warn` or `error`.
* dbt will also display timestamps and how far apart they are.

If `orders` still shows `pass` (rare), tighten further:

```yml
freshness:
  warn_after: {count: 1, period: hour}
  error_after: {count: 2, period: hour}
```

Run the freshness command again.

### Step 5C) Restore stable thresholds

Change `orders` back to the wide thresholds so your lab runs are predictable:

```yml
freshness:
  warn_after: {count: 4000, period: day}
  error_after: {count: 7000, period: day}
```

Run freshness one last time:

```bash
dbt source freshness --select source:olist
```

You are not trying to “make freshness green forever”.

You are proving:

* thresholds control `pass/warn/error`
* freshness is an upstream health signal

---

## 6) If `accepted_values` fails for `order_status`

Do not guess.

Pull the actual raw values from Snowflake and make your YAML match the raw data exactly.

Run:

```sql
select
  order_status,
  count(*) as nrows
from OLIST.RAW.ORDERS
group by order_status
order by nrows desc;
```

Then update the `accepted_values` list under `orders.order_status`.

---

## 7) Compare changes and commit Day 09

See exactly what changed from your Day 08 checkpoint:

```bash
git diff
```

If the diff looks correct, commit Day 09:

```bash
git add -A
git commit -m "day09 define sources tests and freshness"
```
