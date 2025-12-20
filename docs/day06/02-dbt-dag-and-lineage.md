# dbt Execution Flow, DAG, and Lineage

## How dbt discovers work

When dbt runs, it starts by **reading the project**.

It looks for:

* SQL files in `models/`
* References between those files
* Configuration that affects behavior

Each SQL file becomes a **node** in dbt’s internal graph.

At this stage, dbt is not executing anything.
It is building an understanding of relationships.

---

## Dependencies and `ref()`

Dependencies in dbt are declared explicitly using `ref()`.

A simplified example:

```
SELECT
    order_id,
    customer_id
FROM {{ ref('stg_orders') }}
```

This does two things at once:

* It tells dbt where the data comes from
* It tells dbt that one model depends on another

The dependency is **not inferred** from SQL text.
It is declared.

That declaration is critical.

---

## How execution order is determined

dbt uses declared dependencies to determine order.

The rule is simple:

* A model runs only after its dependencies succeed

From this, dbt builds a **Directed Acyclic Graph**.

* Directed — dependencies have direction
* Acyclic — no circular dependencies
* Graph — models connected by relationships

You do not manually control order.
The graph does.

---

## Why order matters

Order ensures correctness.

Without enforced order:

* Downstream models might run too early
* Results would be incomplete or wrong

With enforced order:

* Inputs always exist
* Logic builds step by step
* Failures stop propagation

This makes execution predictable.

---

## What the DAG represents

The dbt DAG represents:

* Logical dependencies
* Execution order
* Data flow through the project

It is **not** a schedule.
It is **not** a performance plan.

It is a map of relationships.

---

## Lineage as a consequence

Lineage is derived from the DAG.

Because dbt knows:

* What depends on what
* In which direction data flows

It can answer questions like:

* If this model changes, what is affected?
* Where did this column come from?
* Why did this downstream model fail?

Lineage is not extra work.
It is a byproduct of discipline.

---

## Using lineage to reason about issues

When something breaks:

* Start upstream
* Check dependencies
* Follow the graph

This avoids guessing.

Lineage narrows the problem space.

---

## Simple mental model

Think of dbt execution like this:

1. Read the project
2. Build the dependency graph
3. Order models safely
4. Execute step by step

Lineage is the visible form of that process.

Understanding this flow removes mystery.

Execution is mechanical.
Relationships drive everything.
