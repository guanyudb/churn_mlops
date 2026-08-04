# Setup Guide

How to stand this project up in your own Databricks workspace and GitHub repository. The demo flow
itself (the three approval gates) is described in the [README](README.md).

---

## Prerequisites

- A **Databricks workspace** with Unity Catalog, and a catalog you can create schemas in.
- **Databricks CLI** v0.240+ — `databricks auth login --host <workspace-url> --profile <name>`
- A **service principal** for CI/CD, with an OAuth (M2M) client secret.
- **Terraform 1.5.7** available to the CLI (see [Troubleshooting](#troubleshooting) — the workflows
  install it automatically in CI).
- Repository admin rights on your GitHub fork/copy (to set secrets, branch protection, environments).

---

## 1. Databricks: catalog, schemas, and data

Create three schemas in your catalog — one per stage:

```bash
CATALOG=<your_catalog>
for S in dbdemos_mlops_dev dbdemos_mlops_staging dbdemos_mlops_prod; do
  databricks schemas create $S $CATALOG -p <profile>
done
```

**Data ownership:** production owns the feature and label tables; dev and staging read from
production rather than keeping their own copies. `src/_resources/00-setup.py` enforces this — it only
creates the source tables when the target schema is the production one.

## 2. Point the bundle at your environment

Replace the placeholders in these two files:

| Placeholder | File | Value |
|---|---|---|
| `<CATALOG>` | `churn-mlops/databricks.yml` **and** `churn-mlops/src/_resources/00-setup.py` | your catalog |
| `<CI_SERVICE_PRINCIPAL_APP_ID>` | `churn-mlops/databricks.yml` | the service principal's application id |

**Workspace URL is not in the bundle.** The Databricks CLI resolves it from the environment — the
`DATABRICKS_HOST` secret in CI, or your CLI profile locally — so you don't hardcode a workspace and
the project stays portable across environments.

> Also search the notebooks for any fully-qualified table or function references that need your
> catalog: `grep -rn "<CATALOG>" churn-mlops/src/`

## 3. Deploy the bundle

```bash
cd churn-mlops
export DATABRICKS_TF_VERSION=1.5.7
databricks bundle validate -t dev -p <profile>
databricks bundle deploy   -t dev -p <profile>
databricks bundle deploy   -t staging -p <profile>
databricks bundle deploy   -t prod -p <profile>
```

Then seed production data once (this creates the shared feature/label tables and the
`avg_price_increase` function):

```bash
databricks bundle run churn_feature_engineering -t prod -p <profile>
```

Verify: the production schema has the feature and label tables; dev and staging have none.

## 4. Grant the CI service principal

Each grant below unblocks a specific step in the pipeline — apply them all.

```bash
SP=<CI_SERVICE_PRINCIPAL_APP_ID>
CATALOG=<your_catalog>

# Catalog: USE_CATALOG, plus CREATE_SCHEMA (the setup notebook runs CREATE DATABASE IF NOT EXISTS)
databricks grants update catalog $CATALOG -p <profile> \
  --json "{\"changes\":[{\"principal\":\"$SP\",\"add\":[\"USE_CATALOG\",\"CREATE_SCHEMA\"]}]}"

# Schemas: read shared data, write models and tables
for S in dbdemos_mlops_dev dbdemos_mlops_staging dbdemos_mlops_prod; do
  databricks grants update schema $CATALOG.$S -p <profile> \
    --json "{\"changes\":[{\"principal\":\"$SP\",\"add\":[\"USE_SCHEMA\",\"SELECT\",\"EXECUTE\",\"CREATE_TABLE\",\"CREATE_FUNCTION\",\"CREATE_MODEL\",\"MODIFY\"]}]}"
done
```

**After the model exists** (i.e. after the first training run registers a version), grant model-level
rights — schema `CREATE_MODEL` alone does not allow adding versions to an existing model, and the
validation/approval notebooks need `MANAGE` to set tags and aliases:

```bash
for S in dbdemos_mlops_staging dbdemos_mlops_prod; do
  databricks grants update function $CATALOG.$S.advanced_mlops_churn -p <profile> \
    --json "{\"changes\":[{\"principal\":\"$SP\",\"add\":[\"CREATE_MODEL_VERSION\",\"MANAGE\"]}]}"
done
```

Two more, easy to miss:
- **Experiment access** — if you run the first training as *yourself*, the MLflow experiment is
  created under your user namespace and the service principal cannot read it. Either run the seed as
  the service principal, or grant it `CAN_MANAGE` on the experiment.
- **Job ownership** — the staging/prod jobs should be owned (or manageable) by the service principal.
  If you deployed them as a user first, grant the SP `CAN_MANAGE` on those jobs, or redeploy as the SP.

## 5. Wire the model approval gate

Training registers a model version; the **deployment job** governs whether it is served. Link the job
to the registered model once:

```python
from mlflow.tracking import MlflowClient
MlflowClient(registry_uri="databricks-uc").update_registered_model(
    "<CATALOG>.dbdemos_mlops_prod.advanced_mlops_churn",
    deployment_job_id="<the churn_deployment_job id>",   # databricks jobs list
)
```

Confirm the link with
`GET /api/2.0/mlflow/unity-catalog/registered-models/get?name=<model>` — you want
`deployment_job_state: CONNECTED`.

## 6. GitHub configuration

1. **Actions secrets** (`Settings → Secrets and variables → Actions`):
   ```
   DATABRICKS_HOST            = <workspace url>
   DATABRICKS_CLIENT_ID       = <service principal application id>
   DATABRICKS_CLIENT_SECRET   = <service principal OAuth secret>
   ```
2. **Branch protection on `main`** (Gate 1): require a pull request with **1 approving review**, and
   require the **`validate`** status check.
3. **Environment `prod`** (Gate 2): `Settings → Environments → New environment → prod`, add a
   **required reviewer**. The production stage of `churn-cd` targets this environment, so promotion
   pauses for approval.
4. Create the **`release`** branch (it records what production is running; nothing triggers from it).

---

## Troubleshooting

**Terraform download fails (`openpgp: key expired`)**
Setting `DATABRICKS_TF_VERSION` alone isn't enough — the CLI still tries to download Terraform.
Install 1.5.7 and point the CLI at it:
```bash
curl -fsSL https://releases.hashicorp.com/terraform/1.5.7/terraform_1.5.7_linux_amd64.zip -o /tmp/tf.zip
unzip -oq /tmp/tf.zip -d /tmp/tfbin
export DATABRICKS_TF_EXEC_PATH=/tmp/tfbin/terraform DATABRICKS_TF_VERSION=1.5.7
```

**`PERMISSION_DENIED` during a pipeline run** — almost always a missing grant from step 4. The message
names the object (catalog, schema, model, experiment, or job); grant that specific privilege.

**Model version never appears / nothing to approve** — the training job must include the
**registration** step (notebook `03b`) after training; training alone only logs to MLflow tracking.
Check `resources/churn_jobs.yml`.

**Serving endpoint doesn't update after approval** — serverless model serving requires a budget
policy the workspace may not grant the running principal. The serving notebook treats that error as
non-fatal so the pipeline stays green and the model is still promoted; grant the policy to have the
endpoint update automatically.

**Deployment job fails at the serving step with a missing-table error** — the deployment job's tasks
must pass the stage schema (`stage_db`) so the notebooks resolve the right schema.

**`KeyError: 'optuna.study_direction'` on a repeat run** — hyperparameter studies are stored in a
shared MLflow-backed store; the training notebook uses a unique study name per run to avoid reloading
a previous study. Keep that behavior if you modify the notebook.

**Working from a Databricks Git folder** — use `Repos`/*Git folder* (clone through the Databricks UI
or `databricks repos create`). A plain copy of the files into a workspace folder is not Git-connected,
so the branch/commit/push controls will not appear.
