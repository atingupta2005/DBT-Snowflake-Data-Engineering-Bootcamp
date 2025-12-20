# 03 — Storage and Query Concepts

## Objective

Explain how Snowflake stores data and executes queries so dbt models can be designed for predictable performance and cost.

---

## How Snowflake Stores Table Data

Snowflake stores table data in a centralized, columnar storage layer.

Important characteristics:

* Data is automatically compressed
* Storage is immutable
* Storage is shared by all warehouses

You do not manage files, blocks, or partitions directly.

Example:

* A table `raw.orders` is stored once
* Ten different warehouses can query it at the same time
* Storage cost does not increase with more warehouses

---

## Micro-partitions

Snowflake automatically divides tables into micro-partitions.

Properties:

* Created automatically during data load
* Contain metadata (min/max values per column)
* Immutable once written

```
Table: raw.orders

[ MP1 ][ MP2 ][ MP3 ][ MP4 ] ...
```

You never define micro-partitions explicitly, but your data layout affects them.

---

## Why Micro-partitions Matter

Micro-partition metadata enables **pruning**.

Example:

* Column: `order_date`
* Query filter: `order_date = '2024-01-10'`

Snowflake behavior:

* Scans metadata for micro-partitions
* Reads only partitions that may contain the date
* Skips the rest

Good pruning reduces both runtime and cost.

---

## What Breaks Pruning

Certain patterns reduce pruning effectiveness.

Examples:

* Applying functions in filters
* Casting inside WHERE clauses

Bad example (conceptual):

```
WHERE TO_VARCHAR(order_date) = '2024-01-10'
```

Good example:

```
WHERE order_date = '2024-01-10'
```

This matters directly for dbt model SQL.

---

## Query Execution Flow (Simplified)

```
SQL Query
   |
   v
Optimizer
   |
   v
Micro-partition Pruning
   |
   v
Scan + Compute (Warehouse)
```

The optimizer uses metadata before any compute is consumed.

---

## Virtual Warehouses and Cost

Warehouses control query execution speed.

Key points:

* Larger warehouses finish queries faster
* Cost is based on warehouse runtime
* Storage cost is independent

Example:

* A dbt model runs slowly on X-Small

---

## Query Profiles (Conceptual)

Snowflake provides a query profile for each executed query.

It shows:

* Bytes scanned
* Partitions scanned vs skipped
* Execution stages

You will inspect query profiles later during optimization exercises.

---

## Time Travel and Safe Reprocessing

Snowflake retains historical versions of data.

Examples:

* Query a table as it existed 30 minutes ago
* Recover from accidental deletes

For dbt:

* Failed runs can be investigated safely
* Reprocessing does not require reloading data

---

## Practical Mental Model for dbt

When writing dbt models:

* Assume storage is cheap
* Optimize for pruning, not indexes
* Control cost through warehouse size and runtime

Design SQL so Snowflake can skip as much data as possible.
