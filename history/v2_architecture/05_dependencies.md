# Step 5: dependencies

> **Reviewed:** [step-5_dependencies.md](../v3_review_plan/step-5_dependencies.md) corrects this step. Where they differ, the review wins.

Decision log for step 5 of [00_roadmap.md](00_roadmap.md). Builds on [03_resource-inventory.md](03_resource-inventory.md) and [04_iam.md](04_iam.md).

**Status values:** as in [00_roadmap.md](00_roadmap.md).

<br>

## Summary

Terraform works out the creation order from references between resources. The module adds an explicit `depends_on` in two situations only, both where a dependency is real but invisible to Terraform: the function waits for the IAM grants it needs, and each view block waits for the tables or views its SQL reads.

IAM grants can take a few minutes to take effect. The module does not add an artificial wait. If the first `apply` fails on a permission error, a second `apply` finishes the job; a wait is added only if this happens repeatedly.

Data is uploaded only after `apply` has finished, so the trigger always exists before the first file lands.

`terraform destroy` removes all 33 resources in reverse order, with their contents. A few Google-managed leftovers remain (container images, logs, build history) and are accepted.

<br>

## Decisions

| ID   | Decision                                                                                                         | Status    |
| ---- | ---------------------------------------------------------------------------------------------------------------- | --------- |
| S5.1 | Creation order comes from references. `depends_on` only where a reference cannot express the need: the function (S5.2) and the view blocks (S3.7). | Delegated |
| S5.2 | The function has `depends_on` on the build account's 3 grants and the trigger account's `eventReceiver` grant.  | Delegated |
| S5.3 | No `time_sleep` at first. A permission error on the first `apply` is fixed by running `apply` again.            | Delegated |
| S5.4 | Data is uploaded only after `apply` has finished (runbook).                                                      | Delegated |
| S5.5 | The `run.invoker` grant is created after the function; the short gap is accepted.                                | Delegated |
| S5.6 | `destroy` removes everything in one command; Google-managed leftovers are accepted.                              | Delegated |

<br>

## Terraform background: the dependency graph

- **Graph:** before `apply`, Terraform builds a graph of all resources. An arrow from A to B means "A must exist before B".
- **Implicit dependency:** when B's arguments reference A (for example `bucket = google_storage_bucket.data.name`), Terraform draws the arrow itself.
- **Explicit dependency:** `depends_on = [A]` draws the arrow by hand. It is needed only when B relies on A without referencing any of A's values.
- **Parallelism:** resources without an arrow between them are created at the same time (up to 10 by default).
- **Destroy:** the same graph, followed in reverse. A resource is deleted only after everything that depends on it.
- **Seeing the graph:** `terraform graph` prints it; useful for explaining the module.

<br>

## Creation order

Resources in the same wave have no arrows between them and are created in parallel.

| Wave | Resources                                                                                     | Waits for (reference)                         |
| ---- | --------------------------------------------------------------------------------------------- | --------------------------------------------- |
| 1    | 3 service accounts, 2 buckets, 3 datasets; the zip is built (`archive_file`)                  | Inputs only                                   |
| 2    | 7 IAM grants (all except `run.invoker`), zip upload, 6 staging tables                          | Service accounts, buckets, datasets           |
| 3    | Cleaned views `blog_posts`, `articles`                                                         | Staging tables (`depends_on`, S3.7)           |
| 4    | View `content`; function (with its trigger)                                                    | Cleaned views (`depends_on`, S3.7); zip upload, accounts, grants (`depends_on`, S5.2) |
| 5    | 6 analysis views; `run.invoker` grant                                                          | `content` and cleaned views (`depends_on`, S3.7); function |

The waves are a simplification. Terraform starts each resource as soon as its own arrows are satisfied, not wave by wave.

<br>

## Reasoning

### S5.1 and S5.2: where `depends_on` is used

| Where                | Hidden dependency                                        | Decided in |
| -------------------- | -------------------------------------------------------- | ---------- |
| The function         | IAM grants on the accounts it references                 | S5.2       |
| Each view block      | Tables and views named inside the SQL text               | S3.7       |

The function references the service accounts (their emails), not the grants. By references alone, Terraform could create the function in wave 2, at the same time as the grants.

| Grant the function needs at creation          | Why                                                         | What fails without it |
| --------------------------------------------- | ----------------------------------------------------------- | --------------------- |
| `content-build`: `storage.objectViewer`        | Cloud Build reads the zip                                   | The build             |
| `content-build`: `artifactregistry.writer`     | Cloud Build pushes the image                                | The build             |
| `content-build`: `logging.logWriter`           | Cloud Build writes its log                                  | The build             |
| `content-trigger`: `eventarc.eventReceiver`    | The trigger account must be allowed to receive events       | Event delivery; possibly trigger creation (verify at implementation) |

- **Not in `depends_on`:** the runtime account's grants. They are used only when the function runs, which happens after the upload (S5.4).
- **Why `depends_on` stays rare:** it hides intent. A reference shows why one resource needs another; `depends_on` only shows that it does. Each use gets a comment in the code that names the hidden dependency.

### S5.3: IAM propagation

- **The issue:** a new grant usually takes effect within a minute or two, but can take several minutes. `depends_on` guarantees order, not that the grant is already active.
- **Possible symptom:** the first `apply` fails on the function with a permission error from Cloud Build or Eventarc.

| Option                                   | How it works                                                     | Verdict                     |
| ---------------------------------------- | ---------------------------------------------------------------- | --------------------------- |
| **Run `apply` again**                    | Terraform creates whatever is still missing                      | Chosen as the first step    |
| `time_sleep` (`hashicorp/time` provider) | A resource that waits a fixed time (for example 60 seconds) after the grants | Fallback if the error recurs |
| Grants in Part A                         | Grants exist long before the module runs                         | Rejected: breaks S1.8       |

- **Why not `time_sleep` from the start:** it adds a third provider and a fixed wait on every first `apply`, for a problem that may not occur. It is added only if the build phase shows the error more than once.
- **Terraform background:** a resource that fails halfway through creation is marked **tainted** in state. The next `apply` deletes and recreates it. This is why a second `apply` is safe.
- **Already avoided:** APIs and the storage service agent grant are prepared in Part A long before (S1.4, S1.5), so their propagation never affects Part B.

### S5.4: upload after `apply`

- **Why:** the trigger must exist before a file lands, or no event fires (D1.2).
- **How:** the runbook runs `terraform apply`, then the upload script. Two separate commands, one after the other.

### S5.5: the `run.invoker` gap

- **Why the gap exists:** the grant targets the function's Cloud Run service, which exists only after the function is created.
- **Why it is harmless:** no event arrives before the upload (S5.4). If one did, Eventarc would retry until the grant is active (D1.10).

<br>

## Update behaviour

What a later `apply` does after a change. `plan` shows "update in-place" or "must be replaced".

| Change                                      | Effect                                                      |
| ------------------------------------------- | ----------------------------------------------------------- |
| Function code                               | New zip name (hash), function rebuilt and updated in place  |
| View SQL                                    | View updated in place                                       |
| Staging schema: new column                  | Table updated in place                                      |
| Staging schema: column removed or retyped   | Table replaced; rows lost, reload by upload (D1.3)          |
| Labels                                      | Updated in place                                            |
| Region                                      | Buckets, datasets and function replaced (D3.2)              |

<br>

## Destroy behaviour (S5.6)

### Order

The reverse of creation: analysis views and the `run.invoker` grant first, then the function and the `content` view, and so on down to the service accounts, buckets and datasets.

### What makes it work

| Setting                              | Resource         | Effect                                                          | From |
| ------------------------------------ | ---------------- | --------------------------------------------------------------- | ---- |
| `force_destroy = true`               | Buckets          | Deletes uploaded and processed files with the bucket            | D1.4 |
| `delete_contents_on_destroy = true`  | Datasets         | Deletes tables the module did not create                        | D1.4 |
| `deletion_protection = false`        | Tables and views | Allows Terraform to delete them                                 | D1.4 |

- **Terraform background:** `deletion_protection` is read from **state**. A table created with `true` must first be changed to `false` with an `apply` before `destroy` works. Setting `false` from the start avoids this.
- **Trigger plumbing:** Eventarc creates a Pub/Sub topic and a bucket notification for the trigger. They are deleted with the function, before the bucket.

### What remains after `destroy`

| Leftover                                    | Owner           | Why accepted                                      |
| ------------------------------------------- | --------------- | ------------------------------------------------- |
| Part A items (project, state bucket, APIs)  | Part A          | By design (S1.1)                                  |
| Container images in `gcf-artifacts`         | Google / Part A | A few MB; a cleanup policy is a Part A concern (S1.6) |
| Function and build logs                     | Cloud Logging   | Expire on their own (30 days by default)          |
| Cloud Build history                         | Cloud Build     | Metadata only; no cost                            |

### Rebuild after `destroy`

- **Bucket names:** become available again after deletion; an immediate rebuild normally works.
- **Datasets and service accounts:** names can be reused at once. Re-created accounts get new internal IDs; the grants are re-created with them.

<br>

## Open questions

- **`time_sleep`:** added only if the build phase shows propagation errors more than once (S5.3).
