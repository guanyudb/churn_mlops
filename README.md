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

### 3. Repo Actions permission (REQUIRED — `auto-pr` fails without it)
Settings → Actions → General → **Workflow permissions**:
- ✅ **Read and write permissions**
- ✅ **Allow GitHub Actions to create and approve pull requests**

Without this, `auto-pr` errors `GitHub Actions is not permitted to create or approve pull requests`.
It's a **repo setting — not a secret, not branch protection** — so it's easy to miss. Via API:
```bash
gh api -X PUT repos/<OWNER>/<REPO>/actions/permissions/workflow \
  -F default_workflow_permissions=write -F can_approve_pull_request_reviews=true
```
> A PR opened *by the auto-pr bot* may have its first `churn-ci` run land in **`action_required`** —
> approve it once in the PR's Checks tab (GitHub's first-run gate for bot-authored PRs).

### 4. Branch protection on `main` = Gate 1
Settings → Branches → add rule for `main`:
- ✅ **Require a pull request before merging** (+ *Require approvals: 1* — narrated for a solo demo;
  see note)
- ✅ **Require status checks to pass** → select the **`validate`** check. *(The required-check
  context is the **job** name `validate`, not the workflow name `churn-ci`.)*
- Create branches **`main`** (protected trunk) and **`release`** (prod audit pointer — no protection,
  no workflow trigger).

> **Solo-presenter note:** github.com has **no "allow self-approval"** — an author can't approve their
> own PR. For a one-person demo, rely on the **required `validate` check** as the enforced gate and
> **narrate** the review/approval (what a team adds), or merge via **admin** (`gh pr merge --admin`,
> works only with `enforce_admins=false`). To enforce approval for real, add a second reviewer
> identity. (This differs from Azure DevOps, which *does* allow self-approval.)

### 5. Databricks side (see the full playbook)
Provision workspace → create the 3 schemas → apply notebook patches (already in this repo) →
`bundle deploy` per target → **the CI service-principal permission chain** → link the deployment job
to the UC model (Gate 2). The complete, ordered steps + the permission chain + every known issue are
in **`MLOPS_DEMO_PLAYBOOK.md`** (the platform-agnostic playbook; this README is the GitHub-specific
layer).

---

## CI/CD

The repo ships **three workflows** under `.github/workflows/`; each installs the Databricks CLI +
Terraform 1.5.7 and authenticates as the CI service principal (M2M OAuth from the repo's Actions
secrets). Together they implement the two-gate flow — **push a `feature/**` branch, and the only
human steps left are Merge (Gate 1) and Approve the model (Gate 2).**

| Workflow | Trigger | What it does |
|---|---|---|
| `auto-pr.yml` | Push to `feature/**` | Opens a PR from the feature branch into `main` (skips if one is already open), so the dev's only manual step is Merge. Uses `GITHUB_TOKEN`. |
| `churn-ci.yml` | PR opened/synced against `main` | **Gate 1 (automated half).** The `validate` job runs `databricks bundle validate -t staging` + `bundle deploy -t staging` — proves the code is deployable before it can merge. This is the **required status check** in branch protection; combined with the reviewer/Merge, no unverified code reaches `main`. |
| `churn-cd.yml` | Push to `main` (i.e. a PR merged) | One chained multi-job run: **`deploy-staging`** (`bundle deploy -t staging` → `bundle run churn_training` = train **and register** a model; batch inference best-effort = the E2E gate) → **`promote`** (fast-forward `release` to this commit — `release` is an audit pointer of what prod runs, deliberately **not** a trigger) → **`deploy-prod`** (`bundle deploy -t prod` → `bundle run churn_training` registers a NEW prod version, which auto-triggers the deployment job and **parks at the UC approval gate**). |

**Where the gates live:**
- **Gate 1 — the PR page** (`.../pulls`): merge to `main` is blocked by branch protection until the
  `churn-ci` **`validate`** check is green **and** a reviewer approves. The *automated* half is
  churn-ci; the *human* half is the review + Merge.
- **Gate 2 — the UC model UI:** `churn-cd`'s prod job registers a new version that parks at
  "Approval needed"; the deployment job's `Approval_Check` task raises until a human clicks
  **Approve** (sets tag `Approval_Check=Approved`) — only then does `@Champion` flip and the serving
  endpoint roll.

**A few details worth knowing (say these while CI runs):**
- **`release` is an audit pointer, not a trigger.** The `promote` job fast-forwards it as a record of
  "what prod runs"; the prod deploy happens in the *same* churn-cd run (job `deploy-prod`), so there's
  no second pipeline and no duplicate runs.
- **Test what you ship.** The commit that passes the staging E2E gate is the exact commit deployed to
  prod, in one run.
- **First churn-ci run on a bot-opened PR** may show **"action_required"** — approve the workflow run
  once in the PR's Checks tab (GitHub gates first runs on PRs opened by a workflow). One-time per PR.
- **Solo presenter:** github.com has no self-approval; either merge via **admin** (`gh pr merge
  --admin`, works because `enforce_admins=false`) or add a second reviewer. The required `validate`
  check is always genuinely enforced.

> **vs. the reference `databricks-solutions/microbricks`** (which ships six workflows incl. per-PR
> preview environments, path-scoped test matrices, and nightly Lakebase-branch GC): this repo keeps
> the CI/CD deliberately lean because it's a *model* pipeline, not a fleet of apps. The extra thing it
> has that microbricks doesn't is the **second human gate at the model** (UC Approve) — MLOps-specific
> and stronger than a purely tag-triggered prod deploy. Ideas worth borrowing later: per-PR preview
> schemas, a path-scoped matrix, and a nightly cleanup cron.

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
