# 03 — Build Mart Models (Facts and Dimensions)

Now we publish **mart models**.

Marts are the models you expect downstream users to rely on. They should be stable, clearly named, and built from clean intermediate inputs.

In this file, you will:

1. Define mart model responsibilities
2. Define grain explicitly
3. Build one dimension model
4. Build one fact model
5. Run marts with correct dependencies

## Rules for mart models

Mart models are not a place to “keep shaping data forever.”

A mart model must be:

* Built from staging and intermediate models using `ref()`
* Explicit about grain
* Cleanly named and readable
* Focused on one purpose

Mart models should not:

* Pull directly from `source()`
* Mix multiple grains
* Hide business logic across many scattered CTEs

If you need heavy shaping, do it in intermediate first.

## 1) Pick marts to build today

For Day 08, keep marts small and obvious.

You will build at least:

* One **dimension** model
* One **fact** model

We will use the Olist dataset tables only:

* customers
* orders
* order_items
* payments
* products

A practical set is:

* Dimension: customers
* Fact: orders

If your intermediate models support it, you can also build an order items fact. Do not do both facts unless your order fact is already stable.

## 2) Define grain before you write SQL

Do this first.

Write the grain in a single sentence:

* “`dim_customers` is one row per customer.”
* “`fct_orders` is one row per order.”

If you cannot state the grain clearly, stop.

Grain controls everything:

* Which joins are safe
* Whether aggregations are required
* Whether the model is reviewable

## 3) Build a dimension model

Dimension models describe entities.

For Day 08, build a customer dimension.

### What goes into a dimension

Include:

* The entity primary key

* Stable descriptive attributes
  nDo not include:

* Metrics

* Aggregated measures

* Multi-row transaction details

### Surrogate keys (simple)

We will use a simple surrogate key approach.

In many warehouses, dimensions include a surrogate key separate from the natural key. In dbt, this is often implemented using `dbt_utils.generate_surrogate_key`, but Day 08 does not allow custom macros.

For today:

* Keep the natural key as the primary key in the dimension
* Optionally, you can create a surrogate key using a built-in hash function available in Snowflake

If you generate a surrogate key, it must be:

* Deterministic
* Based only on stable natural keys

Do not build complex key logic today.

### Dimension input

A dimension should be built from:

* A staging model, or
* An intermediate model (if you added useful enrichment)

Prefer building from intermediate if you already created enrichment there.

## 4) Build a fact model

Fact models describe events.

For Day 08, build an orders fact.

### What goes into a fact

Include:

* The fact grain key (order_id if one row per order)
* Foreign keys to dimensions (customer_id)
* Important timestamps
* Measures (order totals, payment totals) if you can compute them cleanly

Do not include:

* Repeated descriptive attributes better suited for dimensions
* Fields that create hidden grain changes

### Choosing the right input

Fact models should be built from intermediate models.

If your intermediate order model already:

* Joins customer keys
* Aggregates order items and payments to order level

Then your fact model can be straightforward.

If you are still doing those joins and aggregations inside the fact model, you pushed too much logic into marts.

## 5) `ref()` usage across layers

Mart models must use `ref()`.

* Dimension models should `ref()` staging or intermediate
* Fact models should `ref()` intermediate models

Do not use `source()` in marts.

## Running marts

Run marts only after staging and intermediate are stable.

Select by folder:

```bash
dbt run --select models/marts
```

Or select by model name once you have them:

```bash
dbt run --select dim_customers fct_orders
```

Expected outcome:

* dbt builds upstream dependencies automatically
* Mart models build without errors
* The grain matches what you stated

## Mart sanity checks

Do these checks in Snowflake after the run.

### Dimension checks

* Row count is close to the number of unique customers
* Primary key is not NULL
* Primary key is unique (spot check duplicates)

### Fact checks

* Row count is close to the number of orders
* Order IDs are not duplicated (unless you intentionally changed grain)
* Customer foreign keys look populated

If row counts explode, you almost always joined a multi-row table without aggregating.
