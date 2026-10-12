# Decision overview

All decision tables from the brainstorming in one place: the scope from files 01 and 03, the step and cross-cutting decisions from files 04 to 07. Each section links to its source file, which holds the reasoning. Scope and repository layout have no file of their own, so their short reasons are here.

**Next step:** the standalone module plan (`architecture/`) starts from these decisions. D0 to D3 carry over unchanged; several D4 and D7 decisions are re-decided for Part A and Part B (`architecture/00_roadmap.md`, section "Earlier decisions re-decided").

**Status values:** _Agreed_ (confirmed by the maintainer), _Delegated_ (chosen by Claude at the maintainer's request).

**Order:** cross-cutting topics appear in discussion order (1, 2, 6, 5, 3, 4), so decision IDs stay ascending. Data terms (full dataset, dummy data, raw, processed, cleaned) follow the [glossary](#data-glossary).

<br>

## Data glossary

| Term                  | Meaning                                                                                              | Where it lives                                                                                                 |
| --------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Original Wix data** | Real Wix exports of the original project; never enters this project                                  | Original project only                                                                                          |
| **Full dataset**      | Fictional blogs and articles with the Wix export structure; private (significant effort to create)   | Laptop only; git-ignored `data/raw/` in the implementation repository                                          |
| **Dummy data**        | 2/3 of each full dataset file; references intact; lorem ipsum text; IDs, dates and metrics unchanged | `dummy_data_blogs/`, `dummy_data_articles/` in this repository; `data/dummy/` in the implementation repository |

| Stage         | Meaning                                                          | Where it rests      |
| ------------- | ---------------------------------------------------------------- | ------------------- |
| **Raw**       | Uploaded files, unchanged: API envelopes, embedded HTML          | Bucket `raw/`       |
| **Processed** | Half-cleaned: envelopes removed, HTML cleaned, NDJSON            | Bucket `processed/` |
| **Staging**   | 1:1 typed copies of the processed files                          | BigQuery `staging`  |
| **Cleaned**   | Final shape: one wide view per content type, plus a unified view | BigQuery `cleaned`  |
| **Analysis**  | One view per analysis question                                   | BigQuery `analysis` |

<br>

## Scope

Source: [01_bea_original_vs_current_project_background.md](01_bea_original_vs_current_project_background.md#scope-decision-events-data) (D0.1), [03_bea_answers_and_thoughts.md](03_bea_answers_and_thoughts.md#step-4-optional-visualisation-in-looker-studio) (D0.2)

| ID   | Decision                                                         | Status |
| ---- | ---------------------------------------------------------------- | ------ |
| D0.1 | Blogs and articles both stay in scope.                           | Agreed |
| D0.2 | Looker Studio is an optional stretch goal, not part of the core. | Agreed |

- **Why articles stay (D0.1):** they hold the only embedded HTML and the `items` envelope, form the second file group, and make the `UNION ALL` content model possible. The infrastructure is the same for 4 files or 6.
- **Stretch goals:** a Looker Studio report (D0.2); a refactor of the function to a Cloud Run service with a Dockerfile (D1.8).

<br>

## Step 1: raw data upload

Source: [04_claude_step1_raw_data_upload.md](04_claude_step1_raw_data_upload.md)

| ID    | Decision                                                                                      | Status |
| ----- | --------------------------------------------------------------------------------------------- | ------ |
| D1.1  | The laptop upload replaces the Wix API fetch. The data is fictional; there is no API to call. | Agreed |
| D1.2  | Data files are uploaded with `gcloud storage cp`, not with Terraform.                         | Agreed |
| D1.3  | Cloud data is disposable. The raw files on the laptop are the source of truth.                | Agreed |
| D1.4  | `terraform destroy` removes buckets and datasets together with their contents.                | Agreed |
| D1.5  | Processing is event-driven: a file landing in the bucket starts it.                           | Agreed |
| D1.6  | The processing step does both NDJSON reshaping and HTML cleaning.                             | Agreed |
| D1.7  | One bucket with prefixes (for example `raw/`, `processed/`).                                  | Agreed |
| D1.8  | Compute is a Cloud Run function; refactor to a Cloud Run service is optional at the end.      | Agreed |
| D1.9  | A marker file (`_READY`) uploaded last signals that a file group is complete.                 | Agreed |
| D1.10 | Processing always overwrites its output; it never appends.                                    | Agreed |
| D1.11 | Terraform zips, uploads and deploys the function code.                                        | Agreed |
| D1.12 | Function code lives in a separate source bucket, not in the data bucket.                      | Agreed |

<br>

## Step 2: data transformation

Source: [05_claude_step2_data_transformation.md](05_claude_step2_data_transformation.md)

| ID   | Decision                                                                                             | Status |
| ---- | ---------------------------------------------------------------------------------------------------- | ------ |
| D2.1 | The function writes NDJSON files to `processed/`; BigQuery loads from those files.                   | Agreed |
| D2.2 | Python does only what SQL cannot do well (reshaping, HTML cleaning); SQL does all relational work.   | Agreed |
| D2.3 | The function starts the BigQuery load jobs itself, after writing a group's NDJSON files.             | Agreed |
| D2.4 | The cleaned layer is a set of BigQuery views defined in Terraform.                                   | Agreed |
| D2.5 | `processed/` mirrors `raw/`: same group folders, same base names, `.ndjson` extension, fixed names.  | Agreed |
| D2.6 | Raw files follow one naming pattern: `<group>_<entity>.json`, snake_case, plural, no `_data` suffix. | Agreed |

<br>

## Step 3: BigQuery

Source: [06_claude_step3_bigquery.md](06_claude_step3_bigquery.md)

| ID    | Decision                                                                                            | Status |
| ----- | --------------------------------------------------------------------------------------------------- | ------ |
| D3.1  | One dataset per layer: `staging`, `cleaned` (and `analysis`, D3.13).                                | Agreed |
| D3.2  | The datasets and the data bucket share one region.                                                  | Agreed |
| D3.3  | Staging tables are 1:1 copies of the processed files, with Wix field names and nesting kept.        | Agreed |
| D3.4  | Schemas live in JSON files, referenced from Terraform.                                              | Agreed |
| D3.5  | Cleaned views use nested and repeated fields (arrays of structs), not bridge tables.                | Agreed |
| D3.6  | Cleaned views rename columns to snake_case.                                                         | Agreed |
| D3.7  | No partitioning or clustering.                                                                      | Agreed |
| D3.8  | Every dataset carries Terraform labels (for example `project`, `layer`).                            | Agreed |
| D3.9  | Strict staging: typed columns; any unexpected value or field fails the load.                        | Agreed |
| D3.10 | Unused, always-empty arrays stay in the staging schema as `REPEATED STRING` and are never queried.  | Agreed |
| D3.11 | Unified content view: common columns, with `categories` and `tags` as string arrays for both types. | Agreed |
| D3.12 | Region `europe-west1` (Belgium) for all regional resources.                                         | Agreed |
| D3.13 | Analysis queries are views in a third dataset, `analysis`, created by Terraform.                    | Agreed |
| D3.14 | All view SQL lives in `.sql` files; Terraform reads them with `templatefile()`.                     | Agreed |

<br>

## Cross-cutting topic 1: project setup and Terraform state

Source: [07_claude_cross_cutting_topics.md](07_claude_cross_cutting_topics.md)

| ID   | Decision                                                                                                          | Status |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ------ |
| D4.1 | A one-time bootstrap script (`gcloud`) creates the dedicated project, links billing and creates the state bucket. | Agreed |
| D4.2 | Terraform state lives in a Cloud Storage bucket (`gcs` backend) with object versioning on.                        | Agreed |
| D4.3 | Terraform enables the required APIs (`google_project_service`), with `disable_on_destroy = false`.                | Agreed |
| D4.4 | `terraform destroy` removes everything except the bootstrap resources (project, state bucket).                    | Agreed |

<br>

## Cross-cutting topic 2: IAM

Source: [07_claude_cross_cutting_topics.md](07_claude_cross_cutting_topics.md)

| ID   | Decision                                                                                                      | Status |
| ---- | ------------------------------------------------------------------------------------------------------------- | ------ |
| D4.5 | The maintainer holds `Owner` on the dedicated project and runs Terraform with their own `gcloud` credentials. | Agreed |
| D4.6 | Each workload has its own service account: function runtime, trigger, function build, CI.                     | Agreed |
| D4.7 | Roles are granted at the smallest scope that works.                                                           | Agreed |
| D4.8 | Terraform uses additive IAM resources (`*_iam_member`) only.                                                  | Agreed |
| D4.9 | No service account keys.                                                                                      | Agreed |

<br>

## Cross-cutting topic 6: GitHub Actions

Source: [07_claude_cross_cutting_topics.md](07_claude_cross_cutting_topics.md)

| ID    | Decision                                                                                                   | Status |
| ----- | ---------------------------------------------------------------------------------------------------------- | ------ |
| D4.10 | CI never changes real infrastructure. `apply` and `destroy` run from the laptop only (no tier 3).          | Agreed |
| D4.11 | Tier 1 (checks) runs on every push to a feature branch, on pull requests to `main` and on merge to `main`. | Agreed |
| D4.12 | Tier 2 (`terraform plan`) runs on pull requests to `main` only.                                            | Agreed |
| D4.13 | The CI service account is read-only; `plan` runs without a state lock.                                     | Agreed |
| D4.14 | The project ID reaches workflows as a GitHub secret, so logs mask it.                                      | Agreed |

<br>

## Cross-cutting topic 5: local testing and tier 1 tools

Source: [07_claude_cross_cutting_topics.md](07_claude_cross_cutting_topics.md)

| ID    | Decision                                                                                             | Status    |
| ----- | ---------------------------------------------------------------------------------------------------- | --------- |
| D4.15 | The same checks run on the laptop (pre-commit hooks) and in CI (tier 1), from one configuration.     | Delegated |
| D4.16 | Python transformation logic is pure functions, unit-tested with pytest against the dummy files.      | Delegated |
| D4.17 | No local SQL engine. BigQuery validates view SQL on `apply`; a smoke test validates the data.        | Delegated |
| D4.18 | The end-to-end test is a smoke test: upload dummy data, then compare row counts and query the views. | Delegated |

<br>

## Cross-cutting topic 3: cost control and teardown

Source: [07_claude_cross_cutting_topics.md](07_claude_cross_cutting_topics.md)

| ID    | Decision                                                                                                       | Status    |
| ----- | -------------------------------------------------------------------------------------------------------------- | --------- |
| D4.19 | The bootstrap script creates a small billing budget with email alerts at 50 %, 90 % and 100 %.                 | Delegated |
| D4.20 | Terraform creates the Artifact Registry repository for the function's container images, with a cleanup policy. | Delegated |
| D4.21 | A teardown checklist in the README: `destroy`, check for leftovers, delete the project at the very end.        | Delegated |

<br>

## Cross-cutting topic 4: public repository

Source: [07_claude_cross_cutting_topics.md](07_claude_cross_cutting_topics.md)

No decision table: a reminder list the maintainer handles (`.gitignore`, push protection, dummy data, private Looker Studio report). Only dummy data is committed; the full dataset stays private because creating it took significant effort and others could reuse it.

<br>

## Repository layout

Source: decided at the end of the brainstorming, without a separate file; the short reasons follow the table. The module plan re-plans the layout in `architecture/06_repository-layout.md`.

| ID   | Decision                                                                                          | Status    |
| ---- | ------------------------------------------------------------------------------------------------- | --------- |
| D7.1 | One repository with one folder per concern: Terraform, function, tests, scripts, data, workflows. | Delegated |
| D7.2 | Tests live outside the function folder, so the function zip contains only runtime code.           | Delegated |
| D7.3 | One Terraform root configuration, split into files by topic. No modules.                          | Delegated |
| D7.4 | The state bucket name is passed at `terraform init` from a git-ignored `backend.hcl`.             | Delegated |
| D7.5 | Naming: kebab-case for GCP resource names, snake_case for BigQuery and Terraform identifiers.     | Delegated |

- **Tests outside the function folder (D7.2):** the zip that Terraform uploads contains only runtime code.
- **One root configuration (D7.3):** the simplest layout for one environment. Re-decided in the module plan: a module plus a deployment folder.
- **`backend.hcl` (D7.4):** a `backend` block cannot use variables, and the state bucket name is a private value. A git-ignored file passed at `terraform init` solves both.
- **Naming (D7.5):** service account IDs allow hyphens but not underscores; BigQuery dataset and table names allow underscores but not hyphens.
