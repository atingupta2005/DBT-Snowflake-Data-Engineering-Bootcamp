# Day 12 Lab — Jinja, Macros, dbt-utils, and Hooks

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## What you will do today

Up to Day 11, you mostly wrote static SQL.

Today you will start writing dbt the way production teams write it:

* use Jinja to avoid copy-paste SQL
* extract repeated logic into macros
* use a standard package for common utilities (`dbt-utils`)
* add a hook to run an operational command after a model builds

You will make small changes that are easy to review.

---

## Files for this lab (work in this order)

Do these in order.

1. `jinja.md`
2. `macros.md`
3. `dbt-utils.md`
4. `hooks.md`

Do not skip ahead.

---

## 0) Confirm you finished Day 11

From your project root:

```bash
cd ~/dbt_olist_project
source .venv/bin/activate
```

These must work before you start Day 12:

```bash
dbt run --select fct_orders
```

```bash
dbt test
```

If either fails, fix Day 11 first.

---

## 1) Save a git checkpoint (do this before you change anything)

Today you will introduce Jinja and macros.

A small syntax mistake can break parsing.

Checkpoint first so you can see exactly what changed.

```bash
git status
```

If you see `not a git repository`:

```bash
git init
```

Commit your Day 11 state:

```bash
git add -A
git commit -m "day11 checkpoint before day12 jinja macros hooks"
```

If git says “nothing to commit”, that is fine.

---

## Final checkpoints (you are done when all are true)

### A) Jinja compiles

```bash
dbt parse
```

### B) Packages are installed

```bash
dbt deps
```

### C) Changed models build

```bash
dbt run --select stg_customers stg_orders stg_products fct_orders
```

### D) The generated surrogate key exists

Run in Snowflake:

```sql
select
  order_id,
  customer_id,
  order_customer_key
from OLIST.ANALYTICS_DEV.FCT_ORDERS
limit 10;
```

### E) The post-hook ran (best-effort)

If your role can grant permissions, your build log should show the `GRANT` running after `fct_orders`.

If your role cannot grant permissions, `fct_orders` may still build but the hook can error.

In either case, the table should be queryable:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

---

## Compare changes and commit Day 12

Review the diff:

```bash
git diff
```

Commit:

```bash
git add -A
git commit -m "day12 add clean_string macro jinja loop dbt-utils and post-hook"
```
