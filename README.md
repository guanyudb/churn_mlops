# Two-Gate MLOps on Databricks — GitHub Actions edition

Promote one churn model **dev → staging → prod** through GitHub + Databricks Asset Bundles,
with **two human gates**:

- **Gate 1 — code review on a PR to `main`** (branch protection: required `churn-ci` check +
  review). No unreviewed, unverified code merges.
- **Gate 2 — model approval in Unity Catalog.** Even released code serves nothing in prod until a
  human clicks **Approve** on the new model version.

**Deploy-code** (one codebase, per-stage *schema*), **shared data** (prod owns the feature/label
tables; dev + staging *read* prod), MLflow 3 **deployment jobs** for the UC gate, service-principal
CI/CD.

```
Databricks Git folder: branch feature/** -> edit -> commit & push
  └─(auto) .github/workflows/auto-pr.yml ──────> opens PR feature/** -> main
  └─(auto) churn-ci on the PR: bundle validate + deploy staging   ┐ GATE 1
  YOU: review the diff -> Merge ..................................┘ (required check must be green)
  └─(auto) churn-cd — one run on push to main (jobs chained via needs:):
       deploy-staging : deploy + train -> register model  (the E2E gate)
       promote        : fast-forward `release` to this commit (audit pointer, no trigger)
       deploy-prod    : deploy + train -> register NEW prod version -> parks at Gate 2
  YOU: UC model UI -> new version -> Approve .....................  GATE 2
  └─(auto) deployment job: Approval_Check ✅ -> @Champion flips -> serving endpoint rolls
```

> **This repo carries the human-gate patches in code** (`04a` validate-only, `06` budget-policy
> resilience) — see [What's patched](#whats-patched-vs-stock-dbdemos). Stock dbdemos auto-approves
> and has no budget catch; do not revert those.

---

## Repo layout
```
.github/workflows/
  auto-pr.yml     # push feature/** -> open PR to main
  churn-ci.yml    # PR to main -> validate + deploy staging (Gate 1 required check)
  churn-cd.yml    # push main -> deploy-staging -> promote -> deploy-prod (chained jobs)
churn-mlops/      # the Databricks Asset Bundle
  databricks.yml            # targets dev/staging/prod (per-stage schema)
  resources/churn_jobs.yml  # feature-eng, training(+register), batch-inference jobs
  resources/deployment_job.yml  # MLflow 3 deployment job = Gate 2
  src/_resources/00-setup.py    # shared-data setup (prod owns data)
  src/02-mlops-advanced/        # dbdemos advanced churn notebooks (01..08) + patches
```

---

## Setup (once per Databricks workspace)

### 1. Fill in your values
| Value | Where |
|---|---|
| Workspace URL | `churn-mlops/databricks.yml` (`host:` on each target) |
| Catalog (`catalog_sbx_*`) | `databricks.yml` var **and** hardcoded in `src/_resources/00-setup.py` |
| CI service principal app id | `databricks.yml` var `service_principal_app_id` |

Schemas (one catalog): `dbdemos_mlops_dev` / `_staging` / `_prod`. **prod owns the data**; dev &
staging are model-only and read prod. (FEVM Azure sandboxes use a Default-Storage metastore — you
**cannot `CREATE CATALOG`**; create only schemas in the provided `catalog_sbx_*`.)

### 2. GitHub Actions secrets (Settings → Secrets and variables → Actions)
The service principal's OAuth (M2M) credentials — used by every workflow for Databricks auth:
```
DATABRICKS_HOST            = https://<workspace>.azuredatabricks.net
DATABRICKS_CLIENT_ID       = <SP application/client id>
DATABRICKS_CLIENT_SECRET   = <SP OAuth secret>
```

### 3. Branch protection on `main` = Gate 1
Settings → Branches → add rule for `main`:
- ✅ **Require a pull request before merging** (+ *Require approvals: 1* — narrated for a solo demo;
  see note)
- ✅ **Require status checks to pass** → select **`churn-ci`** (the hard, automated gate)
- Create branches **`main`** (protected trunk) and **`release`** (prod audit pointer — no protection,
  no workflow trigger).

> **Solo-presenter note:** github.com has **no "allow self-approval"** — an author can't approve their
> own PR. For a one-person demo, rely on the **required `churn-ci` check** as the enforced gate and
> **narrate** the review/approval (what a team adds). To enforce approval for real, add a second
> reviewer identity. (This differs from Azure DevOps, which *does* allow self-approval.)

### 4. Databricks side (see the full playbook)
Provision workspace → create the 3 schemas → apply notebook patches (already in this repo) →
`bundle deploy` per target → **the CI service-principal permission chain** → link the deployment job
to the UC model (Gate 2). The complete, ordered steps + the permission chain + every known issue are
in **`MLOPS_DEMO_PLAYBOOK.md`** (the platform-agnostic playbook; this README is the GitHub-specific
layer).

---

## Daily loop (the demo)
1. In the Databricks **Git folder** for this repo: branch `feature/<x>`, edit (e.g. a hyperparameter
   or a job tag), **Commit & Push**.
2. `auto-pr` opens the PR. `churn-ci` runs (validate + staging deploy). **Review → Merge** = Gate 1.
3. `churn-cd` runs one chained pipeline: staging deploy+train+register → fast-forward `release` →
   prod deploy+train+register → new version **parks at "Approval needed"**.
4. UC model UI → the new version → **Approve** = Gate 2 → `@Champion` flips → endpoint rolls.

3 human actions total: **push · merge the PR · approve the model.**

---

## What's patched vs stock dbdemos
- **`04a_challenger_validation.py`** — *validate-only.* Stock auto-sets `@Champion` +
  `Approval_Check=Approved`, which removes the human gate. Patched to record metrics + set
  `@Challenger` only; `04b` promotes **only after** a human sets `Approval_Check=Approved`.
- **`06_serve_features_and_model.py`** — *budget-policy resilience.* Wraps `fe.publish_table` and the
  serving-endpoint update so a classic-sandbox `UseBudgetPolicy` error is non-fatal (job stays green;
  `@Champion` still flips). Remove the catch once an account admin grants the policy → endpoint rolls
  for real.
- **`02_model_training_hpo_optuna.py`** — serving-compatible pins (`pandas>=2.1,<3`, cloudpickle),
  fresh-run logging + disabled debug studies (avoids MLflow's 1000-metric-per-model quota that
  looked like "flaky training").
- **`00-setup.py`** — shared-data: hardcoded catalog, `data_source_schema` gate so only prod builds
  data, per-stage experiment name.
- **`03b`** — best-effort metadata/alias updates (CI SP is not the model owner).

## Notes
- `release` is an **audit pointer** (what prod runs), moved by the `promote` job — not a trigger.
- CI installs **Terraform 1.5.7** and sets `DATABRICKS_TF_VERSION=1.5.7` (works around a CLI TF
  key-expiry issue).
- Full rebuild steps, the permission chain, and 12 known-issues/war-stories: **`MLOPS_DEMO_PLAYBOOK.md`**.
