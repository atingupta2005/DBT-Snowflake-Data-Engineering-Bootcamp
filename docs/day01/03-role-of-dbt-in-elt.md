# 03 — Role of dbt in ELT

## Objective

Explain why dbt exists in modern ELT architectures and how it standardizes, organizes, and governs SQL-based transformations inside the data warehouse.

---

## Problem Before dbt

Before dbt, SQL transformations were usually handled in ad-hoc ways.

Common patterns:

* Long SQL scripts run manually
* SQL embedded inside ETL tools
* Logic duplicated across BI tools

Example:

* One analyst creates a revenue calculation in a dashboard
* Another analyst reimplements the same logic slightly differently
* Numbers stop matching

The issue was not SQL itself, but the lack of structure around SQL.

---

## What dbt Introduces

```
Raw Tables  →  dbt Models  →  Analytics Tables
```

dbt introduces software engineering discipline to SQL.

Core ideas:

* Treat SQL as code
* Make transformations modular
* Make dependencies explicit
* Test assumptions automatically

---

## dbt as the Transformation Layer

In ELT, dbt owns the transformation layer.

Responsibilities:

* Define transformations using SQL
* Execute transformations inside the warehouse
* Control execution order

Non-responsibilities:

* Data ingestion
* Scheduling ingestion jobs
* Managing source systems

Example:

* Ingestion loads `raw.orders`
* dbt transforms it into `stg_orders`
* dbt builds `fct_orders` from staging models

---

## How dbt Organizes SQL

Instead of one large SQL script, dbt splits logic into models.

Example structure:

```
models/
├── staging/
│   └── stg_orders.sql
├── intermediate/
│   └── int_orders_enriched.sql
└── marts/
    └── fct_orders.sql
```

Each model:

* Has a single purpose
* Can be tested independently
* Can be reused by downstream models

---

## Dependency Management with ref()

Dependencies between models are defined explicitly.

Example (illustrative):

```sql
SELECT
    order_id,
    customer_id,
    order_amount
FROM {{ ref('stg_orders') }}
```

What this achieves:

* dbt knows execution order
* Changes propagate safely
* Lineage becomes visible

No manual ordering is required.

---

## DAG and Lineage

```
raw.orders
     |
     v
stg_orders
     |
     v
fct_orders
```

dbt automatically builds a Directed Acyclic Graph (DAG).

This graph is used to:

* Determine run order
* Visualize dependencies
* Identify downstream impact of changes

---

## Standardization Across Teams

dbt creates a shared standard for analytics teams.

Examples:

* One place for business logic
* Consistent naming conventions
* Centralized tests

Without dbt:

* Logic spreads across dashboards
* Changes are risky

With dbt:

* Logic lives in version-controlled SQL
* Changes are reviewable

---

## What dbt Does Not Replace

It is important to understand boundaries.

What dbt does NOT replace:

* Ingestion tools
* Workflow orchestrators
* BI tools

Example:

* dbt transforms data
* An orchestrator decides *when* dbt runs
* BI tools consume final tables

---

## Practical Mental Model

When deciding whether something belongs in dbt, ask:

* Is this transformation SQL-based?
* Does it depend on warehouse data?
* Is this logic reused across analyses?

If yes, it belongs in dbt.

---

Proceed to hands-on topics only after this mental model is clear.
