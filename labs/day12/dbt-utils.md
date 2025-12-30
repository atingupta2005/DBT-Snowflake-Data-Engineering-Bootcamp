# Day 12 Lab — dbt-utils

In this file you will install one dbt package and use one macro from it.

Package:

* `dbt-utils`

Macro:

* `dbt_utils.generate_surrogate_key`

You will add a generated key to `fct_orders`.

---

## Why packages exist

In dbt, packages are shared macro libraries.

They help you avoid rewriting the same utility logic across projects.

In real teams, packages are used for:

* standard surrogate key generation
* safe date spine generation
* generic tests and comparisons

Today we use one macro only.

---

## A) Add `dbt-utils` to your project

### A1) Create `packages.yml`

From the dbt project root (same folder as `dbt_project.yml`), create:

```bash
nano packages.yml
```

Paste:

```yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
```

Save and exit.

### A2) Download packages

Run:

```bash
dbt deps
```

Expected behavior:

* dbt creates/updates `dbt_packages/`
* dbt downloads `dbt_utils`

If this fails:

* you may be in the wrong directory
* confirm you are in the folder that contains `dbt_project.yml`

---

## B) Confirm dbt can see the package

Run:

```bash
dbt parse
```

If `dbt parse` fails and mentions `dbt_utils`:

* rerun `dbt deps`
* confirm `packages.yml` is correctly indented

---

## C) Use `generate_surrogate_key` in `fct_orders`

### What a surrogate key is

A surrogate key is a generated identifier.

In analytics engineering, you often generate surrogate keys for:

* stable joins across multiple columns
* composite keys (where no single natural key exists)
* incremental merge keys (in some models)

Today, we will generate a key from:

* `order_id`
* `customer_id`

This is just training.

In a real project, you would generate surrogate keys only when you need them.

---

### C1) Open `fct_orders.sql`

```bash
nano models/marts/fct_orders.sql
```

### C2) Add the generated key in the final SELECT

Find the final `SELECT` list.

Add this column after `customer_id`:

```sql
  {{ dbt_utils.generate_surrogate_key(['order_id', 'customer_id']) }} AS order_customer_key,
```

Important:

* Keep the trailing comma if more columns follow.
* Do not remove any existing columns.

---

## D) Parse and run

### D1) Parse

```bash
dbt parse
```

If parsing fails:

* confirm you wrote `dbt_utils` (underscore)
* confirm your brackets are balanced (`[` `]`)

### D2) Run the model

Run:

```bash
dbt run --select fct_orders
```

Because `fct_orders` is incremental, this should run a merge.

If you see errors, read them carefully.

Most issues at this step are syntax mistakes in the SELECT list.

---

## E) Verify in Snowflake

### E1) Confirm the column exists

Run:

```sql
select
  order_id,
  customer_id,
  order_customer_key
from OLIST.ANALYTICS_DEV.FCT_ORDERS
limit 20;
```

Expected outcome:

* `order_customer_key` is populated (not null)

---

## F) Sanity check the key is stable

You want the key to be deterministic.

That means:

* same inputs → same output

Pick one order_id and run this query twice.

```sql
select
  order_id,
  customer_id,
  order_customer_key
from OLIST.ANALYTICS_DEV.FCT_ORDERS
where order_id is not null
limit 1;
```

Now rerun the exact same query.

Expected:

* the key value stays the same

---

## G) Common classroom failures

### G1) “dbt_utils is undefined”

This usually means `dbt deps` was not run successfully.

Fix:

```bash
dbt deps
dbt parse
```

### G2) “Compilation error in fct_orders.sql”

This is usually:

* missing comma in the SELECT list
* mismatch in brackets/quotes

Fix pattern:

1. run `dbt compile --select fct_orders`
2. open the compiled SQL under `target/compiled/`
3. find the exact line where SQL is broken

To find the compiled file:

```bash
find target/compiled -type f -name "fct_orders.sql" -print
```

---

## Done criteria

You are done with this file when:

```bash
dbt deps
```

```bash
dbt parse
```

```bash
dbt run --select fct_orders
```

all succeed, and in Snowflake you can query:

* `order_customer_key` from `OLIST.ANALYTICS_DEV.FCT_ORDERS`

---

## Next file

Next you will add a post-hook to run a GRANT after building `fct_orders`.

Go to `hooks.md`.
