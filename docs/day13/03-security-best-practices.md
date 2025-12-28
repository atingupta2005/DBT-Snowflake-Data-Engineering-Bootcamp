# 03 — Security Best Practices for Environments

Environment isolation is useless if credentials are handled casually.

Most “security mistakes” in analytics are not clever attacks.

They are:

* someone commits a password
* someone pastes a key in Slack
* someone uses a shared account forever
* someone forgets to rotate a token after a leak

Today we keep this simple and executable.

We will set up dbt so that:

* `profiles.yml` contains **no secrets**
* secrets are provided by environment variables
* DEV and PROD can use different credentials if needed

In the lab, we still connect to the same Snowflake setup.

But we keep the habits you need for production.

## What must never be in Git

If your repository is shared, assume anything committed is public to your coworkers forever.

Do not commit:

* passwords
* private keys
* OAuth tokens
* PATs (personal access tokens)

Also be careful with:

* account identifiers
* usernames
* role names
* warehouse names

Some companies treat these as non-sensitive.

Some companies treat them as sensitive.

When in doubt, use environment variables.

## Why environment variables are the practical baseline

Environment variables solve three problems at once.

### 1) They keep secrets out of code review

If secrets live in code files:

* they show up in diffs
* they get copied into tickets
* they get forwarded in email

If secrets live in environment variables:

* you can rotate them without touching Git
* you reduce accidental exposure

### 2) They support multiple environments cleanly

DEV and PROD often differ.

Examples:

* different Snowflake accounts
* different roles
* different warehouses
* stricter PROD permissions

If the profile references environment variables, you can swap contexts without editing YAML.

### 3) They work in restricted corporate environments

A lot of teams are not allowed to use fancy secret managers locally.

Environment variables work everywhere.

* terminal session
* CI runners (later)
* scheduled jobs (later)

## How dbt reads environment variables

dbt supports `env_var()` in YAML.

This allows a profile field to be pulled from an environment variable.

Example shape:

```yaml
password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
```

If the environment variable is not set, dbt will fail.

That is good.

A silent fallback is how you accidentally connect to the wrong place.

### Optional default values

`env_var()` can take a default.

Example:

```yaml
threads: "{{ env_var('DBT_THREADS', '4') }}"
```

Be careful with defaults.

Defaults are fine for non-sensitive fields.

Do not use defaults for passwords or tokens.

## Profiles discipline that prevents production accidents

This is where teams either operate smoothly or constantly fight fires.

### Rule 1: Default target should be dev

If your default target is `prod`, someone will eventually run:

```bash
dbt run
```

…and write into production.

Keep the default target set to `dev`.

Force `prod` to be explicit.

### Rule 2: Prod role should not have destructive permissions

In real production:

* production roles are locked down
* “drop schema” is usually restricted
* elevated permissions are separate and audited

Even though dbt builds tables, the role still matters.

For this course, we do not change roles.

But keep the real-world principle:

* your prod connection should be least privilege

### Rule 3: Separate credentials per environment when possible

Many teams use:

* a personal user in DEV
* a service user in PROD

That helps auditing.

It also helps with access control:

* your personal user can build in DEV
* your personal user might not have rights in PROD

In the lab, we simulate this with the same user.

But we still structure the profile so you could swap credentials later.

## Practical setup on CentOS Stream 9

We need a repeatable workflow that works on slow corporate laptops.

You will:

* set environment variables in your shell session
* run dbt commands inside your project

### Setting environment variables (session scope)

In bash, you can export variables like this:

```bash
export SNOWFLAKE_ACCOUNT="your_account"
export SNOWFLAKE_USER="your_user"
export SNOWFLAKE_PASSWORD="your_password"
export SNOWFLAKE_ROLE="your_role"
export SNOWFLAKE_WAREHOUSE="your_warehouse"
export SNOWFLAKE_DATABASE="your_database"
```

This sets variables for the current terminal session.

If you close the terminal, they are gone.

That is often good.

It reduces the chance you forget you left credentials lying around.

### Verifying variables are set

Before running dbt, verify you did not typo a variable name.

Example:

```bash
echo "$SNOWFLAKE_USER"
```

Expected outcome:

* prints your username

If it prints nothing, the variable is not set.

Fix that before running dbt.

## Environment variables for DEV vs PROD

There are two common approaches.

### Approach A: Same variables, different schema per target

This is what we will do in this course.

* Same account/user/password
* Different schema in dev vs prod target

This keeps the lab simple.

It still teaches the habit of changing target context.

### Approach B: Different variables per environment

In real production, you might have:

* `SNOWFLAKE_USER_DEV`
* `SNOWFLAKE_USER_PROD`

and map them separately in the profile.

This is more secure.

It is also more work.

We will not force it in this course.

But you should recognize it when you see it.

## Handling sensitive environment variables in practice

You will eventually be in one of these situations:

* someone asks you to paste your `profiles.yml`
* someone asks “what’s your Snowflake password?”
* someone wants help debugging a connection

Here is the safe way to handle it.

### Do not paste raw values into chat

If you need help, paste the shape.

Example:

* show `env_var('...')` references
* do not show the actual exported values

### Redact consistently

If you must share a config snippet, redact like this:

* `<REDACTED>`

Do not use partial redaction.

Partial redaction leaks patterns.

### Rotate after a leak

If a password or token is posted anywhere public to your org:

* assume it is compromised
* rotate it

Do not debate this.

Rotation is cheaper than an incident.

## Common classroom failures (and how to avoid them)

### Failure 1: “dbt says env var is not set.”

Usually means:

* you exported the variable in a different terminal
* you misspelled the variable name
* you forgot to export after opening a new session

Fix:

* run `echo "$VARIABLE"` for the variable name dbt expects

### Failure 2: “It works in dev but not prod.”

Usually means:

* the schema name differs and does not exist
* your role cannot create objects in the prod schema

Fix:

* confirm the schema exists
* confirm permissions
* run `dbt debug --target prod`

### Failure 3: “Someone committed profiles.yml by accident.”

Fix immediately:

* remove the secrets
* rotate credentials
* purge history if your organization requires it

The key lesson:

* deleting the file in the next commit is not enough
* Git history still contains the secret

You do not need to memorize the procedure today.

You need to remember the rule:

* if a secret leaks, rotate it

## What you will do in the lab

In the lab you will:

* ensure your `profiles.yml` uses `env_var()` for sensitive fields
* confirm variables are set in your terminal
* run `dbt debug` in both DEV and PROD targets

Once your targets work cleanly, you will use tags to run selective slices of the project.
