# 02 — ELT Architecture Overview

## Objective

Describe the standard layers of a modern ELT architecture and clearly separate ingestion responsibilities from transformation responsibilities.

---

## The Modern Analytics Stack (High Level)

Modern analytics platforms are built as a set of clearly separated layers. Each layer has a single responsibility.

```
[ Source Systems ]
        |
        v
[ Ingestion Layer ]
        |
        v
[ Raw Data (Warehouse) ]
        |
        v
[ Transformation Layer (dbt) ]
        |
        v
[ Analytics / BI ]
```

The key idea is separation of concerns. Each layer solves one problem well.

---

## Source Systems

Source systems are where data is originally produced.

Examples:

* Application databases (orders, users, payments)
* SaaS tools (CRM, marketing platforms)
* Event streams

These systems are optimized for operational workloads, not analytics.

Example:

* An application database stores orders row by row as users place them.
* Querying this database directly for analytics would impact application performance.

---

## Ingestion Layer

The ingestion layer is responsible for moving data from source systems into the data warehouse.

Responsibilities:

* Extract data
* Load data as-is
* Preserve source structure

Non-responsibilities:

* Business logic
* Joins
* Aggregations

Example:

* Copy the `orders` table from the application database into the warehouse as `raw.orders`.
* No renaming, no casting, no calculations.

---

## Raw Data Layer (Warehouse)

Raw data lives inside the cloud data warehouse.

Characteristics:

* One-to-one mapping with source systems
* Minimal or no transformations
* Full history retained when possible

```
raw.orders
raw.customers
raw.payments
```

This layer exists so transformations can be re-run safely without re-ingesting data.

---

## Transformation Layer

The transformation layer converts raw data into analytics-ready tables.

This is where:

* Cleaning happens
* Business rules are applied
* Multiple datasets are combined

In modern ELT, this layer runs **inside the warehouse** using SQL.

Example transformation steps:

1. Standardize column names
2. Convert data types
3. Join related datasets
4. Create facts and dimensions

This layer is owned by analytics engineers.

---

## Where dbt Fits

```
Raw Tables  --->  dbt Models  --->  Analytics Tables
```

dbt operates entirely in the transformation layer.

What dbt does:

* Defines transformations as SQL models
* Manages dependencies between models
* Runs transformations in the warehouse
* Applies tests and generates documentation

What dbt does NOT do:

* Extract data
* Load data from source systems

---

## Analytics and BI Layer

This is where end users consume data.

Examples:

* Dashboards
* Reports
* Ad-hoc analysis

These tools expect:

* Clean schemas
* Stable table names
* Well-defined metrics

Good ELT architecture ensures BI tools never touch raw data directly.

---

## Layer Ownership Summary

```
Source Systems        → Application teams
Ingestion Layer       → Platform / Data engineering
Raw Data              → Platform ownership
Transformation (dbt)  → Analytics engineering
Analytics / BI        → Analysts / Business users
```

Clear ownership prevents duplicated logic and confusion.

---

## Common Architecture Mistakes

Example 1: Business logic in ingestion

* Aggregating data during ingestion
* Result: Hard to change logic later

Example 2: BI tools querying raw tables

* Each dashboard re-implements logic
* Result: Inconsistent metrics

Example 3: Mixing responsibilities

* ETL tools doing transformations and analytics logic
* Result: Unmaintainable pipelines

---

## Practical Mental Model

When designing pipelines, always ask:

* Is this ingestion or transformation?
* Should this logic live in raw data or dbt?
* Will I need to re-run this logic in the future?

If the answer involves business rules or joins, it belongs in the transformation layer.

---

## What Comes Next

The next topic explains **why dbt exists** and how it standardizes the transformation layer.

Proceed to **03-role-of-dbt-in-elt.md**.
