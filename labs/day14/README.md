# Day 14 Lab — dbt CI on Pull Requests (GitHub Actions)

## What this lab does

This lab sets up **pull‑request CI for dbt**.

Every time you open or update a PR, GitHub will:

* spin up a fresh machine
* install dbt
* connect to Snowflake
* validate your project

Nothing here depends on your laptop.

That is the point.

---

## Project context (fixed values)

These values are assumed throughout this lab.

* Git provider: GitHub
* dbt project name: `dbt_olist_project`
* Snowflake database: `OLIST`
* CI schema: `ANALYTICS_CI`
* Warehouse: `COMPUTE_WH`
* Python version: `3.11`

---

## Prerequisites

Before you start, confirm:

* The dbt project is pushed to GitHub
* You can create branches and PRs
* You can add GitHub repository secrets
* You have working Snowflake credentials

Important context:

* CI runs on a **clean VM** every time
* There is **no local state**
* There is **no local profiles.yml**

If the workflow does not create `profiles.yml`, dbt will fail.

---

## Step 0 — Snowflake preparation for CI

CI must never write into DEV or PROD.

CI failures should be cheap and disposable.

For that reason, CI writes to its own schema:

* `ANALYTICS_CI`

This schema exists only to validate code changes.

---

### Step 0.1 — Confirm CI schema

```sql
SHOW SCHEMAS LIKE 'ANALYTICS_CI';
```

If missing and permitted:

```sql
CREATE SCHEMA IF NOT EXISTS ANALYTICS_CI;
```

---

### Step 0.2 — Confirm CI permissions

The role used by CI must be able to:

* use the database
* use the CI schema
* create and replace tables/views

Minimum grants:

```sql
GRANT USAGE ON DATABASE OLIST TO ROLE OLIST_ROLE;
GRANT USAGE ON SCHEMA OLIST.ANALYTICS_CI TO ROLE OLIST_ROLE;
GRANT CREATE TABLE ON SCHEMA OLIST.ANALYTICS_CI TO ROLE OLIST_ROLE;
GRANT CREATE VIEW  ON SCHEMA OLIST.ANALYTICS_CI TO ROLE OLIST_ROLE;
```

If these are missing, CI will connect and then fail at runtime.

Fix this now.

---

## Step 1 — Create a feature branch

From the repo root:

```bash
git status
```

Expected:

* You are on `main`
* Working tree is clean

Create a branch:

```bash
git checkout -b day14-ci
```

---

## Step 2 — Create workflow directory

```bash
mkdir -p .github/workflows
```

Confirm:

```bash
ls -la .github/workflows
```

---

## Step 3 — Create workflow file

```bash
touch .github/workflows/dbt.yml
```

Open the file and start with:

```yaml
name: dbt CI

on:
  pull_request:

jobs:
  dbt:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
```

Commit and push:

```bash
git add .github/workflows/dbt.yml
git commit -m "Add dbt CI workflow skeleton"
git push -u origin day14-ci
```

---

## Step 4 — Add GitHub secrets

CI must not hardcode credentials.

Add these repository secrets.

Required:

* `SNOWFLAKE_PASSWORD`

Recommended (strongly):

* `SNOWFLAKE_ACCOUNT`
* `SNOWFLAKE_USER`
* `SNOWFLAKE_ROLE`
* `SNOWFLAKE_WAREHOUSE`
* `SNOWFLAKE_DATABASE`

---

## Step 5 — Complete the CI workflow

This is the core of the lab.

You will now describe, in YAML, exactly what CI does.

Every line here runs on a brand‑new machine.

Nothing is cached.

---

### Why we write profiles.yml in CI

CI does not have access to your laptop.

It does not know your Snowflake account.
It does not know your role.
It does not know your password.

The only safe option is to:

* store secrets in GitHub
* write `profiles.yml` at runtime

This is the standard dbt CI pattern.

---

### Why we use a heredoc

We use:

```
<<'YAML'
```

This prevents:

* shell variable expansion
* accidental secret leakage
* quoting bugs

Do not change this unless you know why.

---

Replace the file with the following complete workflow.

```yaml
name: dbt CI

on:
  pull_request:

jobs:
  dbt:
    runs-on: ubuntu-latest

    env:
      SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
      SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
      SNOWFLAKE_ROLE: ${{ secrets.SNOWFLAKE_ROLE }}
      SNOWFLAKE_WAREHOUSE: ${{ secrets.SNOWFLAKE_WAREHOUSE }}
      SNOWFLAKE_DATABASE: ${{ secrets.SNOWFLAKE_DATABASE }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dbt
        run: |
          python -m pip install --upgrade pip
          pip install dbt-core dbt-snowflake

      - name: Write profiles.yml
        run: |
          mkdir -p ~/.dbt
          cat > ~/.dbt/profiles.yml <<'YAML'
          dbt_olist_project:
            target: ci
            outputs:
              ci:
                type: snowflake
                account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
                user: "{{ env_var('SNOWFLAKE_USER') }}"
                password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
                role: "{{ env_var('SNOWFLAKE_ROLE') }}"
                warehouse: "{{ env_var('SNOWFLAKE_WAREHOUSE') }}"
                database: "{{ env_var('SNOWFLAKE_DATABASE') }}"
                schema: "ANALYTICS_CI"
                threads: 4
          YAML

      - name: dbt debug
        run: dbt debug

      - name: dbt deps
        run: dbt deps

      - name: dbt compile
        run: dbt compile

      - name: dbt build
        run: dbt build

      - name: dbt docs generate
        run: dbt docs generate
```

---

## Step 6 — Commit and push workflow

```bash
git add .github/workflows/dbt.yml
git commit -m "Run dbt CI on pull requests"
git push
```

---

## Step 7 — Open Pull Request

In GitHub:

1. Open a PR from `day14-ci` to `main`
2. Open the **Checks** tab
3. Click the running workflow

You should see:

* Ubuntu runner starts
* Python installs
* dbt installs
* profiles.yml is created
* dbt commands run in order

---

## Step 8 — Validate failure handling

Trigger a controlled failure.

Option A (recommended):

* Remove `SNOWFLAKE_PASSWORD` from `env:`
* Commit and push

Expected:

* dbt debug fails with auth error

Restore the secret and push again.

---

