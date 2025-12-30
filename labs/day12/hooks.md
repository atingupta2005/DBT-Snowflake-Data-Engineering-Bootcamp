# Day 12 Lab — Hooks

In this file you will add a **post-hook** to a model.

A post-hook runs after dbt builds the model.

We will use a post-hook to run a GRANT.

This is a common operational pattern:

* dbt builds tables
* dbt applies the permissions needed for downstream users

---

## What hooks are (plain English)

Hooks are SQL statements dbt runs automatically:

* `pre_hook`: runs before dbt builds a model
* `post_hook`: runs after dbt builds a model

A hook runs in the same database context as the model build.

That means:

* it uses your target schema
* it uses your credentials/role

Hooks are powerful.

That also means hooks can break deployments if you are careless.

We will keep this hook simple:

* grant select on the model to a role

---

## What we will do today

We will attach a post-hook to `fct_orders`.

The hook SQL will be:

* `GRANT SELECT ON <this table> TO ROLE OLIST_ROLE`

In dbt, `{{ this }}` resolves to the built relation.

So the hook targets the correct schema in dev/prod.

---

## A) Add the post-hook to `fct_orders`

### A1) Open the file

```bash
nano models/marts/fct_orders.sql
```

### A2) Update the config block

At the top of the file, you already have a config block.

Replace the entire config block with the one below.

Keep it as the first thing in the file.

```sql
{{
  config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='order_id',
    on_schema_change='sync_all_columns',
    post_hook=["GRANT SELECT ON {{ this }} TO ROLE OLIST_ROLE"]
  )
}}
```

Save and exit.

---

## B) What can go wrong (read before you run)

This hook assumes:

* the role you are using has permission to grant on objects it owns
* the role `OLIST_ROLE` exists

In a corporate Snowflake environment, either assumption can be false.

Two important behaviors to understand:

1. The model build can succeed, but the post-hook can fail.
2. The error message will tell you exactly what permission is missing.

In production, teams often treat hook failures as deployment failures.

In this training environment, we will read the error and learn from it.

---

## C) Parse (fail fast)

Run:

```bash
dbt parse
```

If parse fails:

* confirm you used `post_hook=["..."]` (a list)
* confirm your quotes are balanced

---

## D) Run the model and watch the logs

Run:

```bash
dbt run --select fct_orders
```

While it runs, watch for:

* the model build step (MERGE)
* the post-hook SQL execution

If the hook runs successfully, you should see dbt execute the `GRANT`.

If the hook fails, dbt will show an error that often contains:

* “insufficient privileges”
* “role does not exist”

Do not ignore that error.

Read it.

---

## E) Verify the table is still queryable

Even if the GRANT fails, the model itself should exist.

Run in Snowflake:

```sql
select count(*) as nrows
from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

---

## F) Best-effort verification that the GRANT applied

This is environment-dependent.

If you have access to a second role in class, test it.

If you do not, use one of these checks.

### Option 1: Inspect grants on the table

Run:

```sql
show grants on table OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

Look for a row showing:

* privilege: `SELECT`
* grantee: `ROLE`
* name: `OLIST_ROLE`

If you do not see it:

* the hook may have failed
* or your role cannot view grants

### Option 2: Try selecting as another role (if available)

If your environment allows role switching:

```sql
use role OLIST_ROLE;
select count(*) as nrows from OLIST.ANALYTICS_DEV.FCT_ORDERS;
```

If this works, the hook did what you wanted.

If it fails:

* the role does not exist
* or the grant did not apply
* or your user cannot switch roles

---

## G) Why `{{ this }}` matters (don’t skip)

If you hardcode the table name in the hook:

* it will break when you change target schema
* it will break when you run in prod

Using `{{ this }}` makes the hook portable.

In dev it resolves to:

* `OLIST.ANALYTICS_DEV.FCT_ORDERS`

In prod it resolves to:

* `OLIST.ANALYTICS.FCT_ORDERS`

That is why hooks belong in config.

---

## H) Common hook mistakes

### H1) Writing a string instead of a list

This is wrong:

```sql
post_hook="GRANT SELECT ..."
```

Use a list:

```sql
post_hook=["GRANT SELECT ..."]
```

### H2) Forgetting quotes inside the list

This is wrong:

```sql
post_hook=[GRANT SELECT ON {{ this }} TO ROLE OLIST_ROLE]
```

You need quotes:

```sql
post_hook=["GRANT SELECT ON {{ this }} TO ROLE OLIST_ROLE"]
```

### H3) Putting the hook somewhere other than config

Hooks must be in the config block.

Do not put GRANT statements at the bottom of the model SQL.

That becomes hard to reason about and harder to maintain.

---

## Done criteria

You are done with this file when:

```bash
dbt parse
```

and

```bash
dbt run --select fct_orders
```

succeed, and in Snowflake:

* you can query `OLIST.ANALYTICS_DEV.FCT_ORDERS`

Best-case:

* `SHOW GRANTS` confirms `OLIST_ROLE` has SELECT

If the GRANT fails due to permissions, you still pass the lab as long as:

* you can explain what the error means
* you can point to the hook SQL that ran

---

## Next step

Go back to `labs/day12/README.md` and run the final checkpoints.
