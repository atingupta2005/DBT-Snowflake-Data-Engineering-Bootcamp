# 01 — Snowflake Architecture

## Objective

Explain Snowflake’s core architectural components and how separation of storage and compute supports scalable ELT workloads.

---

## Core Architectural Principle

Snowflake is built on a strict separation of **storage**, **compute**, and **services**. Each component scales independently.

```
            +-------------------+
            |   Cloud Services  |
            | (metadata, auth,  |
            |  optimizer)       |
            +---------+---------+
                      |
      +---------------+----------------+
      |                                |
+-----v------+                  +------v-----+
| Warehouse  |                  | Warehouse   |
|  (Compute) |                  |  (Compute)  |
+-----+------+                  +------+------+
      |                                |
      +---------------+----------------+
                      |
              +-------v-------+
              |   Storage     |
              | (Tables,      |
              |  micro-parts) |
              +---------------+
```

This design avoids the traditional trade-off between performance and cost.

---

## Storage Layer

Snowflake stores all table data in a centralized, columnar storage layer.

Characteristics:

* Automatically compressed
* Immutable micro-partitions
* Shared across all compute clusters

Example:

* A table `raw.orders` is stored once.
* Multiple warehouses can query it at the same time.
* No data duplication occurs.

This is why creating additional warehouses does not increase storage cost.

---

## Micro-partitions (Conceptual)

Tables are internally divided into micro-partitions.

Key properties:

* Created automatically
* Contain metadata (min/max values)
* Enable pruning during query execution

Example:

* A query filters `order_date = '2024-01-01'`
* Snowflake scans only micro-partitions that may contain that date

You do not manage micro-partitions manually. Design decisions affect them indirectly.

---

## Compute Layer (Virtual Warehouses)

Compute is provided by **virtual warehouses**.

A warehouse:

* Executes SQL queries
* Can be started or stopped independently
* Does not own data

Example:

* `WH_DBT_DEV` runs dbt transformations
* `WH_BI` serves dashboard queries

If one workload spikes, the other remains unaffected.

---

## Concurrency and Scaling

Snowflake handles concurrency by allowing multiple warehouses to access the same data.

Example:

* dbt runs transformations on `WH_DBT`
* Analysts query dashboards on `WH_ANALYTICS`
* Both read the same tables without blocking

For heavy workloads, a warehouse can be scaled up or configured for multi-cluster execution.

---

## Cloud Services Layer

This layer coordinates the platform.

Responsibilities:

* Authentication and access control
* Query optimization
* Metadata management
* Transaction coordination

Example:

* When dbt runs a model, Snowflake uses metadata to determine which micro-partitions to scan.

You do not interact with this layer directly.

---

## Databases, Schemas, and Objects

Logical organization in Snowflake is hierarchical.

```
Account
  └── Database
       └── Schema
            └── Tables / Views
```

Example:

* `RAW` database stores ingested data
* `ANALYTICS` database stores dbt models
* Schemas separate domains or environments

This structure maps cleanly to ELT layering.

---

## Time Travel and Cloning (Conceptual)

Snowflake supports querying historical data and cloning objects.

Examples:

* Query a table as it existed 1 hour ago
* Clone a schema for testing without copying data

These features reduce risk during transformations and deployments.

---

## Why This Architecture Fits ELT

Snowflake’s architecture aligns naturally with ELT:

* Raw data can be stored cheaply
* Transformations run close to the data
* Compute can be isolated per workload

Example:

* A failed dbt run can be re-executed using a different warehouse size
* No data reload is required

---

## Practical Mental Model

When designing pipelines on Snowflake:

* Assume storage is shared
* Treat warehouses as disposable compute
* Isolate workloads by warehouse

These assumptions guide all dbt and environment design decisions later in the course.
