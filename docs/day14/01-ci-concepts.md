# CI Pipeline Concepts

Today you are going to run dbt without touching your laptop.

That sounds small. It is not.

Once your project runs in CI, your team can trust Pull Requests. You stop asking:

* “Did you run the tests?”
* “Which version of dbt did you use?”
* “Works on my machine… why is it failing for you?”

CI does not remove all bugs.

It removes **avoidable** bugs: missing dependencies, broken SQL, failing tests, and bad configuration.

---

## What CI is

**CI (Continuous Integration)** is the practice of automatically checking code every time it changes.

In this course, “change” means:

* You push commits to a branch.
* You open a Pull Request.
* You update a Pull Request.

The key idea: **every change is validated the same way, every time, by a clean machine.**

That clean machine is a runner (a temporary virtual machine) that GitHub creates for the job.

---

## Why we check code before merging

In a team repo, merging is the irreversible moment.

Once code is merged:

* Everyone pulls it.
* Downstream pipelines might run.
* Someone else has to debug it.

You can avoid most of that pain if the branch proves it can build before it merges.

CI is the gate.

You can still merge broken code if you ignore the checks.

But the whole point of CI is that the checks are visible, consistent, and hard to argue with.

---

## What CI is protecting you from

Here are the failures we want CI to catch **before** merge.

### 1) Broken SQL

You change a model and it no longer compiles.

Example causes:

* A column name typo
* A missing comma
* A reference to a model that no longer exists

In dbt terms, `dbt compile` is the fastest “did I break SQL?” check.

### 2) Missing dependencies

You add a package, but forget to run `dbt deps`.

Locally it might work because you already ran it yesterday.

CI starts from scratch every time, so it will fail immediately.

### 3) Failing data tests

You change logic and it violates a data test.

Example: you write a model where `customer_id` is no longer unique.

Your tests should catch that.

CI makes sure nobody merges changes that break data contracts.

### 4) Configuration drift

Your laptop has:

* A cached package install
* A different dbt version
* A different Python version

CI gives you one controlled environment.

If the project only works on your laptop, it is not production-ready.

---

## What runs CI

GitHub Actions is GitHub’s built-in automation system.

You define **workflows** as YAML files in:

```
.github/workflows/
```

A workflow defines:

* When it runs (events like Pull Request)
* What machine it runs on (runner like `ubuntu-latest`)
* What steps it performs (checkout code, install dbt, run commands)

---

## Virtual machine mental model

When a workflow runs, GitHub gives you a fresh runner.

Treat it like:

* A brand-new Ubuntu machine
* Empty disk
* No project files
* No Python dependencies
* No dbt installed

Your workflow must do everything needed to run dbt.

That is the whole value: the script is the truth.

If your workflow forgets a step, CI fails.

---

## What “Slim CI” means here

We are not building a complex pipeline.

We want the smallest set of checks that prevents broken merges.

For this course, slim CI means:

1. Install what dbt needs
2. Resolve dbt dependencies
3. Compile the project
4. Build models and run tests

That maps to these commands:

```bash
dbt deps
dbt compile
dbt build
```

### Why these three commands

* `dbt deps` ensures your project has all packages.
* `dbt compile` ensures SQL + Jinja compile cleanly.
* `dbt build` runs models and tests in the correct order.

`dbt build` is the main safety net.

But running `dbt deps` and `dbt compile` first gives faster, clearer failures.

---

## What CI needs from you

CI is non-interactive.

That means:

* No prompts
* No typing passwords
* No “click to login”

So CI needs credentials provided as **secrets**.

We will use GitHub repository secrets for Snowflake auth.

At minimum, we will set:

* `SNOWFLAKE_PASSWORD`

Other connection details may also be set as secrets depending on how your dbt profile is configured.

The rule is simple:

* Secrets belong in GitHub Settings.
* Code stays in the repo.

Never commit passwords.

---

## Real-world use cases

This is not a classroom-only workflow.

This is how teams run dbt in production.

### Use case 1: PR safety

A teammate changes a model used by revenue reporting.

CI runs `dbt build`.

If tests fail, the PR cannot be merged.

You just prevented bad numbers from reaching stakeholders.

### Use case 2: Catching missing packages

Someone updates `packages.yml`.

They forget to run `dbt deps` locally.

CI fails immediately.

That failure saves time because the fix is obvious.

### Use case 3: Standardized tooling

One person is on dbt 1.x, another is on dbt 1.y.

They get different behavior.

CI pins the version in one place.

The repo becomes consistent.

---

## What to do when CI fails

Treat CI failures like production failures.

Do this every time:

1. Open the failed job.
2. Scroll to the first error.
3. Fix the cause.
4. Push a commit.
5. Let CI re-run.

Do not start with the last error.

Most errors cascade.

The first error is usually the real one.

---

Next: `02-github-actions-setup.md`
