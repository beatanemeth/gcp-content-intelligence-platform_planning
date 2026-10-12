# Step 3: resource inventory

> **Reviewed:** [step-3.1_storage.md](../v3_review_plan/step-3.1_storage.md), [step-3.2_function.md](../v3_review_plan/step-3.2_function.md) and [step-3.3_bigquery.md](../v3_review_plan/step-3.3_bigquery.md) correct this step. Where they differ, the review wins.

Decision log for step 3 of [00_roadmap.md](00_roadmap.md). Builds on [01_platform-contract.md](01_platform-contract.md) and [02_module-interface.md](02_module-interface.md).

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

The module creates 22 resources before IAM: 2 buckets, 1 zip upload, 1 function, 3 datasets, 6 staging tables and 9 views. One data source zips the function code.

Both buckets block public access and use IAM only. Neither keeps old or deleted files, because cloud data is disposable (D1.3).

The function runs at most 2 instances (one per file group) and accepts traffic from inside Google Cloud only. Terraform hands it the names it needs (project, staging dataset, location) as environment variables, so no name is written twice.

Resources of the same shape and purpose share one block with `for_each`: datasets, staging tables and views. Views are split into one block per layer. Each view block waits for the layer it reads with `depends_on`, because Terraform cannot see table names inside SQL text.

One analysis view measures the cleaned article text, so the HTML cleaning in Python has a visible result.

<br>

## Decisions

| ID    | Decision                                                                                                      | Status    |
| ----- | ------------------------------------------------------------------------------------------------------------- | --------- |
| S3.1  | Both buckets: uniform bucket-level access, public access prevention enforced.                                 | Delegated |
| S3.2  | No object versioning and no soft delete on either bucket.                                                     | Delegated |
| S3.3  | Function: at most 2 instances, minimum 0, internal ingress only.                                              | Delegated |
| S3.4  | `for_each` for resources of the same shape and purpose: datasets, staging tables, views per layer.            | Delegated |
| S3.5  | Separate blocks for the two buckets: different purposes, read separately.                                     | Delegated |
| S3.6  | Views split into one block per layer; the unified `content` view has its own block.                           | Delegated |
| S3.7  | View SQL uses only `${project}` and dataset placeholders; each view block has `depends_on` on the layer it reads. | Delegated |
| S3.8  | The API list in S1.4 is confirmed after adding the Cloud Resource Manager API.                                | Delegated |
| S3.9  | Terraform passes `PROJECT_ID`, `STAGING_DATASET` and `BQ_LOCATION` to the function as environment variables.   | Delegated |
| S3.10 | A sixth analysis view, `article_text_stats`, measures the cleaned article text.                               | Delegated |
| S3.11 | Each analysis SQL file starts with a comment that defines its metric.                                         | Delegated |

<br>

## Inventory

### Storage

| Resource                | Terraform type          | Name                          | Key arguments                                                      | From            |
| ----------------------- | ----------------------- | ----------------------------- | ------------------------------------------------------------------ | --------------- |
| Data bucket             | `google_storage_bucket` | `<project-id>-content-data`   | `location`, `force_destroy = true`, S3.1, S3.2, labels             | D1.4, D1.7      |
| Source bucket           | `google_storage_bucket` | `<project-id>-content-source` | Same as the data bucket                                            | D1.12           |

### Function code

| Resource                | Terraform type                  | Name                                   | Key arguments                                              | From  |
| ----------------------- | ------------------------------- | -------------------------------------- | ---------------------------------------------------------- | ----- |
| Zip of the code         | `data "archive_file"`           | Not in GCP; a local file               | `source_dir` = the module's function folder                | D1.11 |
| Zip upload              | `google_storage_bucket_object`  | `function-source-<md5 of zip>.zip`     | `bucket` = source bucket, `source` = the zip               | D1.11 |

**Terraform background:** a **data source** (`data` block) reads or computes something without creating it in the cloud. `archive_file` builds the zip on the laptop during `plan`. It is the only item in this inventory that never appears in GCP.

### Function

| Resource  | Terraform type                     | Name                | Key arguments                                                                                       | From       |
| --------- | ---------------------------------- | ------------------- | --------------------------------------------------------------------------------------------------- | ---------- |
| Function  | `google_cloudfunctions2_function`  | `content-transform` | `build_config` (Python runtime, entry point, zip), `service_config` (S3.3, runtime service account, environment variables S3.9), `event_trigger` | D1.5, D1.8 |

`event_trigger` settings:

| Argument                | Value                                         | From  |
| ----------------------- | --------------------------------------------- | ----- |
| `event_type`            | `google.cloud.storage.object.v1.finalized`    | D1.5  |
| `event_filters`         | `bucket` = the data bucket                    | D1.5  |
| `trigger_region`        | `var.region` (same as the bucket)             | D3.2  |
| `retry_policy`          | `RETRY_POLICY_RETRY`                          | D1.10 |
| `service_account_email` | The trigger's service account (step 4)        | S1.8  |

`service_config.environment_variables` (S3.9):

| Variable          | Value                                    | Used by the code for                                  |
| ----------------- | ---------------------------------------- | ----------------------------------------------------- |
| `PROJECT_ID`      | `var.project_id`                         | The BigQuery client's project                         |
| `STAGING_DATASET` | The `content_staging` dataset's ID       | The load jobs' destination dataset                    |
| `BQ_LOCATION`     | `var.region`                             | The load jobs' location (must match the dataset, D3.2) |

### BigQuery

| Resource       | Terraform type             | Count | Names                                                                                              | Key arguments                                                       | From        |
| -------------- | -------------------------- | ----- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ----------- |
| Datasets       | `google_bigquery_dataset`  | 3     | `content_staging`, `content_cleaned`, `content_analysis`                                           | `location`, `delete_contents_on_destroy = true`, labels with `layer` | D3.1, S2.2  |
| Staging tables | `google_bigquery_table`    | 6     | `blog_posts`, `blog_categories`, `blog_tags`, `members`, `articles`, `article_categories`          | `schema = file(...)`, `deletion_protection = false`                 | D3.3, D3.4  |
| Cleaned views  | `google_bigquery_table`    | 2     | `blog_posts`, `articles`                                                                           | `view` block with `templatefile(...)`, `use_legacy_sql = false`, `depends_on` (S3.7) | D2.4, D3.5  |
| Unified view   | `google_bigquery_table`    | 1     | `content`                                                                                          | Same as cleaned views                                               | D3.11       |
| Analysis views | `google_bigquery_table`    | 6     | `content_per_month`, `tag_frequency`, `engagement_by_category`, `content_running_total`, `top_posts_per_category`, `article_text_stats` | Same as cleaned views                   | D3.13, S3.10 |

**Terraform background:** a BigQuery view is a `google_bigquery_table` with a `view` block instead of a `schema`. BigQuery itself treats views as a kind of table.

### Count

| Component       | Resources |
| --------------- | --------- |
| Storage         | 2         |
| Function code   | 1 (plus 1 data source) |
| Function        | 1         |
| BigQuery        | 3 + 6 + 2 + 1 + 6 = 18 |
| **Total**       | **22**, plus IAM (step 4) |

<br>

## Reasoning

### S3.1: bucket security

- **`uniform_bucket_level_access = true`:** access is controlled by IAM only. Per-object ACLs (an older, second permission system) are switched off.
- **`public_access_prevention = "enforced"`:** no object can ever be made public, even by mistake.
- **Why on a demo:** both cost nothing, and a reviewer checks for them first. Many organisations enforce them with an organisation policy.

### S3.2: no versioning, no soft delete

- **Versioning off:** every overwrite (D1.10) would keep the old file. The laptop is the source of truth (D1.3), so old cloud copies have no value.
- **Soft delete off:** by default, Cloud Storage keeps deleted objects for 7 days and bills for them. `soft_delete_policy { retention_duration_seconds = 0 }` turns this off.
- **Source bucket:** each code change adds a new zip (hash in the name). The zips are a few kilobytes; no cleanup rule is needed.

### S3.3: function sizing and ingress

| Setting              | Value                          | Why                                                                                         |
| -------------------- | ------------------------------ | ------------------------------------------------------------------------------------------- |
| `max_instance_count` | 2                              | One per file group. Caps cost if a bug causes a loop of events.                             |
| `min_instance_count` | 0                              | Scales to zero between uploads; no idle cost.                                               |
| `ingress_settings`   | `ALLOW_INTERNAL_ONLY`          | Eventarc delivers events from inside Google Cloud. Nothing on the internet can call the function. |
| `available_memory`   | `256M`                         | 570 KB of data; HTML parsing needs little memory.                                           |
| `timeout_seconds`    | 300                            | The function waits for 6 load jobs to finish so it can log row counts.                       |

- **Events per upload:** about 14 (6 raw files, 2 markers, 6 processed files). Only the 2 markers start real work; the rest exit at once (not `_READY`).
- **When the limit is reached:** an extra event is rejected and delivered again later. Expect some rejections and retries in the logs. The retry policy and overwriting (D1.10) make them safe.

### S3.4 and S3.5: when to use `for_each`

**Terraform background:** `for_each` takes a map and creates one resource per entry, from a single block. Inside the block, `each.key` is the entry's name (for example `blog_posts`) and `each.value` its value. Each instance has its own address in state, such as `google_bigquery_table.staging["blog_posts"]`.

| Group           | `for_each` | Why                                                                                  |
| --------------- | ---------- | ------------------------------------------------------------------------------------ |
| Datasets        | Yes        | Same shape; only the name, `layer` label and description differ. `each.key` becomes the `layer` label. |
| Staging tables  | Yes        | Same shape; only the name and schema file differ.                                    |
| Views per layer | Yes        | Same shape; only the name and SQL file differ.                                       |
| Buckets         | No         | Different purposes. Two named blocks read more clearly than a map.                   |

- **Rule of thumb:** `for_each` when items are interchangeable copies; separate blocks when each item has its own role.
- **Why a map, not `count`:** with `count`, items are numbered. Removing the second of six renumbers the rest, and Terraform destroys and recreates them. With a map, each item keeps its key.

### S3.6 and S3.7: view blocks and creation order

**The problem:** BigQuery checks a view's SQL when the view is created. Every table or view the SQL reads must already exist. Terraform creates the instances of one `for_each` block in parallel, so a view cannot safely read another view from the same block.

| Block                    | Reads                                 | Views                            | `depends_on`                          |
| ------------------------ | ------------------------------------- | -------------------------------- | ------------------------------------- |
| `staging` tables         | Nothing (filled by load jobs)          | 6 tables                         | None                                  |
| `cleaned` views          | Staging tables                         | `blog_posts`, `articles`         | `staging` tables                      |
| `content` view           | Cleaned `blog_posts` and `articles`    | `content`                        | `cleaned` views                       |
| `analysis` views         | Cleaned views and `content`            | 6 views                          | `cleaned` views, `content` view       |

- **Why `content` is separate:** it reads the two other cleaned views. In the same block, it might be created before them.
- **Placeholders (S3.7):** the SQL files use only `${project}` and dataset placeholders such as `${staging_dataset}`. Table and view names are written directly in the SQL, so the files read like normal BigQuery SQL.
- **Why `depends_on` here:** Terraform builds its creation order from references in `.tf` files. It cannot see the table names inside SQL text. The dataset placeholder makes a view wait for the dataset, not for the tables inside it. `depends_on` on the whole previous block closes that gap.
- **Rejected alternative:** passing each table's ID from its resource into `templatefile()`. Terraform would infer the order, but the SQL would fill with expressions such as `${staging_tables["blog_posts"]}` and become harder to read.
- **Full creation order:** step 5.

### S3.8: API list confirmed

- **Added:** `cloudresourcemanager.googleapis.com`. The 4 project-level grants in step 4 (`google_project_iam_member`) read and update the project's IAM policy through this API.
- **Everything else** maps to an API already in S1.4. Load jobs (started by the function) use the BigQuery API. Logging uses the Cloud Logging API, which is on by default in every project.

### S3.9: Terraform to Python hand-over

- **The problem:** Terraform knows the names it created; the Python code does not. Writing `content_staging` in Python as well would repeat the prefix that S2.2 defines once.
- **The mechanism:** environment variables in `service_config`. The code reads them with `os.environ["STAGING_DATASET"]` at start-up.
- **What is not passed:** the bucket and object name arrive with each event. Table names come from the file names (D2.5).
- **Testability:** the unit tests (phase 3) set the same variables, so the code runs the same way locally and in the cloud.

### S3.10: article text statistics

- **Why:** the HTML cleaning is the main reason the function exists (D1.6, D2.2), but no other view reads the article text. This view shows the cleaning paid off.
- **Content:** word count and estimated reading time per article, and their average per category.
- **SQL techniques:** `SPLIT` and `ARRAY_LENGTH` for word counts, `GROUP BY` for the category averages.
- **Requirement:** the cleaned `articles` view exposes the cleaned text as a column (`article_text`, D3.6).

### S3.11: metric definitions

- **Rule:** every analysis SQL file starts with a comment that states what the view measures and how (for example, which `metrics` fields make up "engagement").
- **Why:** a number without a definition cannot be checked or defended. The comment also documents the view on GitHub.

<br>

## Trade-off accepted: a group can be half-updated

- **What happens:** the function starts one load job per file. If 1 of the 4 blog loads fails, staging briefly mixes new tables with an old one, and the cleaned views join across both.
- **Why it is accepted:** the event is retried, and every load overwrites (D1.10). The next successful run makes the group consistent again. Each upload is a full snapshot of fictional data, with no downstream consumers waiting.
- **Production alternative:** load into temporary tables first, then replace all tables of the group at once. More code, and more to explain, for a problem this demo does not have.

<br>

## Changes to earlier decisions

| Earlier | Was                                                       | Now                                                     |
| ------- | --------------------------------------------------------- | ------------------------------------------------------- |
| D3.13   | 5 analysis views                                          | 6 analysis views, with `article_text_stats` (S3.10)     |
| D3.14   | `.sql` files with placeholders; creation order not decided | Placeholders for project and datasets only; `depends_on` between view blocks (S3.7) |

<br>

## Open questions

- None. The build folder for the zip was settled in step 6 (S6.5).
