# Step 8: build phases and runbook

Decision log for step 8 of [00_roadmap.md](00_roadmap.md). Builds on every earlier step.

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

The build has 10 phases (0 to 9). Each phase adds one layer of the pipeline and ends with a check that proves it works. The next phase starts only when the check passes.

The order follows the data, with one rule: test each piece on its own before automating it. The Python transformation runs locally before it runs in the cloud. The staging schemas are tested with a manual load before the function loads them.

`destroy` followed by `apply` is tested twice: on the empty skeleton, and on the full system.

The runbook is the sequence a reader of the README follows: bootstrap once, then `apply`, upload, verify, `destroy`.

<br>

## Decisions

| ID   | Decision                                                                                                        | Status    |
| ---- | --------------------------------------------------------------------------------------------------------------- | --------- |
| S8.1 | 10 build phases (0 to 9); each ends with a check; a phase starts only when the previous check passes.                     | Delegated |
| S8.2 | The Python transformation is built and tested locally before any cloud compute exists.                          | Delegated |
| S8.3 | Staging schemas are tested with a manual `bq load` before the function automates the load.                      | Delegated |
| S8.4 | `destroy` and rebuild are tested in phase 2 (skeleton) and phase 8 (full system).                               | Delegated |
| S8.5 | One branch and one pull request per phase, so CI runs on every phase.                                           | Delegated |
| S8.6 | The upload script takes the data folder as an argument; the default is `data/dummy`.                            | Delegated |
| S8.7 | Row counts and data quality are checked with one SQL file, `scripts/data_checks.sql`, run with `bq query`.     | Delegated |
| S8.8 | The query results published in the README come from the full dataset.                                          | Agreed    |

<br>

## Build phases

| Phase | Builds                                        | Where          | Done when                                                                                       |
| ----- | --------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------- |
| 0     | Repository and dummy data                     | Laptop         | Folder tree in place; dummy files named per D2.6; full dataset git-ignored                      |
| 1     | Part A stand-in (`bootstrap.sh`)              | GCP            | Project, APIs, state bucket, service agent grant and budget exist                               |
| 2     | Terraform skeleton, buckets, Terraform CI     | GCP, GitHub    | 2 buckets exist; state in Part A's bucket; CI green; `destroy` and `apply` round trip works     |
| 3     | Python transformation, Python CI              | Laptop, GitHub | pytest green; 6 NDJSON files produced locally with the expected record counts                   |
| 4     | Datasets and staging tables                   | GCP            | Manual `bq load` of the local NDJSON files succeeds with strict settings; row counts match      |
| 5     | IAM, function, trigger, upload script         | GCP            | Upload of dummy data fills `processed/` and the staging tables; a second upload gives the same counts |
| 6     | Cleaned views (`blog_posts`, `articles`, `content`) | GCP      | Each view returns one row per post or article; categories and tags are filled                    |
| 7     | Analysis views                                | GCP            | All 6 views return plausible results                                                            |
| 8     | Full dataset run and rebuild test             | GCP            | `destroy`, `apply`, upload of the full dataset; counts match the full dataset; final `destroy`  |
| 9     | README and docs                               | GitHub         | Story, diagrams, runbook, results and decision summary published                                |

<br>

## Phase details

### Phase 0: repository and dummy data

- Create the folder tree from [06_repository-layout.md](06_repository-layout.md) and the `.gitignore`.
- Copy the dummy data from this planning repository (`dummy_data_blogs/`, `dummy_data_articles/`) into `data/dummy/blogs/` and `data/dummy/articles/`, renamed per D2.6 (no `dummy_` prefix).
- **Check (D3.10):** confirm that the dummy files contain the same always-empty arrays as the full dataset, so the staging schemas are tested against them.
- **Check:** `git status` shows no file from `data/raw/`.

### Phase 1: Part A stand-in

`bootstrap.sh` runs once, with the maintainer's own login (S4.5).

| Order | Creates                                              | Decision |
| ----- | ---------------------------------------------------- | -------- |
| 1     | Project, billing link                                | S1.2     |
| 2     | APIs (the S1.4 list, plus the Billing Budget API for the budget) | S1.4 |
| 3     | State bucket: versioning on, uniform access, public access prevention | S1.3 |
| 4     | Cloud Storage service agent grant: `roles/pubsub.publisher` | S1.5 |
| 5     | Budget with email alerts                             | S1.7     |

- **Service agent email:** `gcloud storage service-agent --project=<project-id>` prints it (and creates the agent if it does not exist yet).
- **State bucket versioning:** unlike the pipeline's buckets (S3.2), the state bucket keeps old versions. A corrupted or deleted state file can then be restored.
- **Check:** `gcloud services list --enabled` shows every API; the state bucket exists.

### Phase 2: Terraform skeleton and CI

- **Deployment:** `versions.tf` (backend, provider versions), `providers.tf`, `main.tf` (module call), `variables.tf`, `outputs.tf`, example files.
- **Module:** `versions.tf`, `variables.tf`, `locals.tf`, `outputs.tf`, and `storage.tf` with the 2 buckets only.
- **CI:** `.tflint.hcl` and `terraform-checks.yml`.
- **Why the buckets first:** the smallest resource that proves the whole chain: login, backend, provider, module call, outputs.
- **Check:**
  - `terraform output data_bucket_name` prints the bucket name.
  - The state file exists in Part A's bucket under the `content-pipeline` prefix.
  - `destroy` removes both buckets; `apply` creates them again (S8.4).

### Phase 3: Python transformation (local)

- **Code:** pure functions in `main.py`: unwrap each envelope into records, clean the HTML in `articleText`, write NDJSON (D2.2). The cloud entry point (event handling, load jobs) comes in phase 5.
- **Settings from the environment:** the code reads `PROJECT_ID`, `STAGING_DATASET` and `BQ_LOCATION` (S3.9). The tests set them, so no name is hard-coded.
- **Tests:** pytest against `data/dummy/`, plus `python-tests.yml`.
- **Check:** 6 NDJSON files written to a local temporary folder; the record count of each matches its source file.

### Phase 4: datasets and staging tables

- **Terraform:** the 3 datasets and the 6 staging tables, with their JSON schemas.
- **Schema files:** give every column an explicit `mode` (`NULLABLE`, `REQUIRED` or `REPEATED`). BigQuery fills in a missing mode, and Terraform would then report the same schema change on every `plan`.
- **Manual load (S8.3):** `bq load --replace --source_format=NEWLINE_DELIMITED_JSON` for each local NDJSON file from phase 3, into its staging table, with no tolerance for bad records or unknown fields (D3.9).
- **Why by hand first:** a schema error shows up here, on its own. In phase 5 the same error would be buried in the function's logs, mixed with IAM and trigger problems.
- **Check:** `scripts/data_checks.sql` returns the expected row count for all 6 tables, and 0 for every other check.

### Phase 5: IAM, function and trigger

- **Terraform:** `iam.tf` (3 service accounts, 8 grants) and `function.tf` (zip, upload, function with trigger, `depends_on` from S5.2).
- **Code:** the function's entry point: ignore everything except `_READY`, transform the group, write to `processed/`, start the load jobs, log the record and row counts. Dataset and location come from the environment variables (S3.9).
- **Upload script:** `scripts/upload_data.sh`, reading the bucket name from `terraform output -raw data_bucket_name` (S2.4).
- **First `apply`:** if it fails on a permission error, run `apply` again (S5.3). If that happens more than once, add `time_sleep`.
- **Check:**
  - The function logs show 2 runs that did the work (one per `_READY`) and several that exited at once. Some rejected and retried events are expected (S3.3).
  - `processed/` holds 6 NDJSON files.
  - `scripts/data_checks.sql` matches phase 4.
  - A second upload gives the same counts (D1.10).

### Phase 6: cleaned views

- **Terraform:** the `cleaned` view block and the separate `content` view (S3.6).
- **Check:** row count per view equals the staging row count of `blog_posts` and `articles`; `content` equals their sum; every category and tag ID resolves to a name (no `NULL` names in the arrays); `articles` exposes the cleaned `article_text`.

### Phase 7: analysis views

- **Terraform:** the 6 analysis views, each SQL file starting with its metric definition (S3.11).
- **Check:** each view returns rows; spot-check one number by hand (for example, posts in one month, or one article's word count).

### Phase 8: full dataset and rebuild test

1. `terraform destroy`
2. `terraform apply`
3. `scripts/upload_data.sh data/raw`
4. `scripts/data_checks.sql`: 113 blog posts, 18 articles, 9 blog categories, 17 tags, 8 members, 8 article categories; 0 for every other check.
5. **Check from D3.5:** the 2 posts without tags still appear in the cleaned views and in `content`.
6. `terraform destroy`

### Phase 9: README and docs

- **Story:** Part A and Part B, and why the module is standalone.
- **Diagrams:** pipeline and Terraform structure.
- **Runbook:** the section below.
- **Results:** query output from the analysis views, run on the full dataset in phase 8 (S8.8).
- **Decision summary:** the key decisions with one-line reasons.

<br>

## Runbook

### One-time setup

| Step | Command                                                                  | Purpose                              |
| ---- | ------------------------------------------------------------------------ | ------------------------------------ |
| 1    | `gcloud auth login` and `gcloud auth application-default login`          | Log in for `gcloud` and for Terraform |
| 2    | `bootstrap/bootstrap.sh`                                                 | Create Part A (phase 1)              |
| 3    | Copy `terraform.tfvars.example` and `backend.hcl.example`, fill in values | Project ID and state bucket name     |
| 4    | `terraform init -backend-config=backend.hcl` (in `terraform/deployment/`) | Connect to the state bucket; download providers |

### Every run

| Step | Command                                                     | Purpose                                     |
| ---- | ----------------------------------------------------------- | ------------------------------------------- |
| 1    | `terraform plan`                                            | Review what will change                     |
| 2    | `terraform apply`                                           | Build or update the infrastructure          |
| 3    | `scripts/upload_data.sh` (or `scripts/upload_data.sh data/raw`) | Upload a dataset; the pipeline starts by itself |
| 4    | `gcloud functions logs read content-transform --gen2 --region=europe-west1` | Follow the function's runs   |
| 5    | `bq query --use_legacy_sql=false < scripts/data_checks.sql` | Check row counts and data quality           |
| 6    | Query the views in `content_cleaned` and `content_analysis` | Look at the results                         |
| 7    | `terraform destroy`                                         | Remove everything Part B created            |

<br>

## Reasoning

### S8.1 and S8.2: phases that each prove one thing

- **One new failure source per phase:** when a check fails, the cause is in the code added in that phase.
- **Local before cloud:** a Python bug found by pytest takes seconds to fix. The same bug found through a cloud deploy takes a build, an upload and a log search.

### S8.4: two rebuild tests

- **Phase 2:** proves early that the backend and `destroy` settings work, while there is little to lose.
- **Phase 8:** proves the promise of D1.3: the whole system can be rebuilt from the repository and the laptop's files.

### S8.5: branch per phase

- **CI on every phase:** each pull request runs the workflows from step 7.
- **Readable history:** a reviewer can follow the build phase by phase in the closed pull requests.

### S8.6: data folder as an argument

- **Same script for both datasets:** dummy data for development, the full dataset for phase 8.
- **Safe default:** running the script without an argument never touches the private dataset.

### S8.7: data checks in SQL

One query, built from `UNION ALL` blocks. Each block returns one row: `check_name`, `table_name`, `result`.

| Check                | What it counts                                                             | Expected                   |
| -------------------- | -------------------------------------------------------------------------- | -------------------------- |
| Row count            | Rows per staging table                                                     | The source file's record count |
| Duplicate keys       | `_id` values that appear more than once, per table                         | 0                          |
| Orphan references    | Blog post `categoryIds`, `tagIds`, `memberId` and article `category` values with no match in their lookup table | 0 |
| Missing titles       | Blog posts and articles with an empty or `NULL` title                      | 0                          |

- **Why:** row counts prove that everything loaded; the other checks prove that the joins in the cleaned views will work. Data quality checks are a core Data Engineering habit and cost a few lines of SQL.
- **Read the result:** every row except the row counts must show 0. A non-zero row names the check and the table.
- **Placeholder-free:** the file uses the dataset name `content_staging` directly. The project comes from the `bq` default project, so no ID is committed.
- **Added to the repository tree:** `scripts/data_checks.sql`.

### S8.8: README results from the full dataset

- **Why:** realistic titles, tags and numbers show what the pipeline produces. Placeholder text from the dummy data would not.
- **What becomes public:** query output only (for example titles in `top_posts_per_category`, tag names in `tag_frequency`). The full dataset files stay git-ignored and are never committed.
- **When:** the results are captured in phase 8, before the final `destroy`.

<br>

## Optional extensions

| Extension                                   | Source |
| ------------------------------------------- | ------ |
| Looker Studio report on the analysis views  | D0.2   |
| Refactor the function to a Cloud Run service with a Dockerfile | D1.8 |
| `plan` on pull requests with keyless login  | S7.1   |
| `time_sleep` after the IAM grants           | S5.3   |
| Pre-commit hooks                            | S7.8   |
| Security scanner, SQL linter, `terraform-docs` | Step 7 |

<br>

## Open questions

- None. All 8 planning steps are complete.
