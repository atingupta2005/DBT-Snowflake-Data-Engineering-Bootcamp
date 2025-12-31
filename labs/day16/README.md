# Day 16 Lab — End-to-End Walkthrough of dbt Cloud (Student Runbook)

## Step 1 — Open dbt Cloud

Open a browser and navigate to:

```
https://cloud.getdbt.com
```

Log in using your organization’s dbt Cloud account.

After login, you should see:

* An organization name at the top
* One or more dbt projects

If you see multiple projects, open:

```
dbt_olist_project
```

---

## Step 2 — Connect and Clone the Project from GitHub

Before you can work on any code in dbt Cloud, the project must be **connected to GitHub and cloned**.

This step usually happens once, when the project is first created.

### Step 2.1 — Open Project Settings

From the dbt Cloud navigation:

1. Click **Settings**
2. Click **Projects**
3. Select the project `dbt_olist_project`

---

### Step 2.2 — Connect to GitHub

Inside the project settings:

1. Open the **Repository** or **Version Control** section
2. Choose **GitHub** as the provider

You will be redirected to GitHub for authorization.

Authorize dbt Cloud to access repositories under your GitHub account or organization.

---

### Step 2.3 — Select the Repository

After authorization:

1. Choose the GitHub repository that contains the dbt project
2. Repository example:

```
https://github.com/your-org/dbt_olist_project
```

3. Select the default branch (usually `main`)
4. Save the configuration

At this point, dbt Cloud clones the repository into its managed environment.

You do not need to run `git clone` locally.

---

### Step 2.4 — What Cloning Means in Practice

dbt Cloud now:

* Pulls all project files from GitHub
* Keeps the project in sync with the selected branch
* Creates development branches automatically for each user

All files you see in the IDE come directly from GitHub.

---

## Step 3 — Open the dbt Cloud IDE

From the left navigation menu:

1. Click **Develop**
2. Click **IDE**

Wait for the IDE to load completely.

You should now see:

* File tree on the left
* Editor in the center
* Logs panel at the bottom

This replaces:

* Your local editor
* Your terminal window

---

## Step 4 — Verify Project Files

In the file tree, expand:

```
models/
staging/
```

Open:

```
models/staging/_sources.yml
```

Confirm the following values:

* Source name: `olist`
* Database: `OLIST`
* Schema: `RAW`

These files are **standard dbt files**, not dbt Cloud–specific.

---

## Step 5 — Run a Model from the IDE

Open the model:

```
models/staging/stg_orders.sql
```

At the top of the editor:

1. Click **Run**
2. Select **Run Model**

Observe the bottom panel:

* Compilation output
* Execution logs
* Success status

This is equivalent to running locally:

```
dbt run --select stg_orders
```

---

## Step 6 — Run Tests

From the IDE command bar:

1. Click **Run**
2. Select **Run tests**

Watch the logs as tests execute.

These tests come from YAML files already in the project.

This is equivalent to:

```
dbt test
```

---

## Step 7 — Review Run History

In the IDE, open **Run History**.

Click the most recent run.

Review:

* Models executed
* Execution order
* Timing per model

Logs here are the same as CLI logs, but stored centrally.

---

## Step 8 — Understand Environments

Navigate to:

* **Deploy** → **Environments**

Open **Development**.

Note:

* Schema naming pattern
* Connection details

Now open **Production** and compare.

Important rule:

> Development runs never write to production schemas.

---

## Step 9 — Create a Deployment Job

Navigate to:

* **Deploy** → **Jobs**
* Click **Create Job**

Configure the job:

* Job name: `olist_daily_run`
* Environment: Production
* Commands:

```
dbt run
dbt test
```

Save the job.

This replaces a scheduled shell script or CI job.

---

## Step 10 — Schedule the Job

In the job configuration:

* Enable scheduling
* Set frequency to **Daily**
* Choose a time appropriate for your region

Save the schedule.

---

## Step 11 — Configure Notifications

In the same job:

* Enable email notifications on failure
* Enable Slack notifications if available

No custom alert scripts are required.

---

## Step 12 — Review a Job Run

After the job executes:

1. Open the job
2. Click the latest run

Review:

* Logs
* Warnings or failures
* Execution timing

This is your primary production observability view.

---

## Step 13 — What dbt Cloud Adds Beyond the CLI

Without changing dbt itself, dbt Cloud provides:

* Managed execution infrastructure
* Built-in scheduling
* Centralized logs
* Notifications
* Hosted documentation
* Environment guardrails

The dbt commands are the same.
The surrounding operations are simplified.

---

## Step 14 — How Code Is Stored and Versioned

When you edit files in the IDE:

* Code is saved to a Git branch
* Each developer works on their own branch
* Production jobs run only from the main branch

The IDE is a Git-backed editor.

If dbt Cloud is removed, the code still exists fully in Git.
