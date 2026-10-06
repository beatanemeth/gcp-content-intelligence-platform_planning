# Step 1: raw data upload

Decision log for step 1 of [03_bea_answers_and_thoughts.md](03_bea_answers_and_thoughts.md).

**Status values:** _Agreed_ (confirmed by the maintainer).

<br>

## Summary

The data is fictional, so no API is called. The upload from the laptop replaces the Wix API fetch of the original project (D1.1). The raw files on the laptop are the source of truth; everything in the cloud is disposable (D1.3). Terraform builds the infrastructure; a Bash script uploads the data with `gcloud storage cp` (D1.2). `terraform destroy` deletes the buckets and datasets together with their contents (D1.4).

All pipeline data lives in one bucket, organised by prefixes (D1.7). The upload script copies the data files of a group first and a `_READY` marker last (D1.9). Every new file fires an event (D1.5) that starts a Cloud Run function (D1.8). The function ignores everything except `_READY`; then it reshapes the group's files into NDJSON and cleans the HTML (D1.6). Every run overwrites its previous output (D1.10). Terraform also deploys the function code, from a separate source bucket (D1.11, D1.12).

<br>

## Decisions

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

## Reasoning

### D1.2 to D1.4: data stays out of Terraform

**Principle:** Terraform manages infrastructure; a script manages data.

| Concern               | `gcloud storage cp`                    | `google_storage_bucket_object`                                 |
| --------------------- | -------------------------------------- | -------------------------------------------------------------- |
| **`apply` side effects** | `apply` creates infrastructure only | `apply` also uploads data and starts the pipeline              |
| **Ordering**          | Upload happens after `apply` by design | Files may land before the trigger exists                       |
| **State**             | Data is not tracked in Terraform state | Every file becomes a resource in state                         |

**Disposable data:** the rebuild path is `apply`, then upload. Terraform owns the containers, and destroying a container deletes its contents:

| Resource                  | Argument                            | Effect on `destroy`                                          |
| ------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| `google_storage_bucket`   | `force_destroy = true`              | Deletes all objects, including those uploaded by `gcloud`    |
| `google_bigquery_dataset` | `delete_contents_on_destroy = true` | Deletes all tables, including those created by load jobs     |
| `google_bigquery_table`   | `deletion_protection = false`       | Allows Terraform-managed tables to be deleted (default `true`) |

- **Why `force_destroy = true`:** the data is disposable (D1.3), and frequent teardowns would otherwise stop at every non-empty bucket. With data that has no other copy, `false` is the safer choice: it also blocks an `apply` that would replace the bucket (for example after a name or location change) and delete its contents.

### D1.5 to D1.8: event-driven Cloud Run function

- **Event-driven (D1.5):** an automatic trigger shows how a real pipeline works, even for a one-time upload.
- **One processing step (D1.6):** Python reshapes and cleans; loading and SQL modelling follow in steps 2 and 3.
- **One bucket (D1.7):** the upload writes `raw/`; the function writes `processed/`.
- **Function, not service (D1.8):** a Cloud Run function is a Cloud Run service whose container Google builds. No Dockerfile and no image push; Terraform deploys from source (`google_cloudfunctions2_function` with an `event_trigger` block). A Cloud Run service with its own Dockerfile stays an optional refactor; the transformation code does not change.

### D1.9: marker file

**Problem:** Cloud Storage emits one event per file, and the trigger filters by bucket, not by file name. A run may start when only part of a group has landed. The SQL joins need every file of a group.

| Group        | Files                                                                       | Count |
| ------------ | --------------------------------------------------------------------------- | ----- |
| **Blogs**    | `blog_categories.json`, `blog_posts.json`, `blog_tags.json`, `members.json` | 4     |
| **Articles** | `article_categories.json`, `articles.json`                                  | 2     |

**Upload sequence:** each command waits for the previous one.

| Order | Command uploads                        | Destination           |
| ----- | -------------------------------------- | --------------------- |
| 1     | The 4 blog data files                  | `raw/blogs/`          |
| 2     | The blog marker                        | `raw/blogs/_READY`    |
| 3     | The 2 article data files               | `raw/articles/`       |
| 4     | The article marker                     | `raw/articles/_READY` |

- **Stop on failure:** `set -e` ends the script at the first failed command, so a failed upload never produces a marker.
- **Marker kept apart:** the local marker file sits outside the data folders, so a wildcard cannot upload it early.
- **No self-triggering:** the function's output is never named `_READY`.
- **Rejected alternative:** list the prefix on every event and proceed when all files are present. Two files landing at the same moment would both see a complete group: a race condition.

### D1.10: overwrite, never append

- **Why:** a group can be processed twice: a re-upload, a retry, or a duplicate event (Eventarc delivers **at least once**).
- **Rule:** each run replaces its previous output, so a second run gives the same result (**idempotency**). In BigQuery, the load uses `WRITE_TRUNCATE`, not `WRITE_APPEND`.

### D1.11 and D1.12: Terraform deploys the function code

Code defines the function; data only flows through it. So Terraform uploads code (D1.11) but not data (D1.2).

| Order | Terraform piece                             | Job                                                         |
| ----- | ------------------------------------------- | ----------------------------------------------------------- |
| 1     | `data "archive_file"` (`hashicorp/archive`) | Zips the local function folder                              |
| 2     | `google_storage_bucket_object`              | Uploads the zip; the object name contains the zip's hash    |
| 3     | `google_cloudfunctions2_function`           | Builds and deploys the function from the zip                |

- **Hash in the name:** Terraform compares configuration, not file contents. A fixed name such as `function.zip` hides code changes; a hash-based name changes with the code and triggers a redeploy.
- **Separate source bucket (D1.12):** a zip in the data bucket would fire the trigger on every deploy. Data and deployment artefacts stay apart.
- **Rejected alternative:** `gcloud functions deploy`. The function would sit outside Terraform: `destroy` would miss it, and the repository would not describe the full system.
