# Step 4: IAM

> **Reviewed:** [step-4_iam.md](../v3_review_plan/step-4_iam.md) corrects this step. Where they differ, the review wins.

Decision log for step 4 of [00_roadmap.md](00_roadmap.md). Builds on [01_platform-contract.md](01_platform-contract.md) and [03_resource-inventory.md](03_resource-inventory.md).

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

The pipeline has three jobs done by machines: **build** the function, **trigger** it, and **run** it. Each job gets its own service account, created by the module (S1.8).

Every role is granted at the smallest scope GCP allows: a bucket or a dataset where possible, the project only where the role has no narrower scope.

The module uses additive IAM resources only. It adds its own grants and never touches anyone else's, which keeps Part A and other features safe in the shared project.

The maintainer's own access belongs to Part A. No service account keys exist anywhere.

<br>

## Decisions

| ID   | Decision                                                                                                    | Status    |
| ---- | ----------------------------------------------------------------------------------------------------------- | --------- |
| S4.1 | Three service accounts in Part B: `content-build`, `content-trigger`, `content-runtime`.                    | Delegated |
| S4.2 | Each role is granted at the smallest scope available: bucket, dataset, service, then project.               | Delegated |
| S4.3 | Only additive IAM resources (`*_iam_member`); never `*_iam_policy` or `*_iam_binding`.                       | Delegated |
| S4.4 | No service account keys. Services run as their attached service account.                                     | Delegated |
| S4.5 | The maintainer holds `Owner` on the project (Part A) and runs Terraform with their own `gcloud` login.       | Delegated |
| S4.6 | No prefix-level conditions on the data bucket; the runtime account can access the whole bucket.             | Delegated |

<br>

## Identities and roles

### Overview

| Service account   | Job                                                       | Attached in the function at              |
| ----------------- | --------------------------------------------------------- | ---------------------------------------- |
| `content-build`   | Cloud Build turns the zip into a container image          | `build_config.service_account`           |
| `content-trigger` | Eventarc delivers the storage event to the function       | `event_trigger.service_account_email`    |
| `content-runtime` | The function code reads, writes and starts load jobs      | `service_config.service_account_email`   |

### Roles

| Service account   | Role                              | Scope                              | Why                                                                     |
| ----------------- | --------------------------------- | ---------------------------------- | ----------------------------------------------------------------------- |
| `content-runtime` | `roles/storage.objectUser`        | Data bucket                        | Read `raw/`, write and overwrite `processed/` (D1.10)                    |
| `content-runtime` | `roles/bigquery.dataEditor`       | `content_staging` dataset          | Replace the staging tables' rows with `WRITE_TRUNCATE` (D2.3)            |
| `content-runtime` | `roles/bigquery.jobUser`          | Project                            | Start load jobs. Jobs belong to a project; no narrower scope exists.     |
| `content-trigger` | `roles/run.invoker`               | The function's Cloud Run service   | Call the function with the event                                        |
| `content-trigger` | `roles/eventarc.eventReceiver`    | Project                            | Receive events from Eventarc. No narrower scope exists.                  |
| `content-build`   | `roles/storage.objectViewer`      | Source bucket                      | Read the zip                                                            |
| `content-build`   | `roles/artifactregistry.writer`   | Project                            | Push the image to `gcf-artifacts` (S1.6). Google creates that repository on the first deploy, so it cannot be targeted in advance. |
| `content-build`   | `roles/logging.logWriter`         | Project                            | Write build logs                                                        |

**Totals:** 3 service accounts and 8 role grants: 11 IAM resources. With the 22 resources of step 3, the module creates **33 resources**.

**API note:** the 4 project-level grants go through the Cloud Resource Manager API, which Part A enables (S1.4, S3.8).

**Not needed:**

- **`content-runtime` and logs:** Cloud Run collects anything the code prints to standard output. No logging role is required.
- **`content-runtime` and the load job's file access:** a load job reads the NDJSON files with the permissions of whoever starts it. `roles/storage.objectUser` already covers that.
- **`content-runtime` and the cleaned or analysis datasets:** the function never touches them. Views read staging at query time with the querying user's permissions.

<br>

## Reasoning

### S4.1: three service accounts

| Option                            | Accounts | Verdict  | Why                                                                                    |
| --------------------------------- | -------- | -------- | -------------------------------------------------------------------------------------- |
| Default service accounts          | 0        | Rejected | The Compute Engine default account often holds `Editor` on the whole project, or no roles at all if an organisation policy blocks that. Either way the setup depends on something Part B does not control. |
| One account for everything        | 1        | Rejected | Simple, but one account then holds every permission. The demo shows nothing about least privilege. |
| **One account per job**           | 3        | Chosen   | Each account's roles match one sentence: builds the code, delivers the event, does the work. |

- **Names:** `content-<job>` (S2.2). Service account IDs need 6 to 30 characters.
- **The deploying user:** creating a function that runs as these accounts requires permission to "act as" them (`iam.serviceAccounts.actAs`). The maintainer holds `Owner` (S4.5), which includes it.

### S4.2: smallest scope

**Rule:** grant on the resource the job touches. Fall back to the project only when the role cannot be granted on anything smaller.

| Scope     | Terraform resource                          | Used for                                 |
| --------- | ------------------------------------------- | ---------------------------------------- |
| Bucket    | `google_storage_bucket_iam_member`          | Data bucket, source bucket               |
| Dataset   | `google_bigquery_dataset_iam_member`        | `content_staging`                        |
| Service   | `google_cloud_run_v2_service_iam_member`    | The function's Cloud Run service         |
| Project   | `google_project_iam_member`                 | `jobUser`, `eventReceiver`, `artifactregistry.writer`, `logWriter` |

**Why the invoker grant targets Cloud Run:** a Cloud Run function runs as a Cloud Run service with the same name. Calling it is a Cloud Run permission, so `roles/run.invoker` goes on that service.

### S4.3: additive IAM only

**Terraform background:** every GCP resource has one **IAM policy**: the full list of who has which role. Terraform offers three resource flavours to manage it:

| Flavour         | Example                         | What it controls                                    | Effect on grants made elsewhere           |
| --------------- | ------------------------------- | --------------------------------------------------- | ----------------------------------------- |
| `*_iam_policy`  | `google_project_iam_policy`     | The whole policy                                    | Deletes every grant not in the file       |
| `*_iam_binding` | `google_project_iam_binding`    | All members of one role                             | Deletes other members of that role        |
| `*_iam_member`  | `google_project_iam_member`     | One member in one role                              | None; the grant is added alongside others |

- **Why it matters here:** the project belongs to Part A and is shared with other features (S1.1). A project-level `_policy` or `_binding` in Part B would silently remove the grants of Part A, including the maintainer's `Owner`.
- **On `destroy`:** an `_iam_member` removes only its own grant.
- **Part A is not changed:** adding a member grant leaves every existing grant untouched, so the story "added without changing Part A" holds.

### S4.4: no keys

- **How services authenticate:** a function or build runs **as** its attached service account. Google issues short-lived tokens automatically.
- **Why keys are excluded:** a key file is a long-lived password. If it leaks (for example in a public repository), anyone can act as the account until the key is deleted.
- **Many organisations block key creation** with an organisation policy.

### S4.5: the maintainer

- **Part A:** human access is a platform concern. The maintainer's `Owner` role comes with the project from the bootstrap.
- **How Terraform logs in:** with the maintainer's own credentials, via `gcloud auth application-default login`. No service account is involved.
- **CI:** TFLint needs no GCP access. Whether CI gets a GCP identity is decided in step 7.

### S4.6: no prefix conditions

- **Possible:** an **IAM condition** can limit a bucket grant to object names starting with `raw/` or `processed/`.
- **Why not:** two conditional grants with expression syntax, for a bucket that holds only pipeline data. The data bucket is the scope boundary.
- **When it would matter:** a bucket shared by several jobs or teams.

<br>

## Hand-off to step 5

- **IAM propagation:** a new grant can take up to a few minutes to take effect. If Terraform creates the trigger before `content-trigger` can receive events, or the function before `content-build` can build, the first `apply` may fail. Step 5 decides how the module handles this.

<br>

## Open questions

- **Exact role list for the build account:** confirmed against the current Cloud Run functions documentation at implementation time.
