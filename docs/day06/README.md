# Day 06 — Core dbt Commands, Execution Flow, and Lineage Fundamentals

## Why execution starts here

Up to Day 05, dbt was **conceptual**.

You learned:

* What a dbt project is
* How it is structured
* Where configuration lives
* Why discipline matters before execution

Nothing ran.
That was intentional.

Day 06 is where dbt becomes **real**.

---

## What changes on Day 06

Today is the first time you actually **run dbt**.

This day introduces:

* How dbt connects to the warehouse
* What happens when a dbt command runs
* How dbt decides execution order
* How lineage is created and read

The goal is not complex transformations.
The goal is understanding **execution mechanics**.

---

## What Day 06 is (and is not)

Today focuses on **how dbt behaves**, not what you build.

Today you will:

* Run core dbt commands
* Validate that your project is wired correctly
* Observe how dbt processes a project
* Read lineage created by dbt

Today you will not:

* Build business models
* Write deep SQL logic
* Implement incremental strategies
* Design production pipelines

That comes next.

---

## How Day 05 connects to Day 06

Everything you learned yesterday now shows up in behavior.

* Project structure controls discovery
* Configuration controls execution
* Environment boundaries control where data is written
* Git history explains what changed

If Day 05 was skipped, Day 06 feels confusing.
If Day 05 was understood, Day 06 feels predictable.

---

## What you should understand by the end of today

By the end of Day 06, you should be able to:

* Explain what happens when dbt runs
* Know when to use each core dbt command
* Understand how dbt determines execution order
* Read basic lineage and reason about impact
* Spot setup issues early, before modeling begins

This confidence is required before writing real models.

---

## Documents for Day 06

Read these in order:

1. **01-core-dbt-commands.md**
   What the main dbt commands do and when they are used

2. **02-dbt-dag-and-lineage.md**
   How dbt resolves dependencies and builds execution order

3. **03-materializations.md**
   Why dbt materializes data differently and what options exist

Each document builds on the previous one.

---

## Looking ahead to Day 07

Day 07 is where modeling begins.

You will:

* Write your first real dbt models
* Apply SQL discipline inside dbt
* Use structure and execution knowledge together

Day 06 ensures that when you model, nothing feels magical.

Execution first.
Modeling next.
