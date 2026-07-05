# GitHub Actions Enterprise CI/CD Pipeline

A production-style CI/CD pipeline built with **GitHub Actions reusable workflows** for a Python Flask application deployed to **Amazon ECS via ECR**.

> Built as part of **Aviz Academy — Batch 7 DevSecOps Track**

--- 

## Application

| Item | Detail |
|------|--------|
| Framework | Python 3.12 / Flask |
| Container base | `python:3.12-slim` |
| Port | `5000` |
| Health endpoint | `GET /health` |
| Run command | `gunicorn --bind 0.0.0.0:5000 app:app` |

---

## Pipeline Architecture

```
Push to main / PR to main / Manual trigger
         │
         ▼
┌─────────────────────────────────────────────┐
│  STAGE 1 — Security & Quality  (parallel)   │
│  ├─ Secret Scan       (gitleaks)             │
│  ├─ Code Quality      (flake8+bandit+audit)  │
│  └─ SonarCloud        (coverage+smells)      │
└─────────────────┬───────────────────────────┘
                  │  all three must pass
                  ▼
┌─────────────────────────────────────────────┐
│  STAGE 2 — Unit & Integration Tests         │
│  └─ pytest --cov  (hard gate, no || true)   │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  STAGE 3 — Build & Publish                  │
│  └─ hadolint → docker build → push to ECR   │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  STAGE 4 — Image Vulnerability Scan         │
│  └─ Trivy (fails on CRITICAL/HIGH CVEs)     │
└─────────────────┬───────────────────────────┘
                  │  skipped on PRs
                  ▼
┌─────────────────────────────────────────────┐
│  STAGE 5 — Deploy to Denvironment (ECS)     │
│  └─ OIDC → ECR login → update task def      │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  STAGE 6 — Smoke Test                       │
│  └─ curl /health  (5 retries, 15s delay)    │
└─────────────────┬───────────────────────────┘
                  │  always runs
                  ▼
┌─────────────────────────────────────────────┐
│  STAGE 7 — Slack Notification               │
│  └─ green / red / yellow based on result    │
└─────────────────────────────────────────────┘

         ⏳  3-week staging observation period

┌─────────────────────────────────────────────┐
│  PRODUCTION DEPLOY  (manual trigger only)   │
│  └─ preflight checks → deploy to ECS prod   │
└─────────────────────────────────────────────┘
```

---

## Workflow Files

### Caller Workflows (entry points)

| File | Trigger | Purpose |
|------|---------|---------|
| `00-caller-enterprise-pipeline.yml` | push/PR to `main`, manual | Full 7-stage pipeline |
| `00-caller-demo-pipeline.yml` | push to `main`, manual | Simplified 3-stage demo |
| `00-caller-prod-deploy.yml` | **Manual only** | Production deploy after 3-week staging gate |

### Reusable Workflows (building blocks)

| File | Stage | Tools |
|------|-------|-------|
| `reusable/01-rw-secret-scan.yml` | S1 | gitleaks |
| `reusable/02-rw-code-quality.yml` | S1 | flake8, bandit, pip-audit |
| `reusable/03-rw-sonar-scan.yml` | S1 | SonarCloud |
| `reusable/07-rw-unit-tests.yml` | S2 | pytest, pytest-cov |
| `reusable/04-rw-docker-build-push.yml` | S3 | hadolint, docker buildx, Amazon ECR |
| `reusable/05-rw-image-scan.yml` | S4 | Trivy |
| `reusable/06-rw-deploy-ecs.yml` | S5 / Prod | Amazon ECS (OIDC) |
| `reusable/08-rw-smoke-test.yml` | S6 | curl |
| `reusable/09-rw-notify-slack.yml` | S7 | Slack Incoming Webhook |

---

## Prerequisites

### 1. AWS — OIDC Trust (one-time setup)

The pipeline uses **OIDC** to authenticate to AWS — no long-lived access keys stored anywhere.

**Create the IAM role trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPO_NAME:*"
        }
      }
    }
  ]
}
```

**Permissions needed on the IAM role:**
- `AmazonEC2ContainerRegistryPowerUser` — push images to ECR
- `AmazonECS_FullAccess` — update task definitions and services

### 2. Amazon ECR

Create an ECR repository:

```bash
aws ecr create-repository \
  --repository-name github-actions-app \
  --region ap-south-1
```

### 3. Amazon ECS

You need an existing ECS cluster with:
- A **Fargate service** running the app
- A **task definition** with the container name matching `ECS_CONTAINER_NAME`
- A **load balancer** pointing to the service (its DNS name becomes `STAGING_URL`)

### 4. SonarCloud

1. Log in at [sonarcloud.io](https://sonarcloud.io) and import this repository
2. Go to **My Account → Security → Generate Token**
3. Copy the token — this is your `SONAR_TOKEN`
4. Verify `sonar-project.properties` has the correct values:
   ```properties
   sonar.projectKey=avizway1_awar07-gha-demo
   sonar.organization=avizway1
   ```

### 5. Slack Incoming Webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From Scratch**
2. Enable **Incoming Webhooks** and click **Add New Webhook to Workspace**
3. Choose the channel for pipeline notifications
4. Copy the webhook URL — this is your `SLACK_WEBHOOK_URL`

---

## Secrets and Variables Setup

Go to: **Repo → Settings → Secrets and Variables → Actions**

### Secrets (9 total)

| Secret | Description | Example |
|--------|-------------|---------|
| `AWS_ROLE_ARN` | Full ARN of the OIDC IAM role | `arn:aws:iam::123456789012:role/GHA-Role` |
| `ECR_REPO_NAME` | ECR repository name | `github-actions-app` |
| `ECS_CLUSTER_NAME` | ECS cluster name | `my-cluster` |
| `ECS_SERVICE_NAME` | ECS service name | `my-app-service` |
| `ECS_TASK_DEF_NAME` | ECS task definition family name | `my-app-task` |
| `ECS_CONTAINER_NAME` | Container name inside the task definition | `my-app-container` |
| `SONAR_TOKEN` | SonarCloud project token | `sqp_xxxxxxxxxxxx` |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook URL | `https://hooks.slack.com/services/...` |
| `GITLEAKS_LICENSE` | Gitleaks license key — **orgs only, skip for personal accounts** | |

### Variables (1 total)

| Variable | Description | Example |
|----------|-------------|---------|
| `STAGING_URL` | Public URL of your staging ECS service | `https://staging.myapp.com` |

### Auto-provided (no setup needed)

| Secret | Source |
|--------|--------|
| `GITHUB_TOKEN` | Injected automatically by GitHub into every workflow run |

---

## GitHub Environments Setup

Go to: **Repo → Settings → Environments**

### `staging`
- No protection rules required
- Optionally restrict deployments to the `main` branch only

### `production`
- Add **Required reviewers** — the person who approves releases
- Set deployment branch rule: `main` only
- This creates the manual approval gate before any production deploy runs

---

## Pipeline Behaviour by Trigger

| Trigger | S1 | S2 | S3 | S4 | S5 Deploy | S6 Smoke | S7 Notify |
|---------|:--:|:--:|:--:|:--:|:---------:|:--------:|:---------:|
| Push to `main` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| PR to `main` | ✅ | ✅ | ✅ | ✅ | ⏭ | ⏭ | ✅ |
| Manual `skip-deploy=false` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Manual `skip-deploy=true` | ✅ | ✅ | ✅ | ✅ | ⏭ | ⏭ | ✅ |

> S7 (Slack notification) always runs regardless of pipeline outcome.

---

## Production Deployment Process

Production deployment is intentionally **not automatic**. It uses a separate workflow with a mandatory checklist.

### Minimum criteria before triggering

1. The same image SHA has been running in staging for **at least 3 weeks**
2. No P1 or P2 bugs reported from staging during the observation period
3. Written sign-off received from QA and the release owner

### How to trigger

1. Go to **Actions → Production Deploy (Manual Gate) → Run workflow**
2. Fill in the required inputs:

   | Input | What to enter |
   |-------|--------------|
   | `staging-sha` | Git SHA from the last successful staging deploy |
   | `release-version` | Version label, e.g. `v1.4.2` |
   | `confirmed` | Check the box to confirm 3-week observation |

3. The `preflight` job runs first — it blocks the deploy if the confirmation box is unchecked
4. The `production` GitHub Environment then pauses and waits for a required reviewer to approve
5. On approval, ECS is updated with the new task definition and rolls out

---

## Image Tagging Strategy

Every push to `main` produces two ECR tags:

| Tag | Value | Used by |
|-----|-------|---------|
| `:latest` | Always the newest image | Trivy scan |
| `:<git-sha>` | Immutable, tied to the exact commit | ECS task definition |

ECS always deploys the `:sha` tag — never `:latest` — so every deployment is fully traceable to a specific commit.

---

## Suppressing Known Trivy Findings

Add CVE IDs to `.trivyignore` at the root of the repo:

```
# .trivyignore
CVE-2023-XXXXX
CVE-2024-XXXXX
```

---

## Running Locally

```bash
# Install dependencies
pip install -r requirements.txt

# Run the app
python app.py
# Open http://localhost:5000

# Run tests with coverage
pytest --cov=. --cov-report=term-missing

# Lint
flake8 app.py

# SAST scan
bandit -r app.py

# Dependency CVE check
pip install pip-audit
pip-audit -r requirements.txt
```

---

## Repository Structure

```
.
├── app.py                          # Flask application
├── Dockerfile                      # Container definition
├── requirements.txt                # Python dependencies
├── sonar-project.properties        # SonarCloud configuration
├── .trivyignore                    # Trivy CVE suppressions
└── .github/
    └── workflows/
        ├── 00-caller-enterprise-pipeline.yml   # Main 7-stage CI/CD pipeline
        ├── 00-caller-demo-pipeline.yml          # Simplified demo pipeline
        ├── 00-caller-prod-deploy.yml            # Manual production deploy
        └── reusable/
            ├── 01-rw-secret-scan.yml            # S1 — gitleaks
            ├── 02-rw-code-quality.yml           # S1 — flake8 + bandit + pip-audit
            ├── 03-rw-sonar-scan.yml             # S1 — SonarCloud
            ├── 07-rw-unit-tests.yml             # S2 — pytest
            ├── 04-rw-docker-build-push.yml      # S3 — build + push to ECR
            ├── 05-rw-image-scan.yml             # S4 — Trivy
            ├── 06-rw-deploy-ecs.yml             # S5/Prod — ECS deploy
            ├── 08-rw-smoke-test.yml             # S6 — health check
            └── 09-rw-notify-slack.yml           # S7 — Slack notification
```
