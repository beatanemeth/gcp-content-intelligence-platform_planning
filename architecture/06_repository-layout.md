# Step 6: repository layout

Decision log for step 6 of [00_roadmap.md](00_roadmap.md). Builds on [02_module-interface.md](02_module-interface.md) and [03_resource-inventory.md](03_resource-inventory.md).

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

One public implementation repository, with one folder per concern. The Terraform folder holds two things: the **module** (Part B, reusable) and the **deployment** (the root configuration that calls it).

The module is self-contained (S2.5): its Terraform files, function code, schemas and SQL all sit in one folder. Python tests live outside it, so the function zip contains only runtime code.

Everything specific to one GCP project (project ID, state bucket name) and the full dataset stay out of git. Example files show the expected format.

<br>

## Decisions

| ID   | Decision                                                                                                        | Status    |
| ---- | --------------------------------------------------------------------------------------------------------------- | --------- |
| S6.1 | One repository, one top-level folder per concern.                                                               | Delegated |
| S6.2 | Module at `terraform/modules/content_pipeline/`; root configuration at `terraform/deployment/`.                  | Delegated |
| S6.3 | Terraform files are split by topic, in both the module and the deployment.                                       | Delegated |
| S6.4 | Python tests live in `tests/`, outside the module; pytest finds the function code through its `pythonpath` setting. | Delegated |
| S6.5 | The zip is written to `.build/` in the deployment folder (git-ignored); `archive_file` excludes `__pycache__`.    | Delegated |
| S6.6 | Schema and SQL files are named after their table or view; SQL files sit in one subfolder per layer.              | Delegated |
| S6.7 | Committed: dummy data, lock file, example files. Git-ignored: full dataset, `terraform.tfvars`, `backend.hcl`, build output. | Delegated |
| S6.8 | The upload script creates the empty `_READY` marker itself; no marker file is committed.                         | Delegated |

<br>

## Folder tree

Repository name assumed: `gcp-content-intelligence-platform`.

```text
gcp-content-intelligence-platform/
├── README.md                              Story, architecture, runbook, results
├── .gitignore
├── .tflint.hcl                            TFLint settings (step 7)
├── pyproject.toml                         pytest settings, incl. pythonpath (S6.4)
├── .github/
│   └── workflows/
│       ├── terraform-checks.yml           fmt, validate, TFLint (step 7)
│       └── python-tests.yml               pytest (step 7)
├── bootstrap/
│   └── bootstrap.sh                       Part A stand-in (S1.9)
├── terraform/
│   ├── modules/
│   │   └── content_pipeline/              Part B: the module
│   │       ├── README.md                  Inputs, outputs, prerequisites
│   │       ├── versions.tf                required_version, required_providers (>=)
│   │       ├── variables.tf               project_id, region, labels
│   │       ├── locals.tf                  feature name, names, labels, table and view maps
│   │       ├── outputs.tf                 data_bucket_name, dataset_ids, function_name
│   │       ├── storage.tf                 2 buckets
│   │       ├── function.tf                archive_file, zip upload, function
│   │       ├── bigquery.tf                datasets, staging tables, views
│   │       ├── iam.tf                     3 service accounts, 8 grants
│   │       ├── function/                  Zipped and deployed (D1.11)
│   │       │   ├── main.py
│   │       │   └── requirements.txt
│   │       ├── schemas/                   One JSON file per staging table (D3.4)
│   │       │   ├── blog_posts.json
│   │       │   └── ... (6 files)
│   │       └── sql/                       One file per view (D3.14)
│   │           ├── cleaned/
│   │           │   ├── blog_posts.sql
│   │           │   ├── articles.sql
│   │           │   └── content.sql
│   │           └── analysis/
│   │               ├── content_per_month.sql
│   │               └── ... (6 files)
│   └── deployment/                        Root configuration: calls the module
│       ├── versions.tf                    backend "gcs", required_providers (~>)
│       ├── providers.tf                   google provider
│       ├── main.tf                        module "content_pipeline" { ... }
│       ├── variables.tf                   project_id, region
│       ├── outputs.tf                     Re-exported module outputs (S2.4)
│       ├── .terraform.lock.hcl            Committed (S2.6)
│       ├── terraform.tfvars.example       Committed; copied to terraform.tfvars
│       └── backend.hcl.example            Committed; copied to backend.hcl
├── scripts/
│   ├── upload_data.sh                     gcloud storage cp, marker last (D1.9)
│   └── data_checks.sql                    Row counts and data quality checks (S8.7)
├── tests/
│   └── function/
│       └── test_main.py                   pytest against the dummy data
├── data/
│   ├── dummy/                             Committed
│   │   ├── blogs/                         blog_posts.json, blog_categories.json, ...
│   │   └── articles/                      articles.json, article_categories.json
│   └── raw/                               Git-ignored: the full dataset
└── docs/                                  Diagrams, decision summary
```

<br>

## Reasoning

### S6.1: one repository

- **One story in one place:** a reviewer sees the module, the code it deploys, the SQL, the tests and the CI side by side.
- **Folder per concern:** `bootstrap` (Part A stand-in), `terraform` (infrastructure), `scripts` (data), `tests`, `data`, `docs`.

### S6.2: module and deployment folders

- **`modules/content_pipeline/`:** the standard Terraform location for local modules. The folder name matches the module block name `module "content_pipeline"` (snake_case, D7.5).
- **`deployment/`:** the folder where `terraform init`, `plan` and `apply` run. It belongs to Part B, like the module: the feature team deploys its own module into Part A's project, using Part A's values as inputs.
- **Why not `environments/dev/`:** the module is deployed once (S1.10). An environment folder would suggest instances that do not exist.
- **Module source:** `source = "../modules/content_pipeline"` in the deployment's `main.tf`.

### S6.3: files by topic

**Terraform background:** Terraform reads every `.tf` file in a folder as one configuration. File names carry no meaning to Terraform; they exist for readers. A resource in `function.tf` can reference one in `storage.tf` without any import.

| File          | Content                                              |
| ------------- | ---------------------------------------------------- |
| `versions.tf` | Terraform and provider versions (convention)          |
| `variables.tf`, `outputs.tf`, `locals.tf` | The interface and computed values (convention) |
| `storage.tf`, `function.tf`, `bigquery.tf`, `iam.tf` | One file per component of the inventory (step 3) |

### S6.4: tests outside the module

- **Why outside:** `archive_file` zips the whole `function/` folder. Tests inside it would be deployed with the code (D7.2).
- **How pytest finds the code:** `pythonpath = ["terraform/modules/content_pipeline/function"]` in `pyproject.toml`. Tests then `import main` directly.
- **Trade-off accepted:** the function code sits deep in the Terraform tree. In return, the module is self-contained (S2.5).

### S6.5: where the zip goes

- **`${path.root}/.build/content-function.zip`:** `path.root` is the deployment folder, where Terraform runs. `.build/` is git-ignored.
- **Why not inside the module:** a module may be stored read-only (for example when downloaded from a registry). Writing build output into it is bad practice.
- **`excludes = ["__pycache__"]`:** running tests locally creates `__pycache__` folders. Without the exclusion, they would land in the zip and change its hash, so the function would redeploy for no code change.

### S6.6: file names follow resource names

- **One name through all layers (D2.6):** `schemas/blog_posts.json` defines `content_staging.blog_posts`; `sql/cleaned/blog_posts.sql` defines `content_cleaned.blog_posts`.
- **Subfolder per layer:** each SQL subfolder matches one dataset (`content_cleaned`, `content_analysis`), so a reader sees where each view lands.
- **`for_each` from file names:** the maps in `locals.tf` list the table and view names; each name points to its file. Adding a view means adding one SQL file and one map entry.

### S6.7: what is committed

| Committed                                    | Git-ignored                                   | Why ignored                                    |
| -------------------------------------------- | --------------------------------------------- | ---------------------------------------------- |
| `data/dummy/`                                | `data/raw/`                                   | The full dataset stays private                 |
| `.terraform.lock.hcl`                        | `.terraform/`                                 | Downloaded providers; rebuilt by `init`         |
| `terraform.tfvars.example`                   | `terraform.tfvars`                            | Contains the project ID                        |
| `backend.hcl.example`                        | `backend.hcl`                                 | Contains the state bucket name                 |
| Nothing                                      | `.build/`, `__pycache__/`, `.venv/`           | Generated files                                |

- **No `*.tfstate` locally:** state lives in Part A's bucket (S1.3). `*.tfstate*` is still git-ignored as a safety net.

### S6.8: marker created by the script

- **How:** the script creates an empty temporary file and uploads it as `raw/<group>/_READY` after the group's data (D1.9).
- **Why:** nothing to commit, and no marker can sit inside a data folder where a wildcard could pick it up early.

<br>

## Changes to earlier decisions

| Earlier | Was                                     | Now                                                 |
| ------- | --------------------------------------- | --------------------------------------------------- |
| D7.1    | A top-level folder for the function     | The function sits inside the module folder (S2.5, S6.2) |
| D7.3    | One root configuration, no modules      | A module plus a deployment folder (S6.2)            |
| D7.4    | `backend.hcl` passed at `init`          | Unchanged (S2.8)                                    |
| D7.5    | kebab-case for GCP names, snake_case for BigQuery and Terraform | Unchanged; the `content` prefix is added (S2.2) |

<br>

## Open questions

- None for step 6. The zip location from step 3 is settled by S6.5.
