# Day 10 Lab — Model Tests, One Custom Test, One Seed, One Snapshot

## Constants

* Raw schema: `OLIST.RAW`
* dbt dev target schema: `OLIST.ANALYTICS_DEV`
* dbt prod target schema: `OLIST.ANALYTICS`
* Warehouse: `COMPUTE_WH`

---

## What you will do today

You will add reliability checks and controlled state management to your dbt project.

By the end of this lab you will have:

* Model-level tests on two existing models (`dim_customers`, `fct_orders`)
* One custom singular test (a business rule)
* One seed table (small reference data, versioned in git)
* One snapshot (history tracking for changing attributes)

You will run everything yourself.

---

## 0) Confirm you finished Day 09

Day 10 builds on Day 09.

If sources are failing or freshness is broken, your model tests and snapshots will be noisy.

From your project root:

```bash
cd ~/dbt_olist_project
source ~/.venv/bin/activate
```

Run:

```bash
dbt test --select source:olist
```

```bash
dbt source freshness --select source:olist
```

Both must succeed before continuing.

---

## 1) Save a git checkpoint (do this before you change anything)

Today you will add YAML tests, a custom SQL test, a seed CSV, and a snapshot.

Checkpoint first so you can compare Day 09 vs Day 10 changes.

```bash
git status
```

If you see `not a git repository`:

```bash
git init
```

Commit the Day 09 state:

```bash
git add -A
git commit -m "day09 checkpoint before day10 tests seeds snapshots"
```

If git says “nothing to commit”, that is fine.

---

## Lab order (follow this exactly)

Work through these in order.

1. `01-model-tests.md`
2. `02-custom-test.md`
3. `03-seed.md`
4. `04-snapshot.md`

Do not skip ahead.

---

## Final checkpoints (you are done when all succeed)

Run these from the project root.

```bash
dbt test --select dim_customers fct_orders
```

```bash
dbt test --select test_type:singular
```

```bash
dbt seed --select order_status_map
```

```bash
dbt test --select order_status_map
```

```bash
dbt run --select rpt_orders_by_status_group
```

```bash
dbt snapshot --select customers_snapshot
```

---

## Compare changes and commit Day 10

Review your changes:

```bash
git diff
```

Commit:

```bash
git add -A
git commit -m "day10 add model tests singular test seed snapshot"
```

---

