# 01 — Modern ELT Foundations

## Objective

Explain how data engineering evolved from traditional ETL to modern ELT, and why cloud data warehouses made this shift inevitable.

---

## Traditional ETL: How It Worked

In traditional systems, data pipelines followed an **Extract → Transform → Load** pattern.

* Data was extracted from source systems
* Transformations happened outside the warehouse
* Only transformed data was loaded into the database

This approach worked when:

* Data volumes were small
* Transformations were limited
* Warehouses were expensive and slow to scale

---

## Limitations of Traditional ETL

As data usage grew, ETL pipelines started showing clear problems:

* Transformations required complex, fragile code
* Scaling compute required provisioning new servers
* Debugging failures was difficult
* SQL was not the primary transformation language

ETL tools became bottlenecks instead of enablers.

---

## Rise of Cloud Data Warehouses

Cloud data warehouses changed the economics of data processing.

Key changes:

* Storage became cheap and elastic
* Compute could scale independently
* SQL engines became highly optimized

Examples include Snowflake, BigQuery, and Redshift.

---

## ELT: The Modern Approach

Modern pipelines follow an **Extract → Load → Transform** pattern.

```
Sources
  |
  v
Raw Tables (Warehouse)
  |
  v
Transformations (SQL)
  |
  v
Analytics Tables
```

Key idea:

* Load raw data first
* Transform inside the warehouse using SQL

---

## Why ELT Works Better

ELT aligns with modern warehouse capabilities:

* Transformations run where the data lives
* SQL is easier to read and maintain
* Scaling compute is simple
* Reprocessing data is cheaper

This shift enables faster iteration and better reliability.

---

## Role of SQL in ELT

SQL becomes the primary transformation language.

Benefits:

* Declarative and readable
* Optimized by the warehouse engine
* Easy for analysts and engineers to collaborate

In modern ELT, SQL is treated as **production code**, not ad-hoc queries.

---

## Key Takeaways

* ETL was designed for a different era
* Cloud warehouses made ELT practical
* ELT simplifies pipelines and improves scalability
* SQL-driven transformations are central to modern analytics

---

## Next Topic

Proceed to:

➡️ **02-elt-architecture-overview.md**

This next section explains how ELT is structured into layers.
