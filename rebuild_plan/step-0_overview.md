# Step 0: overview

What the project builds, what it may assume, and how the repository is laid out. Nothing is built in this step.

<br>

## In plain words

**The project:** a cloud version of a local Docker and Jupyter project, the [Wix Content Intelligence Pipeline](https://github.com/beatanemeth/Wix-Content-Intelligence-Pipeline). Blog and article exports from a Wix site are uploaded, cleaned, loaded into BigQuery and turned into answers to 4 content questions.

**The story:** a company already runs a GCP platform. The pipeline is a new feature on it.

| Part       | Owner                   | What it is                                                                 |
| ---------- | ----------------------- | -------------------------------------------------------------------------- |
| **Part A** | Team A (platform team)  | The shared basics: project, billing, APIs, Terraform state bucket          |
| **Part B** | Team B (the maintainer) | The content pipeline, as a standalone Terraform module                     |

- **Ownership rule:** "Would a second feature need it?" Yes: Part A. No: Part B.
- **Part B changes nothing in Part A.** It only adds resources and permissions for itself.
- **Part A is never built here.** Part B is built against the values Part A would hand over.

**Simulation only:** no real GCP is used. Every cloud check runs on the laptop: a mocked Google provider for Terraform, a BigQuery emulator in Docker, and fake cloud clients for the Python tests. Every simulated result is labelled as simulated. A real deployment comes later (step 9) and needs no code change.

<br>

## Technical details

### The pipeline

| Stage          | Service                     | What happens                                                                  |
| -------------- | --------------------------- | ----------------------------------------------------------------------------- |
| **Upload**     | Upload script (laptop)      | The export files go to the data bucket, `raw/<group>/`; a `_READY` marker last |
| **Processing** | Cloud Run function          | Started by the marker; removes the Wix wrapper, cleans HTML, writes NDJSON to `processed/` |
| **Staging**    | BigQuery `content_staging`  | 6 tables, loaded 1:1 from `processed/` with strict schemas                   |
| **Cleaning**   | BigQuery `content_cleaned`  | 3 views: names joined in, one row per post or article                         |
| **Analysis**   | BigQuery `content_analysis` | 4 views: one answer per question                                              |

**The data:** 2 groups, 6 files, in the shape of the real Wix exports.

| Group      | Files                                                         |
| ---------- | ------------------------------------------------------------- |
| `blogs`    | `blog_posts`, `blog_categories`, `blog_tags`, `members`       |
| `articles` | `articles`, `article_categories`                              |

- **`data/dummy/`:** made-up content, committed to git, used by every check.
- **`data/raw/`:** the full dataset, git-ignored, used only in steps 8 and 9.
- **Two meanings of "raw":** on the laptop, `data/raw/` is the full dataset. In the data bucket, `raw/` holds whatever the upload script sent (dummy or full): the exports before cleaning, as opposed to `processed/`.

### Simulated Part A values

Made up; they point to nothing real.

| Value             | Simulated value            | Enters Part B as                    |
| ----------------- | -------------------------- | ----------------------------------- |
| Project ID        | `content-demo-sim`         | `project_id` in `terraform.tfvars`  |
| Region            | `europe-west1`             | `region` in `terraform.tfvars`      |
| State bucket name | `content-demo-sim-tfstate` | `bucket` in `backend.hcl`           |

**What Part B assumes Part A already did** (none of it is needed in the simulation):

- **Billing** linked to the project.
- **10 APIs enabled:** `storage`, `cloudfunctions`, `run`, `cloudbuild`, `artifactregistry`, `eventarc`, `pubsub`, `bigquery`, `iam`, `cloudresourcemanager`.
- **Cloud Storage service agent** holds `roles/pubsub.publisher`, so a finished upload can publish an event.
- **Team B** holds `Owner` on the project.

The module enables no APIs itself: a `destroy` could switch them off for every other feature in the project.

### Repository layout

The Terraform part follows Google's [best practices for style and structure](https://docs.cloud.google.com/docs/terraform/best-practices/general-style-structure) and [for root modules](https://docs.cloud.google.com/docs/terraform/best-practices/root-modules).

```text
.
├── terraform/
│   ├── modules/content_pipeline/   # Part B: the module (the recipe)
│   │   ├── main.tf                 # shared names and labels (locals)
│   │   ├── storage.tf              # 2 buckets
│   │   ├── bigquery.tf             # datasets, tables, views
│   │   ├── iam.tf                  # service accounts and grants
│   │   ├── function.tf             # zip, function, trigger
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf             # minimum versions only
│   │   ├── README.md
│   │   ├── files/                  # read by Terraform, never run by it
│   │   │   ├── function/           # main.py, requirements.txt
│   │   │   └── schemas/            # one JSON schema per staging table
│   │   ├── templates/              # read with templatefile()
│   │   │   ├── cleaned/            # *.sql.tftpl
│   │   │   └── analysis/           # *.sql.tftpl
│   │   └── tests/                  # *.tftest.hcl, mocked Google provider
│   └── environments/dev/           # the root module (the cook): run terraform here
│       ├── main.tf                 # provider and module call
│       ├── backend.tf              # remote state; bucket from backend.hcl
│       ├── versions.tf             # exact version ranges
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars.example
│       ├── backend.hcl.example
│       └── .terraform.lock.hcl     # committed
├── helpers/                        # not called by Terraform
│   ├── upload_data.sh
│   ├── bootstrap.sh                # Part A stand-in, real GCP only
│   └── data_checks.sql
├── simulation/                     # emulator checks and their README
├── tests/function/                 # pytest for the function code
├── data/                           # dummy/ committed; raw/ git-ignored
├── docs/                           # decisions; the runbook (step 9)
├── assets/                         # the pipeline diagram (PNG)
└── .github/workflows/              # offline CI
```

| Term           | Meaning                                                                                       |
| -------------- | --------------------------------------------------------------------------------------------- |
| **Module**     | Reusable Terraform code. Holds the resources; has no provider settings and no state of its own |
| **Root module** | The folder where `terraform init`, `plan` and `apply` run. Sets the provider, the backend and the values; calls the module |
| **`environments/dev/`** | The only environment. A `prod/` would be a copy of this folder with its own values and state |
| **`.tftpl`**   | A file with `${...}` placeholders that `templatefile()` fills. The SQL is not runnable until Terraform fills it |

**Naming:** Terraform names use underscores and are singular (`google_storage_bucket.data`). A resource that is the only one of its type is named `main`. Every GCP name starts with `content`.

### Notes inside Terraform files

- **One short comment** above a resource or a non-obvious argument, saying **why**, not what.
- Each step lists the notes its files should carry.
- No comment repeats what the code already says.

### Tools

Versions used in the first build. Take the latest at the start, then pin them.

| Tool                    | Version    | Used for                                   |
| ----------------------- | ---------- | ------------------------------------------ |
| Terraform               | 1.16       | Infrastructure as code                     |
| Google provider         | 8.6        | GCP resources                              |
| Archive provider        | 2.8        | Zipping the function code                  |
| TFLint                  | 0.64       | Terraform linting                          |
| Python                  | 3.14       | Function, tests, simulation scripts        |
| Docker                  | Any recent | Runs the BigQuery emulator                 |
| goccy/bigquery-emulator | 0.8.1      | Local BigQuery for the simulated checks    |
| pre-commit              | Latest     | Checks at every commit                     |

**Python 3.14 on the laptop too,** not only in CI: older versions reach end of life and the Google client libraries start to warn.

### The steps

One step, one branch, one pull request. Each step adds its own CI checks.

| Step | Topic                                          | Branch                            |
| ---- | ---------------------------------------------- | --------------------------------- |
| 1    | Repository, dummy data, pre-commit             | `chore/repository-and-dummy-data` |
| 2    | Terraform skeleton and the 2 buckets           | `feature/terraform-skeleton`         |
| 3    | Python cleaning, on the laptop                 | `feature/python-cleaning`            |
| 4    | BigQuery datasets, staging tables, data checks | `feature/bigquery-staging`           |
| 5a   | Service accounts and grants                    | `feature/service-accounts`           |
| 5b   | Function, trigger, upload script               | `feature/function-and-trigger`       |
| 6    | Cleaned views                                  | `feature/cleaned-views`              |
| 7    | Analysis views                                 | `feature/analysis-views`             |
| 8    | Full dataset run                               | `docs/full-dataset-run` (records and docs) |
| 9    | Real deployment (runbook now, run later)       | `docs/runbook`; later `ops/real-deployment` |

**Every step file has 3 parts:**

1. **In plain words:** what the step builds and why.
2. **Technical details:** resources, settings, commands, and the notes for the Terraform files.
3. **Done when:** the simulated check that proves the step works.

<br>

## Done when

- The maintainer can explain Part A, Part B and the ownership rule.
- The repository layout and the role of each folder are clear.
- The tools are installed: `terraform version`, `tflint --version`, `python3 --version` (3.14) and `docker --version` answer.
