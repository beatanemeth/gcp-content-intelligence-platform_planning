# Cross-cutting topics

Decision log for the "Requirements and open questions" section of [03_bea_answers_and_thoughts.md](03_bea_answers_and_thoughts.md). Earlier steps: [04_claude_step1_raw_data_upload.md](04_claude_step1_raw_data_upload.md), [05_claude_step2_data_transformation.md](05_claude_step2_data_transformation.md), [06_claude_step3_bigquery.md](06_claude_step3_bigquery.md).

**Status values:** _Agreed_ (confirmed by the maintainer), _Delegated_ (chosen by Claude at the maintainer's request).

**Later changes:** the standalone module plan re-decides part of this file (project setup, APIs, CI) under Part A and Part B; see [08_claude_decision_overview.md](08_claude_decision_overview.md).

<br>

## Maintainer inputs

| Input                                                                  | Affects               |
| ---------------------------------------------------------------------- | --------------------- |
| One person (the maintainer) handles the project; personal Gmail account, no Google Cloud organisation. | Project setup, IAM |
| GitHub Actions are important, not optional.                            | CI scope              |
| Cost and teardown: keep simple; a short-lived project.                 | Topic 3               |
| Public repository: handled by the maintainer; a reminder only.         | Topic 4               |
| Local testing: only where simple.                                      | Topic 5               |
| Step by step: propose a build order.                                   | The module plan       |

Topics appear in discussion order (1, 2, 6, 5, 3, 4): Terraform foundations first, so decision IDs stay ascending.

**Build order:** planned together with the module, in its build phases, because the order depends on what Part A provides. The principles from file 03 hold: local checks and Python tests first, an early `destroy` and `apply` cycle.

<br>

## Topic 1: project setup and Terraform state

**The bootstrap problem:** Terraform needs a project, an identity and a state bucket before its first `apply`, and it cannot create the bucket that holds its own state.

| ID   | Decision                                                                                                          | Status |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ------ |
| D4.1 | A one-time bootstrap script (`gcloud`) creates the dedicated project, links billing and creates the state bucket. | Agreed |
| D4.2 | Terraform state lives in a Cloud Storage bucket (`gcs` backend) with object versioning on.                        | Agreed |
| D4.3 | Terraform enables the required APIs (`google_project_service`), with `disable_on_destroy = false`.                | Agreed |
| D4.4 | `terraform destroy` removes everything except the bootstrap resources (project, state bucket).                    | Agreed |

- **Script, not Terraform (D4.1):** the bootstrap runs once; a short script avoids a second configuration with its own state problem.
- **Remote state (D4.2):** a GitHub workflow cannot reach a state file on the laptop. Versioning lets a broken state be restored; the `gcs` backend also locks the state during a run.
- **APIs stay on (D4.3):** switching an API off during `destroy` can break resources that are still being deleted. Enabled APIs cost nothing.
- **What survives (D4.4):** the project and the state bucket, so the next `apply` can start at once. Deleting the project is the final teardown.

<br>

## Topic 2: IAM

| ID   | Decision                                                                                                      | Status |
| ---- | ------------------------------------------------------------------------------------------------------------- | ------ |
| D4.5 | The maintainer holds `Owner` on the dedicated project and runs Terraform with their own `gcloud` credentials. | Agreed |
| D4.6 | Each workload has its own service account: function runtime, trigger, function build, CI.                    | Agreed |
| D4.7 | Roles are granted at the smallest scope that works.                                                           | Agreed |
| D4.8 | Terraform uses additive IAM resources (`*_iam_member`) only.                                                  | Agreed |
| D4.9 | No service account keys.                                                                                      | Agreed |

**Roles per workload (D4.6, D4.7):**

| Identity                        | Role                      | Scope                                     | Why                                                    |
| ------------------------------- | ------------------------- | ----------------------------------------- | ------------------------------------------------------ |
| **Function runtime**            | Storage Object Admin      | Data bucket                               | Reads `raw/`, writes and overwrites `processed/`       |
|                                 | BigQuery Data Editor      | `staging` dataset                         | Load jobs replace staging table contents               |
|                                 | BigQuery Job User         | Project                                   | Jobs belong to the project                             |
| **Trigger**                     | Cloud Run Invoker         | The function                              | Calling the function                                   |
|                                 | Eventarc Event Receiver   | Project                                   | Receiving events                                       |
| **Function build**              | Cloud Build builder roles | Project, source bucket, image repository  | Building the container (exact roles at implementation) |
| **Cloud Storage service agent** | Pub/Sub Publisher         | Project                                   | Bucket events travel through Pub/Sub                   |

- **Blast radius:** with its own identity, a faulty function can at worst overwrite rebuildable pipeline data. With the maintainer's identity, it could delete the project.
- **Default compute service account:** not used. It has the broad `Editor` role.
- **Maintainer as `Owner` (D4.5):** one person, one personal project. Impersonating a deployment service account is a later improvement.

**Additive IAM resources (D4.8):**

| Style               | Behaviour                                                                          |
| ------------------- | ---------------------------------------------------------------------------------- |
| **`*_iam_policy`**  | Replaces the whole policy; removes every grant not in Terraform, even the maintainer's |
| **`*_iam_binding`** | Owns one role completely; removes any other member holding that role               |
| **`*_iam_member`**  | Adds one member to one role; leaves everything else untouched                      |

**No keys (D4.9):** a key file does not expire and works for whoever holds it. Instead: `gcloud auth application-default login` on the laptop, Workload Identity Federation in GitHub Actions, and the function's own service account in the cloud.

<br>

## Topic 6: GitHub Actions

| Tier | Jobs                                                                          | GCP access           |
| ---- | ----------------------------------------------------------------------------- | -------------------- |
| 1    | `fmt -check`, `validate`, TFLint, a security scanner; Python lint and tests   | None                 |
| 2    | `terraform plan` against the real project                                     | Read                 |
| 3    | `terraform apply` and `terraform destroy` (rejected, D4.10)                   | Write (near `Owner`) |

| ID    | Decision                                                                                                   | Status |
| ----- | ---------------------------------------------------------------------------------------------------------- | ------ |
| D4.10 | CI never changes real infrastructure. `apply` and `destroy` run from the laptop only (no tier 3).          | Agreed |
| D4.11 | Tier 1 (checks) runs on every push to a feature branch, on pull requests to `main` and on merge to `main`. | Agreed |
| D4.12 | Tier 2 (`terraform plan`) runs on pull requests to `main` only.                                            | Agreed |
| D4.13 | The CI service account is read-only; `plan` runs without a state lock.                                     | Agreed |
| D4.14 | The project ID reaches workflows as a GitHub secret, so logs mask it.                                      | Agreed |

- **No apply from CI (D4.10):** applying creates service accounts and IAM grants, so the CI identity would need near-`Owner` rights. With one maintainer, tiers 1 and 2 show the CI skills without that risk.
- **Plan on pull requests (D4.12):** the pull request is where a change is reviewed. Nothing is applied on merge, so a plan there would repeat it.
- **Read-only, no lock (D4.13):** taking the state lock writes to the bucket; `plan -lock=false` only reads.
- **Secret project ID (D4.14):** logs of a public repository are public; GitHub masks secret values.

<br>

## Topic 5: local testing and tier 1 tools

| ID    | Decision                                                                                              | Status    |
| ----- | ----------------------------------------------------------------------------------------------------- | --------- |
| D4.15 | The same checks run on the laptop (pre-commit hooks) and in CI (tier 1), from one configuration.      | Delegated |
| D4.16 | Python transformation logic is pure functions, unit-tested with pytest against the dummy files.       | Delegated |
| D4.17 | No local SQL engine. BigQuery validates view SQL on `apply`; a smoke test validates the data.         | Delegated |
| D4.18 | The end-to-end test is a smoke test: upload dummy data, then compare row counts and query the views.  | Delegated |

| Tool                    | Checks                                                  | Laptop   | CI  |
| ----------------------- | ------------------------------------------------------- | -------- | --- |
| `terraform fmt -check`  | Formatting                                              | Yes      | Yes |
| `terraform validate`    | Syntax and references (after `init -backend=false`)    | Yes      | Yes |
| TFLint (Google ruleset) | Provider-specific mistakes, unused variables, naming    | Yes      | Yes |
| Trivy (`trivy config`)  | Security misconfigurations in Terraform                 | No       | Yes |
| Ruff                    | Python lint and formatting                              | Yes      | Yes |
| pytest                  | Python unit tests                                       | Optional | Yes |

- **One configuration (D4.15):** CI runs `pre-commit run --all-files`, so the laptop and CI cannot drift apart.
- **Pure functions (D4.16):** envelope removal and HTML cleaning take data in and return data out; the entry point does the Cloud Storage and BigQuery calls around them.
- **No DuckDB (D4.17):** the SQL is BigQuery-specific (`UNNEST`, `STRUCT`, placeholders); a translation would test the translation, not the real SQL.
- **Smoke test (D4.18):** `apply`, upload the dummy data, compare records written with rows loaded per file, query each view once.

<br>

## Topic 3: cost control and teardown

| ID    | Decision                                                                                                       | Status    |
| ----- | -------------------------------------------------------------------------------------------------------------- | --------- |
| D4.19 | The bootstrap script creates a small billing budget with email alerts at 50 %, 90 % and 100 %.                 | Delegated |
| D4.20 | Terraform creates the Artifact Registry repository for the function's container images, with a cleanup policy. | Delegated |
| D4.21 | A teardown checklist in the README: `destroy`, check for leftovers, delete the project at the very end.        | Delegated |

- **Budget (D4.19):** an alert, not a cap. It belongs to the billing account and survives `destroy`. Expected spending is close to zero; the free tiers cover this size.
- **Image repository (D4.20):** by default, Google creates a repository for function images that Terraform does not own, so images pile up after `destroy`. A Terraform-owned repository with a cleanup policy avoids that.
- **Teardown (D4.21):** `destroy`, check for leftover buckets, datasets, functions and images; keep the project between demonstrations; delete it at the very end. Logs and Google-managed service agents are harmless leftovers.

<br>

## Topic 4: public repository (reminder)

The maintainer handles this topic.

- **`.gitignore`:** `*.tfstate*`, `.terraform/`, `*.tfvars` (except `*.tfvars.example`), `data/raw/`, the function's generated zip, `.venv/`, `__pycache__/`, `.env`.
- **Commit:** `.terraform.lock.hcl` (pins provider versions), the JSON schema files, the `.sql` files, the dummy data.
- **Dummy data:** as defined in the [glossary](08_claude_decision_overview.md#data-glossary). Edge cases are covered, except posts without tags (D3.5).
- **GitHub push protection:** free for public repositories; blocks pushes that contain recognised secrets.
- **Looker Studio (if built):** the report stays private; the README shows screenshots.
