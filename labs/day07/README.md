# Day 07 Labs — Modeling Design and Layering (Design-Only)

## Lab intent

These labs focus on **modeling judgment**, not execution.

You will not:

* Run dbt
* Write full SQL models
* Join large datasets

You will practice deciding:

* Which layer logic belongs in
* How responsibilities are separated
* How to avoid common modeling anti-patterns

These decisions matter more than syntax.

---

## How to approach these labs

For each scenario:

* Read the requirement carefully
* Decide **where the logic belongs**
* Be able to explain **why**

There may be more than one wrong answer.
The goal is disciplined reasoning.

---

## Lab 01 — Staging vs Not Staging

### Objective

Reinforce what staging models are responsible for.

### Scenario

You receive raw order data with the following needs:

* Column names are inconsistent
* Order timestamps are stored as strings
* Cancelled orders should not appear in business reports

### Tasks

For each requirement, decide:

1. Does this belong in a **staging model**?
2. If not, which layer should handle it?

Explain your reasoning briefly.

---

## Lab 02 — Identifying Intermediate Logic

### Objective

Practice spotting reusable logic.

### Scenario

Multiple reports need:

* Orders joined to customers
* Customer city and state included
* Order date normalized

### Tasks

Answer the following:

1. Should this logic live in a mart model or an intermediate model?
2. Why would duplicating this logic be risky?
3. What benefit does an intermediate layer provide here?

---

## Lab 03 — Mart Responsibility Check

### Objective

Clarify what mart models are responsible for.

### Scenario

A stakeholder asks:

> “I need total revenue per customer per month.”

### Tasks

Decide:

1. Is this requirement best served by a **fact model**, a **dimension model**, or both?
2. What would be the grain of the fact model?
3. What business logic must be clearly visible in the mart?

Avoid writing SQL.
Focus on design.

---

## Lab 04 — Spot the Anti-Pattern

### Objective

Learn to identify bad modeling decisions early.

### Scenario

You see the following proposal:

> “Let’s calculate revenue, customer lifetime value, and churn flags directly in the staging model so everything is ready early.”

### Tasks

Explain:

1. What layers are being misused?
2. What problems this will cause later?
3. How the logic should be reorganized conceptually

---

## Lab 05 — Layer Placement Drill

### Objective

Practice fast, instinctive layer decisions.

### Scenario

For each item below, choose the correct layer:

1. Renaming `cust_id` to `customer_id`
2. Joining orders with payments
3. Defining monthly revenue
4. Creating a reusable cleaned orders dataset

Choose from:

* Staging
* Intermediate
* Mart

Justify each choice in one sentence.

---

## Completion criteria

You are done when you can:

* Defend why logic belongs in a specific layer
* Avoid placing business logic too early
* Design marts that are thin and readable

If you hesitate often, revisit Day 07 theory.

---

## What comes next

Day 08 moves from design to **hands-on modeling**.

Because you practiced placement today:

* SQL will be simpler
* Models will be cleaner
* Refactoring will be rare

Design decisions today determine success tomorrow.
