# Lab — Day 14: dbt CI on Pull Requests (GitHub Actions)

Today you will build and validate a slim CI pipeline.

You will commit one workflow file, open a Pull Request, and watch GitHub run dbt for you.

Your goal:

* CI runs on every PR
* CI runs `dbt deps`, `dbt compile`, `dbt build`
* CI authenticates to Snowflake using GitHub Secrets
* CI can run `dbt docs generate`

---

## What you will create

You will create exactly one file:

```text
.github/workflows/dbt.yml
```

Do not create any other files.

---

## Prerequisites

You need:

* your dbt project pushed to a GitHub repo
* permission to add repository secrets
* working Snowflake credentials

CI runs on a clean machine.

That machine does not have your local `profiles.yml`.

So your workflow must create `profiles.yml` at runtime.

---

## Part 0 — Snowflake prep for CI (1 minute)

CI should not write into your DEV or PROD schemas.

In this lab, CI will write to a separate schema:

* CI schema: `OLIST_CI`

In a Snowflake worksheet, run:

```sql
SHOW SCHEMAS LIKE 'OLIST_CI';
```

If it does not exist and you have permission:

```sql
CREATE SCHEMA IF NOT EXISTS OLIST_CI;
```

If schema creation fails, your role does not have the right permissions.

Fix that before you continue.

---

## Part 1 — Create a feature branch

From the repo root:

```bash
git status
```

Expected:

* you are on your main branch
* working tree is clean

Create a branch:

```bash
git checkout -b day14-ci
```

---

## Part 2 — Create the workflow directory

Create the folder path if it does not exist:

```bash
mkdir -p .github/workflows
```

Confirm:

```bash
ls -la .github/workflows
```

Expected:

* directory exists

---

## Part 3 — Add a workflow skeleton (so you can see it run)

Create the workflow file:

```bash
touch .github/workflows/dbt.yml
```

Open it in your editor and add this minimal header:

```yaml
name: dbt CI

on:
  pull_request:

jobs:
  dbt:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v4
```

Commit this skeleton.

```bash
git add .github/workflows/dbt.yml
git commit -m "Add dbt CI workflow skeleton"
```

Push your branch:

```bash
git push -u origin day14-ci
```

What you should see in GitHub:

* your branch exists
* Actions may not run yet (because you haven’t opened a PR)

---

## Part 4 — Add Snowflake secrets in GitHub

Your workflow must not hardcode credentials.

At minimum, store the password as a secret.

Recommended: store all connection fields as secrets.

### Step 4.1 — Required secret

In GitHub:

1. Open your repo
2. Settings
3. Secrets and variables → Actions
4. New repository secret

Create this secret:

```text
SNOWFLAKE_PASSWORD
```

Value:

* your Snowflake password

### Step 4.2 — Recommended secrets (use these if you want CI to be clean and portable)

Create these secrets too:

* `SNOWFLAKE_ACCOUNT`
* `SNOWFLAKE_USER`
* `SNOWFLAKE_ROLE`
* `SNOWFLAKE_WAREHOUSE`
* `SNOWFLAKE_DATABASE`

If you do not create these, you will have to hardcode those values somewhere.

That works, but it’s harder to reuse across repos.

---

## Part 5 — Add the required CI steps

Now extend `.github/workflows/dbt.yml`.

You need these steps in this order:

1. Setup Python
2. Install dbt
3. Create `~/.dbt/profiles.yml`
4. Run `dbt deps`
5. Run `dbt compile`
6. Run `dbt build`

Keep the runner:

* `ubuntu-latest`

Keep the trigger:

* `pull_request`

### Step 5.1 — Set up Python

Use the official action:

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.11"
```

### Step 5.2 — Install dbt

Use pip:

```yaml
- name: Install dbt
  run: |
    python -m pip install --upgrade pip
    pip install dbt-core dbt-snowflake
```

### Step 5.3 — Create `profiles.yml` at runtime

In CI, dbt expects:

* `~/.dbt/profiles.yml`

You will create that file during the run.

This is the safest pattern:

* make the directory
* write YAML using a heredoc

Example skeleton (you must adapt names):

```yaml
- name: Write profiles.yml
  run: |
    mkdir -p ~/.dbt
    cat > ~/.dbt/profiles.yml <<'YAML'
    <your_profile_name>:
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
          schema: "OLIST_CI"
          threads: 4
    YAML
```

Important details:

* `<your_profile_name>` must match `profile:` in `dbt_project.yml`
* `threads` must be an integer
* the password must come from `env_var('SNOWFLAKE_PASSWORD')`
* the `<<'YAML'` quoting prevents the shell from expanding anything accidentally

Quick sanity check (safe):

```yaml
- name: Show profiles.yml path
  run: |
    ls -la ~/.dbt
```

Do not print secrets.

Do not `cat ~/.dbt/profiles.yml` unless you are confident nothing secret is hardcoded.

### Step 5.4 — Export secrets to the job

Your workflow must provide environment variables.

Example pattern:

```yaml
env:
  SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
```

If you created the recommended secrets, export them too:

```yaml
env:
  SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
  SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
  SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
  SNOWFLAKE_ROLE: ${{ secrets.SNOWFLAKE_ROLE }}
  SNOWFLAKE_WAREHOUSE: ${{ secrets.SNOWFLAKE_WAREHOUSE }}
  SNOWFLAKE_DATABASE: ${{ secrets.SNOWFLAKE_DATABASE }}
```

Where to attach `env:`

* easiest: attach it at the job level so every step inherits it
* acceptable: attach it only to the steps that run dbt

### Step 5.5 — Run the dbt commands

Add these steps:

```yaml
- name: dbt deps
  run: dbt deps

- name: dbt compile
  run: dbt compile

- name: dbt build
  run: dbt build
```

If `dbt build` fails, the earlier steps still matter.

Teams use `deps` and `compile` as fast checks before running full builds.

Optional but useful (still slim):

```yaml
- name: dbt debug
  run: dbt debug
```

Put it after the profile is written and before `dbt deps`.

---

## Part 6 — Commit and push

After you add the steps, commit and push.

```bash
git add .github/workflows/dbt.yml
git commit -m "Run dbt build in GitHub Actions"
git push
```

---

## Part 7 — Open a Pull Request and watch CI

In GitHub:

1. Open a Pull Request from `day14-ci` into your main branch
2. On the PR page, find **Checks**
3. Click the running workflow

What you should see in logs:

* a fresh VM starts (Ubuntu)
* Python is installed
* dbt installs via pip
* your workflow writes `~/.dbt/profiles.yml`
* dbt runs `deps`, `compile`, `build`

How to read failures:

* click the failed step
* read from the first red error line upward
* don’t scroll to the bottom first

---

## Part 8 — Force a failure and fix it

You need to see a failure once.

Pick one of these controlled failure options.

### Option A (recommended): remove the secret export

1. Temporarily remove `SNOWFLAKE_PASSWORD` from the workflow `env:`
2. Commit and push

Expected failure:

* dbt cannot authenticate
* error mentions missing password or missing env var

Then restore the `env:` export, commit, and push again.

### Option B: break Python setup

1. Change `python-version` to something incompatible
2. Commit and push

Expected failure:

* the setup-python step fails

Restore Python 3.11, commit, and push again.

---

## Part 9 — Add docs generation (brief)

Add one more step that runs:

```bash
dbt docs generate
```

Do not publish the docs.

Just generate them to prove CI can run documentation tasks.

Example step:

```yaml
- name: dbt docs generate
  run: dbt docs generate
```

Optional verification (safe):

```yaml
- name: List generated docs artifacts
  run: |
    ls -la target || true
```

Expected outcome:

* `target/` exists
* you see files created by docs generation

Commit and push.

---

## Troubleshooting

### The workflow does not run at all

Check these first:

* file path is exactly `.github/workflows/dbt.yml`
* branch is pushed to GitHub
* you opened a Pull Request (not just pushed commits)

### The job fails with auth errors

Check these first:

* secret name is exactly `SNOWFLAKE_PASSWORD`
* workflow exports `SNOWFLAKE_PASSWORD` in `env:`
* generated `profiles.yml` uses `env_var('SNOWFLAKE_PASSWORD')`

If the error looks like “account not found” or “user not found”, you’re missing other connection fields.

Fix by adding the recommended secrets and exporting them.

### The job fails with “profile not found”

dbt cannot find `~/.dbt/profiles.yml`.

Confirm your workflow:

* creates `~/.dbt/`
* writes `profiles.yml` into that directory

### The job fails during `dbt deps`

This usually means:

* your `packages.yml` references a package or version that can’t be fetched
* GitHub Actions can’t reach the internet (rare)

Check the `deps` step logs and fix package references.

### The job fails during `dbt compile`

This usually means:

* a SQL syntax error
* a missing `ref()` target
* a config typo

CI is doing its job.

Fix the code on your branch and push again.

---

## Stop here

Do not merge the PR.

Leave it open with a passing check.
