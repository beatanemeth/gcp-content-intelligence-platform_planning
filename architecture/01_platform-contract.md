# Step 1: platform contract

> **Reviewed:** [step-1_platform-contract.md](../review_plan/step-1_platform-contract.md) corrects this step. Where they differ, the review wins.

Decision log for step 1 of [00_roadmap.md](00_roadmap.md).

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

Part A (the platform) provides five things. Part B (the pipeline) uses them, but never creates or changes them:

1. The GCP project, linked to billing, with its region policy (`europe-west1`)
2. The Terraform state bucket
3. The enabled APIs
4. The Cloud Storage service agent's permission to publish events
5. The billing budget and its alerts

Part B creates everything that only the pipeline uses: buckets, the function and its trigger, BigQuery datasets, tables and views, and its own service accounts with their roles.

<br>

## Decisions

| ID    | Decision                                                                                                           | Status    |
| ----- | ------------------------------------------------------------------------------------------------------------------ | --------- |
| S1.1  | **Ownership rule:** a shared resource belongs to Part A; a pipeline-specific resource belongs to Part B.            | Agreed    |
| S1.2  | Part A provides the project with a billing link.                                                                   | Delegated |
| S1.3  | Part A provides the Terraform state bucket. Part B stores its state under its own prefix.                          | Delegated |
| S1.4  | Part A enables every API that Part B needs. Part B contains no `google_project_service` resource.                  | Delegated |
| S1.5  | Part A grants the Cloud Storage service agent `roles/pubsub.publisher`.                                            | Delegated |
| S1.6  | No Artifact Registry repository in either part. The function uses the automatically created `gcf-artifacts` repository. | Delegated |
| S1.7  | Part A owns the billing budget and its alerts.                                                                     | Delegated |
| S1.8  | Part B creates its own service accounts and grants their roles (details in step 4).                                | Delegated |
| S1.9  | A `gcloud` bootstrap script stands in for Part A.                                                                  | Delegated |
| S1.10 | The module is deployed once. Names that must be globally unique contain the project ID.                            | Delegated |

<br>

## What Part A provides

| Item                            | How Part B receives it                                               |
| ------------------------------- | -------------------------------------------------------------------- |
| Project with billing            | `project_id` variable, passed from the root configuration to the module |
| Region policy                   | `region` variable, set to `europe-west1` in the root configuration (D3.12) |
| State bucket                    | Backend setting at `terraform init`; the module never sees it        |
| Enabled APIs                    | Assumed; not referenced in code                                      |
| Storage service agent grant     | Assumed; not referenced in code                                      |
| Budget and alerts               | Assumed; not referenced in code                                      |

### APIs Part B expects

Confirmed against the resource inventory in step 3 (S3.8).

| API                                | Needed for                                              |
| ---------------------------------- | ------------------------------------------------------- |
| `storage.googleapis.com`           | Data and source buckets                                 |
| `cloudfunctions.googleapis.com`    | The function                                            |
| `run.googleapis.com`               | The service that runs the function                      |
| `cloudbuild.googleapis.com`        | Building the function's container from source           |
| `artifactregistry.googleapis.com`  | Storing the built container image                       |
| `eventarc.googleapis.com`          | The Cloud Storage trigger                               |
| `pubsub.googleapis.com`            | Event delivery behind the trigger                       |
| `bigquery.googleapis.com`          | Datasets, tables, views, load jobs                      |
| `iam.googleapis.com`               | Creating Part B's service accounts                      |
| `cloudresourcemanager.googleapis.com` | Part B's project-level IAM grants (`google_project_iam_member` reads and updates the project's IAM policy through this API) |

<br>

## Reasoning

### S1.1: ownership rule

- **Story constraint:** Part B is added to Part A without changing it. A pipeline-specific resource in Part A would force a Part A change for every new feature.
- **Test:** "Would a second feature need the same thing?" Yes: Part A. No: Part B.

### S1.4: APIs in Part A

- **Project-wide switch:** an enabled API serves every feature in the project.
- **Terraform risk:** `google_project_service` disables the API on `destroy` by default (`disable_on_destroy = true`). A Part B `destroy` could then break other features. Setting it to `false` avoids that, but the resource would still not belong to Part B.
- **Simpler Part B:** no waiting for newly enabled APIs to become active during the first `apply`.

### S1.5: storage service agent grant in Part A

- **What it is:** a Google-managed identity, one per project, that acts for Cloud Storage. It publishes bucket events to Pub/Sub, and Eventarc delivers them to the function.
- **Why Part A:** every feature with a Cloud Storage trigger needs the same grant. It is shared.
- **Side benefit:** the most common first-apply failure for storage triggers disappears from Part B (the trigger created before the permission takes effect).

### S1.6: no Artifact Registry resource

- **Mechanism:** Cloud Build builds the function's container and stores it in a repository called `gcf-artifacts`, which Google creates on the first deploy.
- **Effect:** neither part manages it. A cleanup policy for old images would be a Part A concern; at this data size it is optional.

### S1.8: service accounts in Part B

- **Ownership rule:** the accounts serve only the pipeline.
- **Least privilege:** roles are granted on Part B's own resources (the data bucket, the staging dataset). Those resources do not exist when Part A is applied.

### S1.9: `gcloud` script as the bootstrap stand-in

- **Part A is out of scope:** in the story, Part A belongs to an imagined platform team. Its only job here is to state what Part B may assume. Writing it as Terraform would turn the imagined platform into real code to maintain, explain and defend, and blur the boundary the demo is about.
- **Chicken-and-egg problem:** Terraform must know where to store its state before it creates anything. Part A creates the state bucket, so Part A's own Terraform would have nowhere to store state on its first run. The usual fix (local state first, then `terraform init -migrate-state` into the new bucket) is an extra concept unrelated to the pipeline. A script keeps no state, so the problem does not arise.
- **Runs once, never changes:** Terraform pays off for infrastructure that evolves (`plan`, drift detection, `destroy`). Part A is created once and kept; a few `gcloud` commands are enough.
- **Reproducible:** anyone can rebuild the demo from the script and the module.
- **Not the same as the data upload:** data stays out of Terraform for a different reason (D1.2): it is not infrastructure.

### S1.10: one instance

- **Simplicity:** no `environment` or prefix input.
- **Global names:** bucket names are unique across all of GCP, so they contain the project ID (for example `<project-id>-content-data`, S2.2).

<br>

## Trade-off accepted

- **Hidden assumptions:** Part B depends on items it does not reference in code (S1.4, S1.5, S1.7). `terraform plan` does not check them; a missing API or grant only fails at `apply`.
- **Mitigation:** this contract and the implementation README list the prerequisites, and the bootstrap script creates them.

<br>

## Open questions

- None. The API list was confirmed in step 3 (S3.8).
