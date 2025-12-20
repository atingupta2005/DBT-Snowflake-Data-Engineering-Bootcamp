# 02 — Working with Snowflake for dbt

## Objective

Explain how Snowflake should be structured and configured to support dbt development, with clear separation between environments, roles, warehouses, and schemas.

---

## Why Snowflake Setup Matters for dbt

dbt assumes the warehouse is the execution engine for transformations. Poor Snowflake structure leads to:

* Accidental overwrites between environments
* Competing workloads on the same compute
* Hard-to-debug permission failures

A small amount of upfront structure avoids these problems.

---

## Core Objects Used by dbt

When dbt runs, it interacts with only a small subset of Snowflake objects.

Required objects:

* Warehouse (compute)
* Database
* Schema
* Role

Example:

* dbt connects using a single role
* That role uses a specific warehouse
* Models are created inside a target schema

---

## Environment Separation Model

A simple and effective model is environment-based isolation.

```
DEV Environment
  ├── Warehouse: WH_DBT_DEV
  ├── Database : ANALYTICS_DEV
  └── Schema   : DBT_ATIN

TEST Environment
  ├── Warehouse: WH_DBT_TEST
  ├── Database : ANALYTICS_TEST
  └── Schema   : DBT_ATIN

PROD Environment
  ├── Warehouse: WH_DBT_PROD
  ├── Database : ANALYTICS
  └── Schema   : CORE
```

Each environment:

* Uses separate compute
* Writes to isolated locations
* Can be promoted safely

---

## Warehouses for dbt

Warehouses provide compute for dbt runs.

Guidelines:

* Use a dedicated warehouse for dbt
* Do not share BI warehouses
* Keep warehouses small in DEV

Example:

* `WH_DBT_DEV` → X-Small
* `WH_DBT_PROD` → Medium

If a dbt run fails, stopping the warehouse immediately stops cost.

---

## Databases and Schemas

Databases represent data lifecycle boundaries.

Common pattern:

* One database per environment
* Multiple schemas inside the database

Example schemas created by dbt:

```
ANALYTICS_DEV.DBT_ATIN.stg_orders
ANALYTICS_DEV.DBT_ATIN.fct_orders
```

dbt owns everything inside its target schema.

---

## Roles and Permissions (Minimal Model)

dbt does not require admin privileges.

Minimal permissions:

* USAGE on warehouse
* USAGE on database
* CREATE on schema
* SELECT on raw/source schemas

Example mental model:

* Platform team creates objects
* dbt role only builds models

This reduces blast radius.

---

## How dbt Uses These Objects

During execution, dbt:

1. Connects using a role
2. Activates a warehouse
3. Creates or replaces tables/views
4. Reads from raw schemas

Example flow:

* `raw.orders` → read only
* `analytics_dev.dbt_atin.stg_orders` → created by dbt

No cross-environment writes occur.

---

## Naming Consistency

Consistent naming prevents confusion.

Recommended:

* Prefix warehouses with workload (`WH_DBT_`)
* Separate environment in database name
* Stable schema name per developer

Bad example:

* One warehouse for everything
* Shared schemas across DEV and PROD

Good naming makes debugging obvious.

---

## Practical Mental Model

When dbt runs, imagine:

* Compute is temporary
* Schemas are disposable
* Raw data is immutable

If a schema is dropped, dbt can rebuild it.
Raw data should never be touched by dbt.
