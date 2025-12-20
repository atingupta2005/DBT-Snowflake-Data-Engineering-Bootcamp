# Git Integration for dbt Projects

## Why dbt projects must live in Git

dbt projects are **code**.

They change over time.
They are reviewed.
They are shared.

Without version control:

* Changes are invisible
* Mistakes are hard to undo
* Collaboration breaks down

Git provides history.
History provides safety.

---

## What changes over time in a dbt project

As analytics work progresses, teams modify:

* SQL transformations
* Configuration defaults
* Folder structure
* Tests and assumptions

These changes are not one-time events.
They are continuous.

Git allows the team to:

* See what changed
* Understand why it changed
* Revert when needed

---

## Why analytics engineering requires version control

Analytics outputs influence decisions.
Decisions rely on trust.

Version control supports that trust by making work:

* Reviewable
* Auditable
* Reproducible

If a number changes:

* You can trace the change
* You can inspect the logic
* You can discuss it as a team

This is not optional.
It is foundational.

---

## What should be committed

Commit files that define **logic and structure**:

* `dbt_project.yml`
* SQL models
* Macros
* Tests
* Seeds
* Documentation files

If a file explains *how the project works*, it belongs in Git.

---

## What must never be committed

Never commit files that contain:

* Passwords
* Tokens
* User-specific credentials
* Local machine paths

Examples include:

* `profiles.yml`
* Environment-specific secrets
* Temporary or personal files

If a file answers **who you are**, not **what the project is**, it does not belong in Git.

---

## Discipline before writing the first model

Before adding transformation logic:

* The repository must exist
* The structure must be agreed upon
* Configuration boundaries must be clear

This discipline prevents:

* Accidental leaks
* Inconsistent behavior
* Unreviewable changes

Once models appear, history starts mattering.

Getting Git right first avoids pain later.

---

## Team workflows and reviewability

In team settings:

* Changes are proposed
* Logic is reviewed
* Assumptions are questioned

Git enables these conversations.

A dbt project without Git is:

* Fragile
* Opaque
* Hard to trust

A dbt project with Git is:

* Intentional
* Understandable
* Safe to evolve

---

## Why this comes before execution

Once dbt starts running:

* Results appear quickly
* Errors propagate fast

Git ensures that every result has a trail.

That trail starts **before** the first model is written.

Execution comes next.
Version control comes first.
