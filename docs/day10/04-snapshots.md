# 04 — Snapshots

Tests tell you whether data is valid **right now**.

But sometimes the question is different:

* “What did we believe yesterday?”
* “When did this value change?”
* “What was the customer’s state at the time of the order?”

If you only keep the latest value, you cannot answer those questions.

This is where **snapshots** come in.

A dbt snapshot tracks changes over time and preserves history.

---

## Why snapshots exist

Most analytics teams eventually face this scenario:

* A dimension table gets updated in place.
* Yesterday’s value is overwritten.
* A stakeholder asks: “When did it change?”

Without history, you guess.

Or you build complex workarounds.

Snapshots are a controlled way to keep change history.

They are especially useful when:

* the upstream system does not provide change history
* you cannot build CDC (change data capture)
* you need historical attributes for reporting

---

## What snapshots are NOT

Snapshots are not a replacement for:

* properly designed raw ingestion
* CDC pipelines
* event logs

Snapshots come with cost:

* they write additional data
* they grow over time
* they add operational complexity

Use them when you need them.

Do not add snapshots “because they are available.”

---

## The core idea: SCD Type 2

Snapshots implement the core idea of **Slowly Changing Dimension Type 2 (SCD2)**.

You keep multiple versions of a row over time.

Each version is valid for a time range.

Conceptually, the snapshot table looks like:

* the original business columns
* plus metadata columns that track validity

Typical metadata (names vary by project and adapter):

* a timestamp when the version started
* a timestamp when the version ended
* a flag indicating current version

The end result:

* you can join facts to the correct historical dimension state

---

## A realistic Olist snapshot use case

Olist includes customer records.

In real business systems, customer attributes change:

* address
* city
* zip code
* even customer segmentation labels (in other datasets)

Suppose you want to answer:

* “How many orders were placed by customers who were in São Paulo at the time?”

If you only have the latest customer city:

* you might incorrectly attribute past orders to the new city

A snapshot lets you keep:

* customer versions over time

Then you can:

* join orders to the customer record valid at the order date

We will not build that full historical join today.

Today is about:

* understanding why snapshots exist
* seeing how configuration works
* building one snapshot correctly

---

## Snapshot decision-making

Before you add a snapshot in production, ask these questions.

### 1) Do we need history?

If no one needs to know “what it used to be,” do not snapshot.

### 2) Does the source already provide history?

If you already ingest change history (CDC, events), you may not need snapshots.

### 3) How quickly does the table change?

Snapshots are more expensive when rows change often.

If a table updates frequently, a snapshot can grow fast.

### 4) Can we define a stable unique key?

Snapshots depend on a stable identifier.

If the “key” changes, snapshots become unreliable.

### 5) Can we define “what counts as a change”?

You need a rule:

* compare values
* or compare timestamps

If you cannot define that, snapshots will be confusing.

---

## Snapshot configuration basics

A snapshot is defined in a `.sql` file under a `snapshots/` directory.

The file contains:

* a `snapshot` block
* a SELECT query that defines the rows you want to track
* a configuration section that defines keys and change detection

You run snapshots with:

```bash
dbt snapshot
```

Snapshots do not run with `dbt run`.

They are their own workflow.

---

## Two common snapshot strategies (basic view)

dbt supports multiple strategies.

We will focus on understanding them conceptually.

### Strategy A: `timestamp`

You track changes based on an `updated_at` column.

If `updated_at` changes, dbt assumes the row changed.

Use this when:

* the source has a reliable update timestamp
* updates always bump `updated_at`

Risk:

* if the timestamp is wrong or not updated, you miss changes

### Strategy B: `check`

You track changes by comparing selected columns.

If any tracked column changes, dbt creates a new version.

Use this when:

* you do not trust `updated_at`
* you want explicit control over what columns define “a change”

Risk:

* expensive if you track many columns
* confusing if you include volatile columns

For Day 10, we keep this simple.

---

## A simple snapshot example concept: customer city changes

Imagine we snapshot customers, tracking changes in city.

We need:

* a unique key: `customer_id`
* a strategy: `check`
* tracked columns: a small set

Conceptual snapshot query:

```sql
SELECT
  customer_id,
  customer_city,
  customer_state
FROM {{ ref('stg_customers') }}
```

Conceptual configuration:

* `unique_key`: `customer_id`
* `strategy`: `check`
* `check_cols`: `customer_city`, `customer_state`

This would record a new version when either city or state changes.

Again:

* we are not adding advanced filtering
* we are not handling all possible change scenarios

We are learning the mechanics and the decision.

---

## Trade-offs you must understand

Snapshots are powerful, but they come with costs.

### Cost 1: Storage growth

A snapshot table grows every time a tracked row changes.

If customers change rarely, growth is slow.

If the table updates constantly, growth can be large.

### Cost 2: Query complexity

Once you have history, you must decide:

* do you want the current version?
* do you want the version at a point in time?

That means:

* more join logic
* more filtering on “current” flags or validity windows

### Cost 3: Operational discipline

Snapshots require consistent runs.

If you run snapshots inconsistently:

* history will have gaps
* you may miss changes

In production, you usually schedule snapshots.

In this course, you will run them manually.

---

## Common misunderstandings

### Misunderstanding 1: “Snapshots are for facts”

Snapshots are usually for dimensions.

Facts tend to be append-only (orders, payments).

When facts do change, you usually handle that with corrections logic, not snapshots.

### Misunderstanding 2: “Snapshots replace incremental models”

They solve different problems.

* incremental models are about performance and loading strategy
* snapshots are about history tracking

### Misunderstanding 3: “Snapshots automatically handle deletes”

Deletes are tricky.

They depend on configuration and your source behavior.

We are not going deep on deletes today.

---
