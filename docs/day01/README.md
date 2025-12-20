# Day 01 — Modern ELT Foundations and Role of dbt

## Objective

Introduce the modern ELT mental model, explain why it replaced traditional ETL, and set clear boundaries between ingestion, transformation, and analytics.

---

## Context

Analytics platforms did not start with ELT. Early data systems were built for reporting on small datasets with limited concurrency. As data volume and usage increased, the original assumptions stopped working.

Example:

* A single nightly ETL job populates summary tables.
* Business teams later ask for new metrics during the day.
* The pipeline cannot adapt without re-engineering.

This gap between data demand and pipeline design is what led to modern ELT.

---

## Mental Model: From ETL to ELT

```
ETL (Old Model)                ELT (Modern Model)
--------------                ------------------
Extract                        Extract
  |                              |
Transform (external engine)    Load (raw data)
  |                              |
Load                           Transform (SQL in warehouse)
```

The critical change is *where* transformation happens.

In ELT:

* Raw data is loaded first
* Transformations run inside the warehouse
* SQL becomes the primary transformation language

---

## Why This Model Scales

Cloud data warehouses changed two constraints:

1. **Compute elasticity**

   * You can scale up or down without redesigning pipelines

2. **Separation of storage and compute**

   * Storing raw data is cheap
   * Reprocessing data is safe and repeatable

Example:

* A broken transformation can be fixed and re-run without reloading source data.

---

## Role Separation Across the Stack

```
Source Systems  →  Ingestion  →  Raw Data  →  Transformations  →  Analytics
```

Each step has a single responsibility:

* Ingestion moves data
* Raw data preserves history
* Transformations apply logic
* Analytics consumes curated tables

Mixing these responsibilities leads to fragile systems.

---

## How to Use This Module

Read this module to understand *how to think* about analytics systems.

Do not focus on tools yet.

Every design decision later in the course assumes this mental model is clear.
