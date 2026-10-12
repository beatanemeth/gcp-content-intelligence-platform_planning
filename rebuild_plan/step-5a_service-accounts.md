# Step 5a: service accounts and grants

**Branch:** `feature/service-accounts`

<br>

## In plain words

The pipeline's own identities and their permissions, before the function exists.

- **3 service accounts, one per job:**
  - **Build:** turns the function's source code into a running service.
  - **Trigger:** delivers the "file uploaded" event to the function.
  - **Runtime:** the identity the function runs as; reads uploads, writes NDJSON, loads BigQuery.
- **Why 3, not 1:** a bug or a leak in one job opens only that job's doors. The runtime account, which handles data, cannot touch the code; the build account cannot read the data.
- **Least privilege:** each role on the smallest scope that works: one bucket, one dataset. The project only when nothing smaller exists.
- **No keys:** Google issues a short-lived token on each run. No secret file exists to leak.
- **Additive only:** every grant adds one member to one role. Nothing removes access that Team A or another team has given.
- **Order:** 7 of the 8 grants come here. The 8th, `run.invoker`, targets the function's service, so it comes in step 5b with the function.

<br>

## Technical details

### Locals (module `main.tf`)

| Local              | Value                                                                       |
| ------------------ | --------------------------------------------------------------------------- |
| `service_accounts` | Map: `build`, `trigger`, `runtime`, each with its display name              |
| `project_roles`    | Map of the 4 project-level grants: each has an `account` and a `role`       |

### How the roles were chosen

1. **List what each job does,** as a verb on a resource: "read a file in `raw/`".
2. **Find the permission for each action:** every API call needs one, such as `storage.objects.get`. Each API's reference names it.
3. **Pick the smallest predefined role that holds those permissions:** the [IAM roles reference](https://cloud.google.com/iam/docs/roles-permissions) lists each role's permissions.
4. **Grant it on the smallest scope:** bucket or dataset first; the project only when the permission does not exist lower down.

| Account | Action                                     | Permission                                   | Role                      | Scope                    |
| ------- | ------------------------------------------ | -------------------------------------------- | ------------------------- | ------------------------ |
| runtime | Read `raw/` files                          | `storage.objects.get`                        | `storage.objectUser`      | Data bucket              |
| runtime | Overwrite `processed/` files               | `storage.objects.create`, `storage.objects.delete` | (same)              | (same)                   |
| runtime | Let the load job read `processed/`         | `storage.objects.get`: a load from a URI reads with the caller's identity | (same) | (same)        |
| runtime | Start a load job                           | `bigquery.jobs.create`                       | `bigquery.jobUser`        | Project: a job belongs to a project, not to a dataset |
| runtime | Replace a staging table's rows             | `bigquery.tables.updateData`                 | `bigquery.dataEditor`     | `content_staging`        |
| trigger | Receive an event from Eventarc             | `eventarc.events.receiveEvent`               | `eventarc.eventReceiver`  | Project: no smaller scope exists |
| trigger | Call the function (step 5b)                | `run.routes.invoke`                          | `run.invoker`             | The function's service   |
| build   | Read the source zip                        | `storage.objects.get`                        | `storage.objectViewer`    | Project, with a condition (see below) |
| build   | Push the built image                       | `artifactregistry.repositories.uploadArtifacts` | `artifactregistry.writer` | Project: Google creates the repository at the first deploy |
| build   | Write the build logs                       | `logging.logEntries.create`                  | `logging.logWriter`       | Project: logs are written per project |

- **Why `objectUser`, not `objectCreator`:** Cloud Storage requires `storage.objects.delete` to overwrite an existing file; `objectCreator` does not have it.
- **Why predefined roles, not custom ones:** a custom role could be smaller, but Google does not update it when an API changes. Predefined roles are the usual choice at this size.
- **Where the answers are confirmed:**
  - **Google's guides:** the Cloud Run functions docs list the build account's roles; the Eventarc docs list the trigger account's.
  - **Error messages in real GCP:** a missing permission is named, such as `Permission 'bigquery.jobs.create' denied`.
  - **IAM Recommender, after real runs:** shows the permissions an account never used, so a role can shrink. Not available in the simulation.

### `iam.tf`: the accounts

`google_service_account.pipeline`, with `for_each = local.service_accounts`.

| Argument       | Value                             |
| -------------- | --------------------------------- |
| `account_id`   | `"${local.feature}-${each.key}"`: `content-build`, `content-trigger`, `content-runtime` |
| `display_name` | `each.value`, such as `Content pipeline: function build` |
| `project`      | `var.project_id`                  |

- **`member` attribute:** each account exports `serviceAccount:<email>`, ready for the grants. No string building is needed.
- **No `google_service_account_key`:** never created.

### `iam.tf`: the 7 grants

| Account | Role                            | Scope                       | Resource type                        | Why                                        |
| ------- | ------------------------------- | --------------------------- | ------------------------------------ | ------------------------------------------ |
| runtime | `roles/storage.objectUser`      | Data bucket                 | `google_storage_bucket_iam_member`   | Read `raw/`, overwrite `processed/`        |
| runtime | `roles/bigquery.dataEditor`     | `content_staging` only      | `google_bigquery_dataset_iam_member` | Replace staging rows; cleaned and analysis stay out of reach |
| runtime | `roles/bigquery.jobUser`        | Project                     | `google_project_iam_member`          | Run load jobs; jobs exist only at project level |
| trigger | `roles/eventarc.eventReceiver`  | Project                     | `google_project_iam_member`          | Receive events; project level only         |
| build   | `roles/artifactregistry.writer` | Project                     | `google_project_iam_member`          | Push the built image                       |
| build   | `roles/logging.logWriter`       | Project                     | `google_project_iam_member`          | Write the build logs                       |
| build   | `roles/storage.objectViewer`    | Project, **with a condition** | `google_project_iam_member`        | Read the source code (see below)           |

- **The 4 plain project grants share one block:** `google_project_iam_member.pipeline`, with `for_each = local.project_roles`.
- **Why no folder-level condition for the runtime account** (`raw/` read, `processed/` write): it would add a condition for little gain; the bucket holds only this pipeline's data.

### The build account's condition

**The problem:** during the build, Google copies the source code into its own buckets (`gcf-v2-sources-...`, `gcf-v2-uploads-...`, `run-sources-...`). They exist only after the first deploy, so a bucket-level grant is impossible. A plain project-level `objectViewer` would also reach the data bucket and Team A's state bucket.

**The solution:** a project-level grant with an **IAM condition**: a CEL expression that IAM checks on every request.

- **Allowed:** objects whose name starts with Google's 3 code bucket prefixes, or with `projects/_/buckets/<source bucket>/`.
- **Everything else:** denied, including the data bucket and the state bucket.
- **The trailing `/`** after the source bucket's name: matches that bucket only, not a bucket whose name merely starts the same way.
- **`title` and `description`** are required for a condition; the title is `content-build-code-buckets-only`.

### Resource types for grants: `*_iam_member` only

| Type             | Effect                                                  | Used |
| ---------------- | ------------------------------------------------------- | ---- |
| `*_iam_member`   | Adds one member to one role; leaves everyone else       | Yes  |
| `*_iam_binding`  | Sets **all** members of one role; removes the others    | No   |
| `*_iam_policy`   | Sets the **whole** policy; removes everything else      | No   |

A `_binding` or `_policy` in a shared project would remove Team A's access, which breaks the rule that Part B changes nothing in Part A.

### Simulated test: `tests/iam.tftest.hcl`

**The mocked `member` problem:** the mocked provider fills every computed value with a random string. The provider checks the format of an IAM member (`serviceAccount:...`) before any call, so the grants fail.

**The fix,** in every test file: each one applies the whole module, so each one now creates the accounts and grants.

| Test file                                  | Fix                                                                         |
| ------------------------------------------ | --------------------------------------------------------------------------- |
| `storage.tftest.hcl`, `bigquery.tftest.hcl` | Inside `mock_provider "google"`: `mock_resource "google_service_account"` with `defaults = { member = "serviceAccount:mock@content-demo-sim.iam.gserviceaccount.com" }`, one value for all 3 accounts |
| `iam.tftest.hcl`                           | One `override_resource` per account (`google_service_account.pipeline["build"]` and so on), each with its own `member`, so the test can tell which account got which role |

`run "service_accounts_and_grants"`, `command = apply`. Assertions:

- The 3 account IDs, all in the project input.
- The runtime account: `objectUser` on the data bucket only; `dataEditor` on `content_staging` only.
- Exactly the 4 project-level roles, each to its own account.
- The build account: `objectViewer` on the project; the condition allows the 3 Google prefixes and the source bucket with its trailing `/`; it does not contain the data bucket.

### Notes in the files

| File                | Note                                                                       |
| ------------------- | -------------------------------------------------------------------------- |
| `main.tf`           | `service_accounts`: one per job. `project_roles`: roles that exist only at project level |
| `iam.tf`            | Why one account per job; no keys; every grant is `*_iam_member`; `run.invoker` lives in `function.tf`; one line per grant saying what it allows; the condition's reason and the trailing `/` |
| `*.tftest.hcl`      | Why the mocked service account needs a valid `member`                      |

### `simulation/README.md`

Add the step 5a row and the mocked `member` note. What the simulation cannot prove: IAM propagation delays, and that the condition really lets the build read its code.

### Docs

Each document grows with the step.

| Document            | Adds                                                                      |
| ------------------- | ------------------------------------------------------------------------- |
| Module `README.md`  | The 3 service accounts and their grants                                   |
| `docs/decisions.md` | 3 accounts, one per job; each role at the smallest scope; the build account's condition; `*_iam_member` only; no keys |

### Commits

1. `feat(terraform): add the 3 service accounts`
2. `feat(terraform): add the runtime and trigger grants`
3. `feat(terraform): add the build grants with the code bucket condition`
4. `test(terraform): give mocked service accounts a valid member`
5. `test(terraform): add simulated IAM test with mocked provider`
6. `docs: add the service accounts and IAM decisions`

<br>

## Done when

- `terraform test`: 3 runs pass (buckets; datasets and tables; accounts and grants).
- `tflint --recursive` reports nothing.
- **Proof that the test can fail:** change the runtime account's `dataEditor` grant to `content_cleaned`; the IAM test fails. Undo the change.
- `pre-commit run --all-files` and the Terraform workflow are green.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
