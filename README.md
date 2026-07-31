# Churn MLOps on Databricks — GitHub Actions

A reference implementation of production MLOps on Databricks, driven from GitHub. One churn model
is promoted **dev → staging → prod** through pull requests and GitHub Actions, governed by **three
human approval gates**:

1. **Code review** — a pull request into `main` must be reviewed and approved before it merges.
2. **Pre-production release** — after the change is validated in staging, a human approves the
   promotion to production.
3. **Model approval** — in Unity Catalog, a human approves the trained model before it is served.

Built on the **deploy-code** pattern: one codebase, promoted across environments; each stage is an
isolated schema. Data is owned by production and read by dev/staging (no copies). Model promotion
and serving are governed by an MLflow 3 **deployment job**. All CI/CD runs as a Databricks service
principal.

```
 branch feature/*  →  edit  →  open a Pull Request into main
        │
   ┌────▼─────────────────────────────────────────────────────┐
   │ GATE 1 — Code review                                      │
   │   churn-ci validates the change (deploys to staging);     │
   │   a reviewer approves; the PR merges.                      │
   └────┬──────────────────────────────────────────────────────┘
        │ merge to main triggers churn-cd (one run):
        │   • Staging  — deploy, train, register a model (end-to-end check)
        │   • Promote  — advance the `release` pointer
   ┌────▼─────────────────────────────────────────────────────┐
   │ GATE 2 — Pre-production release                           │
   │   the pipeline pauses; a human approves promotion to prod │
   └────┬──────────────────────────────────────────────────────┘
        │   • Prod     — deploy, train, register a new prod model version
        │                (parks awaiting model approval)
   ┌────▼─────────────────────────────────────────────────────┐
   │ GATE 3 — Model approval (Unity Catalog)                   │
   │   a data scientist approves the version → it becomes       │
   │   Champion → the serving endpoint updates                 │
   └───────────────────────────────────────────────────────────┘
```

---

## Repository layout
```
.github/workflows/
  churn-ci.yml    # on PR to main: validate + deploy to staging (Gate 1 status check)
  churn-cd.yml    # on merge to main: staging → (Gate 2 approval) → prod
churn-mlops/                       # the Databricks Asset Bundle
  databricks.yml                   # dev / staging / prod targets (one schema per stage)
  resources/churn_jobs.yml         # feature engineering, training (+register), batch inference
  resources/deployment_job.yml     # MLflow 3 deployment job (the model-approval gate)
  src/_resources/00-setup.py       # shared-data setup (production owns the data)
  src/02-mlops-advanced/           # the churn ML notebooks (feature eng, training, validation,
                                   #   approval, serving)
```

---

## The daily loop

1. **Branch & edit.** From the Databricks Git folder (or locally), create a `feature/*` branch and
   make your change — e.g. a model hyperparameter or a job configuration.
2. **Open a pull request** into `main`.
3. **Gate 1 — review & merge.** GitHub Actions runs `churn-ci`, which validates the bundle and
   deploys it to staging. Once the check is green and a reviewer approves, merge the PR.
4. **Automated promotion.** Merging triggers `churn-cd`: it deploys to staging, trains and registers
   a model there (the end-to-end check), then advances the `release` pointer.
5. **Gate 2 — approve the release.** The pipeline pauses before production and waits for a human to
   approve the deployment in GitHub. On approval, it deploys to production, trains, and registers a
   new production model version — which parks awaiting model approval.
6. **Gate 3 — approve the model.** In Unity Catalog, open the new model version and click **Approve**.
   It becomes `@Champion` and the serving endpoint updates to it.

**Three human actions:** approve & merge the PR · approve the pre-production release · approve the model.

---

## CI/CD

Two GitHub Actions workflows under `.github/workflows/`. Each installs the Databricks CLI and
authenticates as the CI service principal (OAuth machine-to-machine, from the repo's Actions secrets).

| Workflow | Trigger | What it does |
|---|---|---|
| `churn-ci.yml` | Pull request into `main` | Validates the bundle and deploys it to **staging** (`bundle validate` + `bundle deploy -t staging`). This is the required status check for Gate 1 — code can't merge unless it is deployable. |
| `churn-cd.yml` | Merge to `main` | One run, three stages: **Staging** (deploy → train → register a model = the end-to-end check) → **Promote** (advance the `release` pointer) → **Prod** (deploy → train → register a new production model version, which parks for model approval). The **Prod** stage is protected by a GitHub Environment approval — **Gate 2**. |

**Where the gates live**
- **Gate 1 — the pull request.** Branch protection on `main` requires the `churn-ci` check to pass
  **and** a reviewer to approve before the PR can merge.
- **Gate 2 — the pre-production release.** The `churn-cd` prod stage targets a protected GitHub
  Environment (`prod`); the run pauses and a designated reviewer approves the promotion.
- **Gate 3 — the model.** The production model version parks in Unity Catalog until a human clicks
  **Approve**; only then does the deployment job promote it to `@Champion` and update the serving
  endpoint.

**Design notes**
- **Test what you ship.** The exact commit that passes the staging end-to-end check is the commit
  deployed to production, in the same pipeline run.
- **`release` is an audit pointer**, not a trigger — it records what production is running. Production
  is deployed within the `churn-cd` run itself, so there is a single pipeline and no duplicate runs.
- **Environment separation.** dev, staging, and prod are isolated schemas in one catalog; production
  owns the data and dev/staging read it, so there are no data copies to drift.

---

## Setup (once per workspace)

### 1. Workspace values
Set these to match your environment:

| Value | Where |
|---|---|
| Workspace URL | `churn-mlops/databricks.yml` — `host:` on each target |
| Catalog | `databricks.yml` variable **and** `src/_resources/00-setup.py` |
| CI service principal (app id) | `databricks.yml` variable `service_principal_app_id` |

Create three schemas in the catalog: `dbdemos_mlops_dev`, `dbdemos_mlops_staging`,
`dbdemos_mlops_prod`. Production owns the data; dev and staging read from it.

### 2. GitHub Actions secrets
`Settings → Secrets and variables → Actions` — the service principal's OAuth credentials, used by
every workflow:
```
DATABRICKS_HOST            = https://<workspace-host>
DATABRICKS_CLIENT_ID       = <service principal application id>
DATABRICKS_CLIENT_SECRET   = <service principal OAuth secret>
```

### 3. Branch protection on `main` (Gate 1)
`Settings → Branches → add rule for main`:
- ✅ **Require a pull request before merging** — require **1** approving review
- ✅ **Require status checks to pass** — select the **`validate`** check *(the check is named after
  the job, `validate`)*
- Create branches **`main`** (the protected trunk) and **`release`** (the production pointer).

### 4. Pre-production release Environment (Gate 2)
`Settings → Environments → New environment → prod` → add a **Required reviewer**. The `churn-cd`
production stage targets this environment, so promotion to prod pauses for that reviewer's approval.

### 5. Databricks side
Deploy the bundle to each target, grant the CI service principal access to the catalog/schemas and
model, and link the MLflow deployment job to the production model (this makes new versions park for
model approval — Gate 3). Full step-by-step setup, including the exact permission grants, is in
**`MLOPS_DEMO_PLAYBOOK.md`**.

---

## Notes

- **Model governance is deliberate.** Training in staging and prod **registers** a model version;
  production versions do not serve until approved (Gate 3). Approval promotes the version to
  `@Champion`, retires the prior `@Challenger` marker, and updates the serving endpoint.
- **Solo demos.** GitHub does not allow a pull-request author to approve their own PR. With a second
  reviewer, Gate 1 is satisfied normally; running solo, a repository admin can merge directly. The
  automated `validate` check is always enforced either way.
- **Full rebuild guide.** To stand this up in a fresh workspace end to end — provisioning, the
  service-principal permission chain, and the deployment-job wiring — see **`MLOPS_DEMO_PLAYBOOK.md`**.
