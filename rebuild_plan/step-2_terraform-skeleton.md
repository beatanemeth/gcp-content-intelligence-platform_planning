# Step 2: Terraform skeleton and the 2 buckets

**Branch:** `feature/terraform-skeleton`

<br>

## In plain words

The first Terraform code: the module's frame, the root module that calls it, and the 2 buckets.

- **The module** (`terraform/modules/content_pipeline/`) is the recipe. It says what Part B creates, with 2 inputs: the project ID and the region.
- **The root module** (`terraform/environments/dev/`) is the cook. It sets the provider, the state location and the values, then calls the module. Every `terraform` command runs here.
- **The 2 buckets:**
  - **Data bucket:** uploads (`raw/`) and the function's output (`processed/`).
  - **Source bucket:** the function's code zip (step 5b).
  - **Why 2:** the function's runtime account can write data. Kept apart, it can never touch the code.
- **The simulated check:** `terraform test` runs the module against a **mocked Google provider**. It "creates" the buckets on the laptop and checks their names and settings. Nothing reaches GCP.
- **CI:** Terraform checks join pre-commit and GitHub Actions.

<br>

## Technical details

### Module: `terraform/modules/content_pipeline/`

| File           | Holds                                                                     |
| -------------- | ------------------------------------------------------------------------- |
| `versions.tf`  | `required_version = ">= 1.7"`; `google` provider `>= 8.0`                 |
| `variables.tf` | `project_id` and `region`: type `string`, a description, **no default**   |
| `main.tf`      | `locals`: the name prefix `feature = "content"` and the 2 labels          |
| `storage.tf`   | `google_storage_bucket.data` and `google_storage_bucket.source`           |
| `outputs.tf`   | `data_bucket_name`                                                        |

- **Minimum versions only (`>=`):** a module should work with any newer version the caller chooses. The root module sets the upper limit.
- **Terraform 1.7 or newer:** mock providers, used by the tests, arrived in 1.7.
- **No provider block in the module:** the caller configures the provider. A module with its own provider block cannot be removed cleanly.
- **No defaults on the inputs:** a missing value stops `plan` at once, instead of building in the wrong project.
- **Each output arrives with its resource:** `dataset_ids` comes in step 4, `function_name` in step 5b.

**Labels** (`local.labels`), on every resource that supports them:

| Label        | Value              | Why                                              |
| ------------ | ------------------ | ------------------------------------------------ |
| `feature`    | `content-pipeline` | Filter the feature's resources and costs in a shared project |
| `managed_by` | `terraform`        | Tells others not to change it by hand            |

### The 2 buckets (`storage.tf`)

Two separate `resource` blocks, not one `for_each`: the buckets have different purposes and will get different permissions.

| Argument                           | Value                                    | Why                                                           |
| ---------------------------------- | ---------------------------------------- | ------------------------------------------------------------- |
| `name`                             | `"${var.project_id}-${local.feature}-data"` (or `-source`) | Bucket names are global; the project ID makes them unique |
| `project`, `location`              | `var.project_id`, `var.region`           | Everything in one project and one region                      |
| `force_destroy`                    | `true`                                   | `destroy` removes the bucket even with files in it; the data lives on the laptop |
| `uniform_bucket_level_access`      | `true`                                   | Access by IAM only, no per-file permissions                   |
| `public_access_prevention`         | `"enforced"`                             | No file can ever be made public                               |
| `soft_delete_policy { retention_duration_seconds = 0 }` | Off | Replaced files are not kept (and billed) for 7 days; everything can be uploaded again |
| `labels`                           | `local.labels`                           | See above                                                     |

**Simulated names:** `content-demo-sim-content-data`, `content-demo-sim-content-source`.

### Root module: `terraform/environments/dev/`

| File                       | Holds                                                                         |
| -------------------------- | ----------------------------------------------------------------------------- |
| `versions.tf`              | `required_version = "~> 1.16"`; `google` provider `~> 8.6`                    |
| `backend.tf`               | `backend "gcs"` with `prefix = "content-pipeline"`; no bucket                 |
| `main.tf`                  | `provider "google"` (project, region) and the `module "content_pipeline"` call |
| `variables.tf`             | `project_id` (no default), `region` (default `europe-west1`)                  |
| `outputs.tf`               | `data_bucket_name`, passed on from the module                                 |
| `terraform.tfvars.example` | `project_id = "your-project-id"`                                              |
| `backend.hcl.example`      | `bucket = "your-project-id-tfstate"`                                          |
| `.terraform.lock.hcl`      | Created by `terraform init`; committed                                        |

- **`~>` (pessimistic constraint):** `~> 8.6` allows 8.6, 8.7 and later 8.x, never 9.0. New features and fixes arrive; breaking changes do not.
- **The lock file** records the exact provider version and its checksums. Committed, so the laptop and CI use the same build.
- **Partial backend:** a `backend` block cannot read variables. The bucket name comes from `backend.hcl` at `terraform init -backend-config=backend.hcl`, so the real project ID stays out of git.
- **Simulation:** `terraform init -backend=false` skips the backend; no state bucket and no login are needed.
- **The region default lives here, not in the module:** the module stays neutral; this environment chooses.

### Simulated test: `tests/storage.tftest.hcl`

- **`mock_provider "google"`:** replaces the real provider. Every resource "succeeds" locally; computed values (such as IDs) get random strings.
- **`variables`:** the simulated values (`content-demo-sim`, `europe-west1`).
- **`run "buckets"` with `command = apply`:** creates the mocked resources, checks them, and destroys them again at the end.
- **Assertions:** both names, project and region, `force_destroy`, uniform access and public access prevention, soft delete off, the labels, and the `data_bucket_name` output.

```bash
cd terraform/modules/content_pipeline
terraform init -backend=false
terraform test
```

### TFLint: `.tflint.hcl` (repository root)

- **Plugins, pinned:** `terraform` (preset `recommended`) and `google`. `tflint --init` installs them.
- **Extra rules:** `terraform_naming_convention` (snake_case names), `terraform_documented_variables`, `terraform_documented_outputs`.
- **No deep checking** in the `google` plugin: it would call the GCP APIs.
- **One file for both folders:** run with `--recursive --config "$(pwd)/.tflint.hcl"`. A relative path would be looked up in each folder.

### pre-commit

Add `antonbabenko/pre-commit-terraform` with 3 hooks:

| Hook                 | Note                                                                    |
| -------------------- | ----------------------------------------------------------------------- |
| `terraform_fmt`      | Formats every `.tf` file                                                |
| `terraform_validate` | Runs `terraform init -backend=false` by itself where needed             |
| `terraform_tflint`   | `--args=--config=__GIT_WORKING_DIR__/.tflint.hcl`, the root file by full path |

### CI: `.github/workflows/terraform-checks.yml`

- **Runs on:** pull requests and pushes to `main`, only when `terraform/**`, `.tflint.hcl` or the workflow itself changes.
- **`permissions: contents: read`:** the token can only read the repository.
- **Steps:**
  1. `terraform fmt -check -recursive`
  2. `terraform init -backend=false` and `terraform validate` in `environments/dev/` (this also checks the module)
  3. `terraform init -backend=false` and `terraform test` in the module
  4. `tflint --init`, then `tflint --recursive`, both with the root config by full path
- **Same versions as the laptop:** Terraform `1.16.x` and TFLint `0.64.0`, set in the setup actions.

### Notes in the Terraform files

| File                 | Note                                                                       |
| -------------------- | -------------------------------------------------------------------------- |
| Module `versions.tf` | Minimum versions only; the root module pins the range. 1.7 adds mock providers |
| `variables.tf`       | Values come from Team A's handover; no defaults, so a missing value stops `plan` |
| `main.tf`            | Every name starts with the prefix; the labels go on every resource         |
| `storage.tf`         | Why 2 buckets; one note each on `force_destroy`, public access and soft delete |
| Root `versions.tf`   | The root module sets the upper limit; the lock file records exact versions |
| `backend.tf`         | A backend cannot read variables; the bucket comes from `backend.hcl`       |
| `*.example`          | Copy to the git-ignored file and fill in the real value                    |
| `storage.tftest.hcl` | The mocked provider replaces `apply`, `output` and `destroy` in real GCP   |

### `simulation/README.md`

Add a row for step 2: the real check would be `apply`, `output`, `destroy`; the simulated check is `fmt`, `validate`, TFLint and `terraform test` with the mocked provider.

### Docs

Each document grows with the step.

| Document            | Adds                                                                                  |
| ------------------- | ------------------------------------------------------------------------------------- |
| `README.md`         | **Run it yourself:** prerequisites (Terraform, TFLint), set-up, `terraform test`      |
| Module `README.md`  | **Created:** what the module is (fixed names, one copy per project); what it needs from Part A (project with billing, the 10 APIs, the service agent's publish permission, `Owner`); inputs; the `data_bucket_name` output; the 2 buckets; the `module` block |
| `docs/decisions.md` | 2 inputs only; the `content` prefix and the labels; 2 buckets; `destroy` removes everything; CI runs offline checks only |

### Commits

1. `feat(terraform): add module versions, variables and locals`
2. `feat(terraform): add data and source buckets`
3. `feat(terraform): add dev environment with module call and backend`
4. `test(terraform): add simulated bucket test with mocked provider`
5. `ops(ci): add Terraform pre-commit hooks and workflow`
6. `docs: start the module README and add the Terraform decisions`

<br>

## Done when

- `terraform fmt -check -recursive` and `terraform validate` (in `environments/dev/`) pass.
- `terraform test` in the module: 1 run passes.
- `tflint --recursive` reports nothing.
- `pre-commit run --all-files` passes.
- The `Terraform checks` workflow is green on the pull request.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
