# GitHub Actions Setup for dbt

This file is about one thing: making dbt run on every Pull Request.

You are going to add a workflow file:

```
.github/workflows/dbt.yml
```

That workflow will:

1. Check out your repo
2. Install Python
3. Install dbt
4. Configure Snowflake credentials using secrets
5. Run `dbt deps`, `dbt compile`, `dbt build`

We keep it slim and predictable.

---

## What a workflow file is

A GitHub Actions workflow is a YAML file.

YAML is indentation-sensitive. Two spaces matter.

If the indentation is wrong, GitHub Actions will not run the workflow.

Your workflow lives in the repo so it is versioned like code.

---

## Workflow structure you must recognize

A typical workflow has these top-level sections:

* `name`: a human-readable label
* `on`: what events trigger it
* `jobs`: what runs on the runner

Inside a job you usually see:

* `runs-on`: which runner OS
* `steps`: the ordered list of commands/actions

Each step either:

* Uses a prebuilt action (`uses:`)
* Runs shell commands (`run:`)

---

## The trigger: Pull Requests

We want CI to run when:

* A Pull Request is opened
* A Pull Request gets new commits

In GitHub Actions, that is `pull_request`.

Example:

```yaml
on:
  pull_request:
```

That is enough for most repos.

---

## The runner: ubuntu-latest

We will use:

```yaml
runs-on: ubuntu-latest
```

This gives us a clean Ubuntu machine each run.

No Docker, no custom images, no extra complexity.

---

## Step 1: Checkout the repository

The runner starts empty.

Checkout copies your repo code into the runner workspace.

You almost always want this first.

```yaml
- name: Checkout repo
  uses: actions/checkout@v4
```

If you forget checkout:

* dbt cannot find `dbt_project.yml`
* Every dbt command fails

---

## Step 2: Install Python

We need Python to install dbt.

Use the official action:

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.11"
```

### Which version should you choose

Use a Python version that:

* Works with your dbt version
* Is stable and widely available

In many teams, Python 3.11 is a good default today.

If your project requires a different version, pin it.

Do not rely on whatever Python happens to be on the runner.

---

## Step 3: Install dbt (and your adapter)

dbt runs through Python packages.

In CI you should install:

* `dbt-core`
* The adapter for Snowflake: `dbt-snowflake`

Example:

```yaml
- name: Install dbt
  run: |
    python -m pip install --upgrade pip
    pip install dbt-core dbt-snowflake
```

### Why we upgrade pip

Old pip versions sometimes fail dependency resolution.

Upgrading pip is cheap insurance.

---

## Step 4: Configure credentials using secrets

dbt connects to Snowflake through a `profiles.yml` file.

In this course, your `profiles.yml` should already use `env_var()` for the password.

That design matters.

It lets CI inject secrets without committing them.

### What we store as a GitHub Secret

At minimum, store:

* `SNOWFLAKE_PASSWORD`

In many repos you will also store:

* `SNOWFLAKE_USER`
* `SNOWFLAKE_ACCOUNT`
* `SNOWFLAKE_ROLE`
* `SNOWFLAKE_DATABASE`
* `SNOWFLAKE_WAREHOUSE`
* `SNOWFLAKE_SCHEMA`

Whether you need these as secrets depends on how your `profiles.yml` is written.

If some values are safe and stable (like role or warehouse), you may keep them in `profiles.yml`.

Passwords should always be secrets.

### How secrets are exposed to the workflow

In GitHub Actions, secrets are available as:

* `secrets.SNOWFLAKE_PASSWORD`

You can pass them into the environment of a step.

Example:

```yaml
- name: Run dbt
  env:
    SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
  run: |
    dbt --version
```

Then in `profiles.yml`, dbt reads it through `env_var('SNOWFLAKE_PASSWORD')`.

### Where the profile file lives in CI

dbt looks for profiles under the `~/.dbt/` directory by default.

In CI, we will create it at runtime.

We do this because:

* It keeps secrets out of the repo
* It makes CI self-contained

A common pattern is:

1. Create `~/.dbt/`
2. Write a `profiles.yml`
3. Run dbt

---

## Step 5: Run dbt deps, compile, build

We run these commands in order:

```bash
dbt deps
dbt compile
dbt build
```

### Why we run them as separate commands

If `dbt build` fails and it is really a dependency problem, the logs can be noisy.

Running `dbt deps` first gives you a clean, short failure.

Running `dbt compile` next isolates compile errors.

Then `dbt build` does the heavier work.

---

## Brief: Generating dbt docs in CI

dbt can generate documentation artifacts.

The two commands are:

```bash
dbt docs generate
dbt docs serve
```

In CI you will usually run only:

```bash
dbt docs generate
```

Because CI is not an interactive web server.

### What publishing looks like in real teams

A common pattern is:

* Generate docs in CI
* Upload the artifacts somewhere

  * GitHub Pages
  * S3
  * Internal static hosting

We are not implementing publishing today.

We want you to understand:

* Docs generation can be automated like tests
* It becomes part of “definition of done”

---

## What you will implement in the lab

In the lab, you will:

* Create `.github/workflows/dbt.yml`
* Add the steps described above
* Add `SNOWFLAKE_PASSWORD` as a GitHub secret
* Open a Pull Request
* Watch the CI check pass or fail

Next: `labs/day14/README.md`
