# Step 2: module interface

> **Reviewed:** [step-2_module-interface.md](../v3_review_plan/step-2_module-interface.md) corrects this step. Where they differ, the review wins.

Decision log for step 2 of [00_roadmap.md](00_roadmap.md). Builds on [01_platform-contract.md](01_platform-contract.md).

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

The module takes three inputs: the project ID, the region and extra labels. It returns three outputs: the data bucket name, the dataset IDs and the function name.

Every name the module creates starts with the feature name `content`. Other features share the project, so `content_staging` cannot clash with another feature's `staging`.

The module is self-contained. Its function code, SQL files and schema files sit inside the module folder. It contains no `provider` or `backend` block; the root configuration supplies both.

<br>

## Decisions

| ID   | Decision                                                                                                       | Status    |
| ---- | -------------------------------------------------------------------------------------------------------------- | --------- |
| S2.1 | Inputs: `project_id` and `region` (both required), `labels` (optional, default empty).                          | Delegated |
| S2.2 | Every created name carries the feature name `content`, defined once as a local value.                          | Delegated |
| S2.3 | The module sets `feature`, `managed_by` and (on datasets) `layer` labels; the caller's labels are merged in.   | Delegated |
| S2.4 | Outputs: `data_bucket_name`, `dataset_ids`, `function_name`. Only values that something reads.                 | Delegated |
| S2.5 | Function code, SQL and schema files live inside the module folder, referenced with `path.module`.              | Delegated |
| S2.6 | The module sets minimum versions (`>=`); the root sets the upper limit (`~>`) and commits the lock file.       | Delegated |
| S2.7 | Resources set `project` and `region` (or `location`) explicitly from the inputs, not from provider defaults.   | Delegated |
| S2.8 | The root configuration holds the backend, the provider, the module call and the re-exported outputs.           | Delegated |

<br>

## S2.1: inputs

| Variable     | Type          | Required | Example (set in the root)     |
| ------------ | ------------- | -------- | ----------------------------- |
| `project_id` | `string`      | Yes      | From a git-ignored `terraform.tfvars` |
| `region`     | `string`      | Yes      | `europe-west1` (D3.12)        |
| `labels`     | `map(string)` | No       | `{ owner = "data-team" }`     |

- **Required means no `default`:** `plan` fails if the caller forgets the value. A missing project or region is caught before anything is created.
- **Why `region` is required:** region is a platform policy (data residency), not a feature choice. The module should not quietly pick one. The value `europe-west1` lives in the root.
- **Why nothing else:** one instance (S1.10), so no environment or name prefix input. Every extra variable is one more thing to explain.

<br>

## S2.2: naming

**Terraform background:** a `locals` block defines named values computed inside the module. The caller cannot set them. One local (`feature = "content"`) feeds every name, so the prefix is written once.

| Resource                     | Name                              | Uniqueness scope   | Character rules                        |
| ---------------------------- | --------------------------------- | ------------------ | -------------------------------------- |
| Data bucket                  | `<project-id>-content-data`       | All of GCP         | Lowercase, digits, hyphens             |
| Source bucket                | `<project-id>-content-source`     | All of GCP         | Lowercase, digits, hyphens             |
| Datasets                     | `content_staging`, `content_cleaned`, `content_analysis` | Project | Letters, digits, underscores only |
| Tables and views             | `blog_posts`, `articles`, ... (D2.6) | Dataset         | The dataset carries the prefix         |
| Function                     | `content-transform`               | Project and region | Lowercase, digits, hyphens             |
| Service accounts             | `content-<role>` (roles in step 4) | Project           | 6 to 30 characters, lowercase, hyphens |

- **Why a prefix:** dataset, function and service account names are unique per project. Other features share the project (S1.1), so an unprefixed `staging` dataset would clash with the next feature's `staging`.
- **Why `content`:** short, readable, and it names the feature. Bucket names stay well under the 63-character limit (project IDs have at most 30).
- **Hyphens or underscores:** each GCP service has its own rules. BigQuery forbids hyphens; bucket and service account names favour them.

<br>

## S2.3: labels

| Label        | Value                                   | Set by     | On                          |
| ------------ | --------------------------------------- | ---------- | --------------------------- |
| `feature`    | `content-pipeline`                      | Module     | Every resource with labels  |
| `managed_by` | `terraform`                             | Module     | Every resource with labels  |
| `layer`      | `staging`, `cleaned`, `analysis`        | Module     | Datasets only               |
| Any other    | For example `owner`                     | Caller     | Every resource with labels  |

- **`merge(var.labels, local.labels)`:** combines two maps. When both contain the same key, the later map wins. The module's labels come last, so a caller cannot overwrite `feature`.
- **`feature` instead of `project` (D3.8):** billing data already shows the project. In a shared project, the useful filter is the feature.
- **Not everything takes labels:** service accounts and IAM bindings have none.

<br>

## S2.4: outputs

| Output             | Value                                                   | Read by                                   |
| ------------------ | ------------------------------------------------------- | ----------------------------------------- |
| `data_bucket_name` | `<project-id>-content-data`                             | The upload script                         |
| `dataset_ids`      | Map: `staging`, `cleaned`, `analysis` to dataset IDs    | Queries in the README, Looker Studio (D0.2) |
| `function_name`    | `content-transform`                                     | Reading the function's logs               |

- **Rule:** output only what something reads. Service account emails are not output; nothing outside the module uses them.
- **Upload script:** reads the bucket name with `terraform output -raw data_bucket_name`. No name is typed twice.
- **Terraform background:** `terraform output` shows only the **root's** outputs. A module output reaches the command line only when the root declares its own output with `value = module.content_pipeline.data_bucket_name` (S2.8).

<br>

## S2.5: files inside the module

- **`path.module`:** the path of the module's own folder. `"${path.module}/sql/cleaned/blog_posts.sql"` works no matter where the module is called from.
- **Self-contained:** a module that brings its own code, SQL and schemas can be called from any root without copying files around.
- **Consequence:** the function zip (D1.11), the schema files (D3.4) and the SQL files (D3.14) all sit under the module folder. The exact tree is settled in step 6.

<br>

## S2.6: version constraints

| Where      | Setting                                           | Why                                                                 |
| ---------- | ------------------------------------------------- | ------------------------------------------------------------------- |
| Module     | `required_version = ">= x.y"`, providers `>= x.y` | States what the module needs. An upper limit would block callers from upgrading. |
| Root       | Providers `~> x.y`                                | Allows patch and minor updates within one major version, blocks the next major. |
| Root       | `.terraform.lock.hcl` committed                   | Records the exact provider versions, so every `init` installs the same ones. |

- **Providers:** `hashicorp/google` and `hashicorp/archive` (D1.11).
- **Exact numbers:** set at implementation time from the current releases.

<br>

## S2.7: explicit project and region

- **What:** each resource sets `project = var.project_id` and `location` or `region = var.region`.
- **Alternative:** omit them and inherit the provider's defaults from the root.
- **Why explicit:** the module needs `project_id` anyway (bucket names, SQL placeholders). A reader sees where each resource lands without opening the root.

<br>

## S2.8: the root configuration

| Piece            | Content                                                                 |
| ---------------- | ----------------------------------------------------------------------- |
| `backend "gcs"`  | State in Part A's bucket (S1.3), under the prefix `content-pipeline`    |
| `provider`       | `google`, with the project and region from the root's variables         |
| `module` block   | `source` points to the module folder; passes `project_id`, `region`, `labels` |
| Outputs          | Re-export the module's three outputs                                    |
| Variables        | `project_id` (from git-ignored `terraform.tfvars`), `region` (default `europe-west1`) |

- **Owner:** Part B. The feature team deploys its own module; Part A only supplies the values (project ID, region, state bucket). Adding the pipeline changes no Part A code.
- **Backend values at `init`:** a `backend` block cannot read variables, because the backend is configured before variables exist. The state bucket name comes from a git-ignored `backend.hcl` passed with `terraform init -backend-config=backend.hcl`.
- **Public repository:** the project ID and state bucket name never appear in committed files. Committed example files (`terraform.tfvars.example`, `backend.hcl.example`) show the format.

<br>

## Changes to earlier decisions

| Earlier | Was                                  | Now                                           | Reason                         |
| ------- | ------------------------------------ | --------------------------------------------- | ------------------------------ |
| D3.1    | Datasets `staging`, `cleaned`, `analysis` | `content_staging`, `content_cleaned`, `content_analysis` | Shared project (S2.2) |
| D3.8    | Labels `project`, `layer`            | Labels `feature`, `managed_by`, `layer`       | Shared project (S2.3)          |

<br>

## Open questions

- None for step 2.
