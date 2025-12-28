# Lab — Day 14: dbt CI on Pull Requests (GitHub Actions)

Today you will build and validate a slim CI pipeline.

You will commit one workflow file, open a Pull Request, and watch GitHub run dbt for you.

Your goal is simple:

* CI must run on every PR.
* CI must run `dbt deps`, `dbt compile`, `dbt build`.
* CI must authenticate to Snowflake using GitHub Secrets.

---

## What you will create

You will create exactly one file:

```
.github/workflows/dbt.yml
```

Do not create any other files.

---

## Prerequisites

You need:

* Your dbt project pushed to a GitHub repo.
* Permission to add repository secrets.
* Working Snowflake credentials.

---

## Part 1 — Create a feature branch

From the root of your repository:

```bash
git status
```

Expected: you are on your main branch and the working tree is clean.

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

Expected: the directory exists. It may be empty.

---

## Part 3 — Add a workflow file skeleton

Create the workflow file:

```bash
touch .github/workflows/dbt.yml
```

Open it in your editor and add a minimal header.

Use this structure (do not paste a full working solution yet):

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

Commit this skeleton so you can see the workflow appear in GitHub.

```bash
git add .github/workflows/dbt.yml
git commit -m "Add dbt CI workflow skeleton"
```

Push your branch:

```bash
git push -u origin day14-ci
```

---

## Part 4 — Add the required CI steps

Now extend your workflow so the job does all of the following:

1. Setup Python
2. Install dbt packages (dbt-core + dbt-snowflake)
3. Create a `profiles.yml` at runtime
4. Run:

   * `dbt deps`
   * `dbt compile`
   * `dbt build`

### Constraints you must follow

* Runner must be `ubuntu-latest`
* The pipeline must run on `pull_request`
* You must not hardcode your Snowflake password

### Hints for writing the workflow

#### A) Setup Python

Use the official action:

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.11"
```

#### B) Install dbt

Use pip:

```yaml
- name: Install dbt
  run: |
    python -m pip install --upgrade pip
    pip install dbt-core dbt-snowflake
```

#### C) Create a profiles.yml

In CI, dbt expects:

* `~/.dbt/profiles.yml`

You will create that file during the run.

You will need `mkdir -p ~/.dbt` and then write YAML to `~/.dbt/profiles.yml`.

Important rule:

* The password must come from an environment variable named `SNOWFLAKE_PASSWORD`.

That means your profile must include:

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

#### D) Provide the secret to the dbt step

You will attach the secret to the step using `env:`.

Example pattern:

```yaml
env:
  SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
```

---

## Part 5 — Add GitHub repository secret

In GitHub:

1. Open your repo.
2. Go to **Settings**.
3. Go to **Secrets and variables** → **Actions**.
4. Click **New repository secret**.
5. Name it exactly:

```
SNOWFLAKE_PASSWORD
```

6. Paste your Snowflake password as the value.
7. Save.

Do not put the password in the workflow file.

Do not commit it anywhere.

---

## Part 6 — Commit and push the workflow

After you add the steps, commit and push.

```bash
git add .github/workflows/dbt.yml
git commit -m "Run dbt build in GitHub Actions"
git push
```

---

## Part 7 — Open a Pull Request and watch CI

In GitHub:

1. Open a Pull Request from `day14-ci` into your main branch.
2. Go to the PR page.
3. Find the **Checks** section.
4. Click into the running workflow.

You should see the steps run in order.

---

## Part 8 — Force a failure and fix it

You need to see a failure once.

Pick one of these controlled failure options.

### Option A (recommended): remove the secret reference

1. In the workflow, temporarily remove the `env:` block that sets `SNOWFLAKE_PASSWORD`.
2. Commit and push.

Expected failure:

* dbt cannot authenticate
* you will see an error related to missing password / env var

Then restore the `env:` block, commit, and push again.

### Option B: change the Python version to something wrong

1. Change `python-version` to a version that is likely unavailable or incompatible.
2. Commit and push.

Expected failure:

* The setup-python step fails

Then restore Python 3.11, commit, and push.

---

## Part 9 — Add docs generation (brief)

Add one more step that runs:

```bash
dbt docs generate
```

Do not publish the docs.

Just generate them to prove the pipeline can run documentation tasks.

Commit and push.

---

## What you should have when you are done

* A PR with a passing CI check
* A workflow file committed to your repo
* A GitHub secret configured
* A clear understanding of where failures appear (and how to read them)

---

## Troubleshooting

### The workflow does not run at all

Check these first:

* The file path is exactly `.github/workflows/dbt.yml`
* The branch is pushed to GitHub
* You opened a Pull Request (not just pushed commits)

### The job fails at `dbt build` with auth errors

Check these first:

* The secret name is exactly `SNOWFLAKE_PASSWORD`
* The workflow step exports `SNOWFLAKE_PASSWORD` in `env:`
* The generated `profiles.yml` uses `env_var('SNOWFLAKE_PASSWORD')`

### The job fails with “profile not found”

That means dbt cannot find `~/.dbt/profiles.yml`.

Confirm your workflow:

* Creates `~/.dbt/`
* Writes `profiles.yml` to that path

---

Stop here.

Do not merge the PR yet.

In the next step, we will compare your workflow to a working reference in the instructor solution.
