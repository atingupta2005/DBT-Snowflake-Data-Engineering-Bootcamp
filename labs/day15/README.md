# Day 15 Lab — dbt Deployment Pipelines with Jenkins

You will:

* install Jenkins (service on the VM)
* define a Jenkinsfile
* store Snowflake credentials in Jenkins
* run a deployment job that executes: `dbt debug`, `dbt deps`, `dbt build`

---

## Project context (fixed values)

These values are assumed throughout this lab.

* dbt project name: `dbt_olist_project`
* Snowflake database: `OLIST`
* PROD schema: `OLIST_PROD`
* Warehouse: `COMPUTE_WH`
* Python version: `3.11`
* Jenkins HTTP port: `8080`

---

## Step 0 — VM prerequisites

You need a CentOS Stream 9 VM where you can run sudo.

Confirm these are available:

```bash
python3 --version
java -version
```

Expected:

* `python3` is installed
* Java is either missing or older

Jenkins requires Java.

---

## Step 1 — Install dbt on the VM

We will install dbt using a virtual environment.

From your home directory:

```bash
source ~/.venv/bin/activate
dbt --version
```

## Step 2 — Install Jenkins

### Step 2.1 — Install Java 17

```bash
sudo dnf update -y
sudo dnf install -y java-17-openjdk
java -version
```

Expected:

* Java reports a 17.x version

---

### Step 2.2 — Add Jenkins repository

```bash
sudo curl -fsSLo /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

Confirm the repo file exists:

```bash
ls -la /etc/yum.repos.d/jenkins.repo
```

---

### Step 2.3 — Install and start Jenkins

```bash
sudo dnf install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

Expected:

* service status shows **active (running)**

---

### Step 2.5 — Get the initial admin password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the value.

---

### Step 2.6 — Open Jenkins in a browser

From your laptop browser, open:

* `http://localhost:8080`

Login using the initial admin password.

Choose:

* **Install suggested plugins**

Create an admin user.

Stop here when you see the Jenkins dashboard.

---

## Step 3 — Prepare the repository workspace

Jenkins will run your pipeline from a workspace.
---

## Step 4 — Configure Snowflake credentials in Jenkins

Jenkins should never store secrets in the Jenkinsfile.

In Jenkins UI:

1. **Manage Jenkins** → **Credentials**
2. Choose **(global)** domain
3. **Add Credentials**

Create the following credentials.

Use **Secret text** for each.

| ID                   | Value                   |
| -------------------- | ----------------------- |
| `snowflake_account`  | your account identifier |
| `snowflake_user`     | your Snowflake user     |
| `snowflake_password` | your Snowflake password |
| `snowflake_role`     | your Snowflake role     |

Important:

* IDs must match exactly
* do not put quotes in the ID

---

## Step 5 — Create Jenkinsfile

From the repository root (same level as `dbt_project.yml`):

```bash
pwd
ls
```

Create the file:

```bash
touch Jenkinsfile
```

Open `Jenkinsfile` in your editor.

---

## Step 6 — Write the deployment pipeline

Paste the following Jenkinsfile content.

This is a declarative pipeline.

```groovy
pipeline {
    agent any

    environment {
        SNOWFLAKE_ACCOUNT   = credentials('snowflake_account')
        SNOWFLAKE_USER      = credentials('snowflake_user')
        SNOWFLAKE_PASSWORD  = credentials('snowflake_password')
        SNOWFLAKE_ROLE      = credentials('snowflake_role')
        SNOWFLAKE_WAREHOUSE = 'COMPUTE_WH'
        SNOWFLAKE_DATABASE  = 'OLIST'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set up Python') {
            steps {
                sh '''
                python3 -m venv .venv
                source .venv/bin/activate
                pip install --upgrade pip
                pip install "dbt-core==1.9.8" "dbt-snowflake==1.9.4"
                dbt --version
                '''
            }
        }

        stage('Write dbt profile') {
            steps {
                sh '''
                mkdir -p ~/.dbt
                cat > ~/.dbt/profiles.yml <<'YAML'
                dbt_olist_project:
                  target: prod
                  outputs:
                    prod:
                      type: snowflake
                      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
                      user: "{{ env_var('SNOWFLAKE_USER') }}"
                      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
                      role: "{{ env_var('SNOWFLAKE_ROLE') }}"
                      warehouse: "COMPUTE_WH"
                      database: "OLIST"
                      schema: "OLIST_PROD"
                      threads: 4
                YAML
                '''
            }
        }

        stage('Run dbt build') {
            steps {
                sh '''
                source .venv/bin/activate
                dbt debug
                dbt deps
                dbt build
                '''
            }
        }
    }
}
```

What this pipeline does:

* checks out your repo
* creates a venv in the Jenkins workspace
* installs dbt
* writes `~/.dbt/profiles.yml` at runtime
* runs `dbt debug`, `dbt deps`, `dbt build`

---

## Step 7 — Create a Jenkins Pipeline job

In Jenkins UI:

1. **New Item**
2. Name: `dbt-olist-deploy`
3. Type: **Pipeline**
4. Click **OK**

In the job configuration:

* Pipeline definition: **Pipeline script from SCM**
* SCM: **Git**
* Repository URL: your repo URL
* Script Path: `Jenkinsfile`

Save.

---

## Step 8 — Run the pipeline

Click **Build Now**.

Open the build.

Open **Console Output**.

Expected sequence:

* Checkout
* Python setup
* dbt version prints
* dbt debug passes
* dbt build completes

If `dbt debug` fails:

* credentials IDs are wrong
* credentials values are wrong
* Snowflake role/warehouse/database access is missing

Fix the root cause and run again.

---
