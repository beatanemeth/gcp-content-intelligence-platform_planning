# Step 5b: function, trigger, upload script

**Branch:** `feature/function-and-trigger`

<br>

## In plain words

The moving part of the pipeline: a file upload starts the function, and the function fills the staging tables.

- **Terraform deploys the code:** it zips `files/function/`, uploads the zip to the source bucket, and creates the function. No manual `gcloud` step.
- **The function** (`content-transform`, a Cloud Run function) runs the cleaning from step 3, writes NDJSON to `processed/`, and loads each file into its staging table.
- **The trigger:** every finished upload in the data bucket sends an event. The function acts only on a `_READY` marker; every other event ends at once.
- **Why a marker:** a group has several files. The marker, uploaded last, says "the group is complete", so the group is processed once, not once per file.
- **The upload script** (`helpers/upload_data.sh`) uploads each group's files, then its marker. It changes the cloud, so in the simulation it never runs; ShellCheck checks it.
- **The 8th grant,** `run.invoker`, lets the trigger account call the function. It needs the function's service, so it comes with the function.
- **The simulated check:** `terraform test` for the function and trigger; pytest runs the entry point on fake events with fake cloud clients.

<br>

## Technical details

### One upload, 14 events

| Upload                                 | Events | What `handle_event` does                |
| -------------------------------------- | ------ | --------------------------------------- |
| 6 export files in `raw/<group>/`       | 6      | Ends at once: not a marker              |
| 2 markers, `raw/blogs/_READY` and `raw/articles/_READY` | 2 | Processes that group               |
| 6 NDJSON files the function writes to `processed/` | 6 | Ends at once: same bucket, so they send events too |

### Versions

Add the `archive` provider: `>= 2.0` in the module's `versions.tf`, `~> 2.8` in the root module's. Run `terraform init` in `environments/dev/`; the lock file gains the new provider. Commit it.

### `function.tf`

**1. The zip:** `data "archive_file" "function"`.

| Argument      | Value                                         | Why                                               |
| ------------- | --------------------------------------------- | ------------------------------------------------- |
| `source_dir`  | `"${path.module}/files/function"`             | The code from step 3                              |
| `output_path` | `"${path.root}/.build/content-function.zip"`  | Built on the laptop at `plan`; `.build/` is git-ignored |
| `excludes`    | `["__pycache__"]`                             | No Python cache in the zip                        |

**2. The upload:** `google_storage_bucket_object.function_source`.

- **`name = "function-source-${data.archive_file.function.output_md5}.zip"`:** the hash is in the name. A code change gives a new name, so Terraform replaces the object and redeploys the function. With a fixed name, Terraform would not see the change.
- **`bucket`:** the source bucket.

**3. The function:** `google_cloudfunctions2_function.transform`, name `content-transform`.

| Block            | Argument                | Value                              | Why                                           |
| ---------------- | ----------------------- | ---------------------------------- | --------------------------------------------- |
| `build_config`   | `runtime`               | `python314`                        | The newest Python runtime                     |
|                  | `entry_point`           | `handle_event`                     | The function in `main.py` that gets the event |
|                  | `source.storage_source` | Source bucket and object           | Where the build reads the zip                 |
|                  | `service_account`       | Build account's **`id`**           | The build wants the full resource name, not the email |
| `service_config` | `max_instance_count`    | `2`                                | Caps cost; rejected events come back through retries |
|                  | `min_instance_count`    | `0`                                | Nothing runs, nothing costs, while idle       |
|                  | `available_memory`      | `256M`                             | Enough for about 130 records                  |
|                  | `timeout_seconds`       | `300`                              | The function waits for its load jobs          |
|                  | `ingress_settings`      | `ALLOW_INTERNAL_ONLY`              | Only Google Cloud (Eventarc) can reach it, not the internet |
|                  | `service_account_email` | Runtime account's email            | The identity the code runs as                 |
|                  | `environment_variables` | `PROJECT_ID`, `STAGING_DATASET`, `BQ_LOCATION` | Settings come from Terraform, never hard-coded in `main.py` |
| `event_trigger`  | `event_type`            | `google.cloud.storage.object.v1.finalized` | A file finished uploading             |
|                  | `trigger_region`        | `var.region`                       | Same region as the bucket                     |
|                  | `event_filters`         | `bucket` = data bucket             | Only this bucket's uploads                    |
|                  | `retry_policy`          | `RETRY_POLICY_RETRY`               | Events rejected at the 2-instance limit come back |
|                  | `service_account_email` | Trigger account's email            | The identity that delivers the event          |

**`depends_on`:** the build account's 3 grants and the trigger account's `eventReceiver` grant.

- **Why needed:** Terraform orders resources by their references. The function refers to the accounts, not to their grants. Without `depends_on`, Terraform could create the function before the grants exist, and the build would fail.
- **Still possible in real GCP:** a grant can take a minute to take effect. If the first `apply` fails on a permission, run `apply` again. No `time_sleep`: simpler code, and it happens at most once.

**4. The 8th grant:** `google_cloud_run_v2_service_iam_member.trigger_invoker`.

- **`name`:** `google_cloudfunctions2_function.transform.service_config[0].service`, the Cloud Run service behind the function. It exists only after the function, so the reference also sets the order.
- **`role`:** `roles/run.invoker`, for the trigger account.

**Output:** `function_name`, in the module and passed on in `environments/dev/outputs.tf`. Used to filter the logs after an upload.

### `main.py`: the entry point

**New constants:**

- **`MAX_EVENT_AGE = 15 minutes`:** see `is_too_old`.
- **`LOAD_CONFIG`:** `NEWLINE_DELIMITED_JSON`, `WRITE_TRUNCATE`. No schema given: BigQuery uses the table's own strict schema from step 4.

**New functions:**

| Function       | Does                                                                                         |
| -------------- | -------------------------------------------------------------------------------------------- |
| `ready_group`  | Returns the group for `raw/<group>/_READY`; `None` for any other file. A marker in the wrong place (such as `raw/videos/_READY`) logs a `WARNING` and returns `None` |
| `is_too_old`   | True when the event's `time` is more than 15 minutes before now                              |
| `process_file` | Download `raw/<group>/<name>.json`; `transform`; overwrite `processed/<group>/<name>.ndjson`; `load_table_from_uri` into staging; wait; log `INFO` with records written and rows loaded |
| `handle_event` | `@functions_framework.cloud_event`. Not a marker: return. Too old: log `WARNING`, return. Otherwise: create the Storage and BigQuery clients, then `process_file` for each file of the group |

- **Why 15 minutes:** a retry keeps the event's original time, so its age grows. A temporary problem recovers within minutes. An older event is a permanent failure; returning without an error ends Eventarc's retries, which would otherwise go on for days.
- **Why a misplaced marker only warns:** an unknown group would raise an error, and the event would be retried for 15 minutes for nothing.
- **Why `load_table_from_uri`:** BigQuery reads the file from Cloud Storage itself; loading is free.
- **Python 3.14:** `datetime.fromisoformat` reads the `Z` in Cloud Storage's event time directly. No workaround needed.

**`requirements.txt`:** add `functions-framework`, `google-cloud-storage`, `google-cloud-bigquery`, pinned (first build: 3.10.2, 3.17.0, 3.45.2). Remove `google-cloud-bigquery` from `requirements-dev.txt`; it now arrives through `-r`.

### pytest: `tests/function/test_handle_event.py`

- **Fake clients:** small classes replace the Storage and BigQuery clients. The fake bucket holds the dummy exports under `raw/<group>/`; the fake load job counts the lines of each processed file as its loaded rows.
- **Fake events:** the attributes Eventarc sends (`type`, `source`, `time`) and the `bucket` and `name` data. Each test sets the event's age.

| Test                               | Proves                                                              |
| ---------------------------------- | ------------------------------------------------------------------- |
| `ready_group`                      | `raw/blogs/_READY` gives `blogs`; a data file or a `processed/` file gives `None` |
| `ready_group`, misplaced markers   | `raw/videos/_READY`, `blogs/_READY`, `_READY`: `None` and a `WARNING` |
| `is_too_old` at the limit          | Exactly 15 minutes: false; 1 millisecond more: true                 |
| A data file event                  | Stops at once: nothing written, nothing loaded, nothing logged      |
| An old marker event                | Stops with a `WARNING`; nothing written                             |
| A fresh marker event, per group    | Writes each NDJSON file; loads each table with `WRITE_TRUNCATE` and no schema; logs records = rows per file |
| The same event twice               | The same files: overwritten, not appended                           |

### Simulated test: `tests/function.tftest.hcl`

`run "function_and_trigger"`, `command = apply`. The `archive` provider is **real**: it zips the code on the laptop. Only Google is mocked.

**Mocks:** `mock_resource` defaults for all accounts, plus an `override_resource` for the trigger account, so the `run.invoker` check can tell it apart.

**Assertions:**

- The zip: in the source bucket, with its hash in the name.
- The function: name, project, region, labels, `python314`, `handle_event`, the zip as source.
- Each of build, runtime and trigger uses its own account.
- 2 instances at most, 0 at least, internal ingress, 256M, 300 seconds.
- The 3 environment variables.
- The trigger: event type, region, retries on, the data bucket filter.
- `run.invoker` for the trigger account, on the function's service.
- The `function_name` output.

### Upload script: `helpers/upload_data.sh`

1. `set -euo pipefail`: stop at the first error, an unset variable, or a failed pipe.
2. Data folder as the first argument; default `data/dummy`.
3. Check that `blogs/` and `articles/` exist.
4. Bucket name: `terraform -chdir=terraform/environments/dev output -raw data_bucket_name`. No name is typed twice.
5. For each group:
   - `gcloud storage cp <folder>/<group>/*.json gs://<bucket>/raw/<group>/`
   - then, as a **separate command**, the empty marker: `gcloud storage cp - gs://<bucket>/raw/<group>/_READY < /dev/null`
6. Print the command that shows the function's name.

- **Why a separate command for the marker:** `gcloud storage cp` uploads several files in parallel, in no fixed order. A marker in the same command could land before a data file.
- **Variable `DATA_GROUPS`, not `GROUPS`:** bash keeps the user's group IDs in `GROUPS` and ignores assignments to it.
- **Safe to run again:** files and markers are overwritten; the loads replace the rows.

**Simulated once, not committed:** put fake `terraform` and `gcloud` scripts first on `PATH` that only print their arguments. Run the script; check the order: each group's files, then its marker. Make the fake `gcloud` fail once; the script must stop before the marker.

### ShellCheck

- **pre-commit:** add `shellcheck-py/shellcheck-py` (`v0.11.0.1-1`), hook `shellcheck`. Installed through pip, so commits do not depend on Docker.
- **CI:** `.github/workflows/shell-checks.yml`. Runs when `helpers/*.sh` or the workflow changes; `permissions: contents: read`; `shellcheck helpers/*.sh` (preinstalled on the runner).

### Notes in the files

| File                   | Note                                                                    |
| ---------------------- | ----------------------------------------------------------------------- |
| `function.tf`          | Terraform zips, uploads and deploys; the hash in the object name; build account by `id`; why 2 instances and internal ingress; settings as environment variables; the trigger fires on every upload, `handle_event` keeps only `_READY`; why retries; why `depends_on`; `run.invoker` after the function |
| `main.py`              | Only the marker means complete; why 15 minutes; returning ends retries; overwrite, so a repeated event gives the same result; no schema in the load, so the table's strict one applies |
| `upload_data.sh`       | Header: what it does, when to run it, usage; marker in a separate command; why not `GROUPS` |
| `function.tftest.hcl`  | The archive provider is real; only Google is mocked                     |

### `simulation/README.md`

Add the step 5b row, the fake clients and fake events, and the upload-order check. What it cannot prove: real Eventarc delivery and retries, real load jobs, the function's real runtime limits.

### Docs

Each document grows with the step.

| Document            | Adds                                                                                |
| ------------------- | ----------------------------------------------------------------------------------- |
| `README.md`         | **How it works:** the diagram (`assets/content_pipeline_overview.png`) and the 5 stages in plain words; the upload script in one line (never runs here) |
| Module `README.md`  | The function and its trigger; the `function_name` output                            |
| `docs/decisions.md` | Laptop upload, not the Wix API; the `_READY` marker; a Cloud Run function built from source; the 15-minute limit; 2 instances and internal ingress; no `time_sleep` after the grants |

### Commits

1. `build: add the archive provider`
2. `feat(function): add handle_event with the _READY check and the 15-minute limit`
3. `test(function): run handle_event on fake events with fake clients`
4. `feat(terraform): zip and deploy the function with its trigger`
5. `feat(terraform): let the trigger account invoke the function`
6. `feat(terraform): add function_name output`
7. `test(terraform): add simulated function test with mocked provider`
8. `feat(scripts): add the upload script, marker last`
9. `ops(ci): add ShellCheck to pre-commit and a shell checks workflow`
10. `docs: add how it works, the diagram and the function decisions`

<br>

## Done when

- `terraform test`: 4 runs pass (buckets; datasets and tables; accounts and grants; function and trigger).
- `pytest` passes, including `test_handle_event.py`.
- `shellcheck helpers/upload_data.sh` reports nothing.
- The fake `PATH` run shows each group's files before its marker.
- **Proof that a test can fail:** set `retry_policy` to `RETRY_POLICY_DO_NOT_RETRY`; the function test fails. Undo the change.
- `pre-commit run --all-files` and all 3 workflows are green.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
