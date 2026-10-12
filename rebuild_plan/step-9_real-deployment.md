# Step 9: real deployment

**Branches:**

- **Now:** `docs/runbook`: the runbook and the Part A stand-in script, written and checked, never run.
- **Later, with a real GCP project:** `ops/real-deployment`: the records of the real run.

<br>

## In plain words

The same code, deployed to real GCP. Nothing in the module changes; only the way it is checked does.

- **Part A must exist first.** In the story, Team A built it. In practice, a small script stands in for Team A and creates the platform basics once.
- **Then Part B, as planned:** fill in the 2 private files, `init` with the real state bucket, `plan`, `apply`.
- **Then the pipeline runs for real:** upload, the function starts, the tables fill, the views answer.
- **Then everything is removed:** `destroy` deletes Part B with its data. The laptop still holds the source data, so the whole pipeline can be rebuilt at any time.
- **What only real GCP proves:** that Google accepts every resource, that the permissions really work, and that Eventarc really delivers the events. The simulation could not prove these.
- **Cost:** a few cents per run at this data size. A budget with email alerts guards the project.

<br>

## Technical details

### Prerequisites

- **A Google account** with a billing account.
- **Google Cloud CLI** (`gcloud`, which includes `bq`) installed on the laptop.
- **Login, twice:**
  - `gcloud auth login`: for `gcloud` and `bq` commands.
  - `gcloud auth application-default login`: for Terraform and the Google client libraries.

### Part A stand-in: `helpers/bootstrap.sh`

Run once, by hand, with `PROJECT_ID` and `BILLING_ACCOUNT` set. It creates what step 0 lists as Part A's job:

| #   | Creates                                  | How                                                                                                                              |
| --- | ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 1   | The project                              | `gcloud projects create`                                                                                                         |
| 2   | The billing link                         | `gcloud billing projects link`                                                                                                   |
| 3   | A budget with email alerts               | `gcloud billing budgets create`: a small monthly amount, alerts at 50, 90 and 100 %                                              |
| 4   | The 10 APIs, plus the Billing Budget API | `gcloud services enable`                                                                                                         |
| 5   | The Terraform state bucket               | `gcloud storage buckets create <project-id>-tfstate`: in `europe-west1`, versioning on, uniform access, public access prevention |
| 6   | The Cloud Storage service agent's grant  | Find the agent's email with `gcloud storage service-agent`; grant it `roles/pubsub.publisher` on the project                     |

- **`set -euo pipefail`,** and each step is safe to run again: the script checks whether a thing exists before creating it.
- **Why a script, not Terraform:** Part A is not this project's code. The script only has to produce its result, with no state to manage.
- **Why versioning on the state bucket:** an older state can be restored if a run corrupts it.
- **Team B's `Owner`:** comes with creating the project; no grant needed.
- **ShellCheck** checks it automatically: it sits in `helpers/`, which the workflow and the pre-commit hook already cover.

### Runbook: `docs/runbook.md`

Title: "Runbook for real GCP". One table: when, step, command. All commands from the repository root.

| When       | Step | Command                                                                                                                                                     |
| ---------- | ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Once       | 1    | Log in (both commands above); `gcloud config set project <project-id>`                                                                                      |
| Once       | 2    | `helpers/bootstrap.sh` with `PROJECT_ID` and `BILLING_ACCOUNT` set                                                                                          |
| Once       | 3    | In `terraform/environments/dev/`: copy `terraform.tfvars.example` and `backend.hcl.example` without `.example`; fill in the project ID and the state bucket |
| Once       | 4    | `terraform -chdir=terraform/environments/dev init -backend-config=backend.hcl`                                                                              |
| Every run  | 5    | `terraform -chdir=terraform/environments/dev plan`, then `apply`                                                                                            |
| Every run  | 6    | `helpers/upload_data.sh` (dummy data) or `helpers/upload_data.sh data/raw` (full dataset)                                                                   |
| Every run  | 7    | `gcloud functions logs read content-transform --region=europe-west1 --limit=50`                                                                             |
| Every run  | 8    | `bq query --use_legacy_sql=false < helpers/data_checks.sql`                                                                                                 |
| Every run  | 9    | Query the views in `content_cleaned` and `content_analysis`                                                                                                 |
| Every run  | 10   | `terraform -chdir=terraform/environments/dev destroy`                                                                                                       |
| At the end | 11   | `gcloud projects delete <project-id>`: removes Part A too, and stops all cost                                                                               |

**Below the table, 3 notes:**

- **The first `apply` may fail on a permission:** a new grant can take a minute to reach Cloud Build and Eventarc. Run `apply` again; Terraform rebuilds the failed function.
- **`terraform init` with the backend:** from now on, the state lives in the bucket, not on the laptop. `-backend=false` is only for the simulation.
- **The Claude Code settings deny these commands** on purpose: the maintainer runs them by hand.

### The real run (later)

Follow the runbook, then check what the simulation could not:

| Check                                      | How                                                  | Expected                              |
| ------------------------------------------ | ---------------------------------------------------- | ------------------------------------- |
| Google accepts every resource              | `apply`                                              | No error (after at most one rerun)    |
| No drift                                   | A second `plan` right after `apply`                  | "No changes"                          |
| The build account reads its code           | The function deploys                                 | `ACTIVE` in the console               |
| Eventarc delivers; only `_READY` does work | The logs after an upload of the dummy data           | 6 `File loaded` lines, records = rows |
| The loads are strict and complete          | `data_checks.sql`                                    | The dummy counts; 0 everywhere else   |
| The views work in real BigQuery            | Query each view                                      | Rows in every view                    |
| Full dataset                               | Upload `data/raw`, then the 2 checks above           | The step 8 numbers                    |
| Simulation and reality agree               | Compare the analysis views with the step 8 CSV files | The same numbers                      |
| `destroy` removes everything               | `destroy`, then list buckets and datasets            | Only the state bucket is left         |

**If a check fails:** note it in `progress.md`, fix it on its own branch, and add the case to `simulation/README.md` under "what the simulation cannot prove".

### Docs

| Document               | Now (`docs/runbook`)                       | Later (`ops/real-deployment`)                                         |
| ---------------------- | ------------------------------------------ | --------------------------------------------------------------------- |
| `docs/runbook.md`      | Created; title says "never run"            | "never run" removed; the date of the real run added                   |
| `README.md`            | Link to the runbook                        | Results labelled "real GCP", next to or instead of the simulated ones |
| `simulation/README.md` | The real Part A: the script, how to run it | What the real run confirmed                                           |
| `docs/decisions.md`    | Part A stand-in as a script, not Terraform | Anything the real run changed                                         |
| `progress.md`          | This step, part 1                          | The real run: date, each check's result                               |

### CI

**Unchanged.** CI holds no GCP credentials and never runs `plan` or `apply`. A real deployment from CI would need Workload Identity Federation, which is out of scope.

### Commits (now)

1. `ops(scripts): add the Part A stand-in script`
2. `docs: add the runbook for real GCP`
3. `docs: link the runbook and record the stand-in decision`

<br>

## Done when

**Now:**

- `shellcheck helpers/bootstrap.sh` reports nothing.
- Every command in the runbook uses the paths and names of the code.
- `progress.md` records this step: what changed, the next action (a real GCP project).

**Later:**

- Every check of the real run gives its expected result.
- `destroy` leaves only the state bucket; the project is deleted at the end.
- The README shows real results, labelled as real.
