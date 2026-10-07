# Step 8: build phases (the order of the work)

**Planning file:** [08_build-phases.md](../architecture/08_build-phases.md)

**Status:** closed

**Built in:** this step is the build itself (phases 0 to 9)

**Review:** how this step fits the story and the Data Engineering aim is at the end of this file ([Review](#review)).

<br>

## In plain words

**In one sentence:** the project is built in 10 small steps; each adds one piece, ends with a check, and the next starts only when the check passes.

**Why small steps:** when a check fails, the cause can only be in the piece just added. A big-bang build would fail with ten possible causes at once.

**Two rules for the order:**

1. **Follow the data:** storage first, then the Python cleaning, then BigQuery, then the views.
2. **Test each piece alone before connecting it:** the Python code runs on the laptop before it runs in the cloud. The BigQuery tables are filled by hand once before the function fills them automatically.

| Phase | Builds                                         | Where          | Done when (in short)                                       |
| ----- | ---------------------------------------------- | -------------- | ---------------------------------------------------------- |
| 0     | Repository and dummy data                      | Laptop         | Folders in place; full dataset never in git                |
| 1     | Team A's stand-in (`bootstrap.sh`)             | GCP            | Project, services, state bucket, budget exist              |
| 2     | Terraform skeleton and the 2 buckets           | GCP, GitHub    | Buckets exist; `destroy` and `apply` work                  |
| 3     | The Python cleaning, on the laptop             | Laptop, GitHub | Tests pass; 6 clean files with the right record counts     |
| 4     | BigQuery datasets and staging tables           | GCP            | The 6 clean files load by hand; row counts match           |
| 5     | Robot accounts, the function, the upload script | GCP           | An upload fills `processed/` and the tables by itself      |
| 6     | Cleaned views                                  | GCP            | One row per post or article; all names filled              |
| 7     | Analysis views                                 | GCP            | The 6 questions get plausible answers                      |
| 8     | Full dataset and rebuild test                  | GCP            | Everything removed, rebuilt, loaded with the real data; counts match |
| 9     | README and docs                                | GitHub         | Story, diagrams, runbook and results published             |

**One branch and one pull request per phase**, so GitHub's checks run on each phase.

<br>

## Technical details

<br>

### What this step decides

| ID   | Decision                                                                              |
| ---- | ------------------------------------------------------------------------------------- |
| S8.1 | 10 phases (0 to 9); each ends with a check; a phase starts only when the previous check passes |
| S8.2 | The Python transformation is built and tested locally before any cloud compute exists |
| S8.3 | Staging schemas are tested with a manual `bq load` before the function automates the load |
| S8.4 | `destroy` and rebuild are tested in phase 2 (skeleton) and phase 8 (full system)      |
| S8.5 | One branch and one pull request per phase                                             |
| S8.6 | The upload script takes the data folder as an argument; the default is `data/dummy`   |
| S8.7 | Row counts and data quality are checked with `scripts/data_checks.sql`, run with `bq query` |
| S8.8 | The query results in the README come from the full dataset                            |

<br>

### Refinements

T and U: accepted.

| ID  | Finding | Proposal |
| --- | ------- | -------- |
| T   | **Phase 4 needs local NDJSON files, but nothing writes them.** Phase 3's tests write the files into a temporary folder that pytest deletes. Phase 4 loads "the local NDJSON files from phase 3" by hand. | Give `main.py` a small command-line entry: `python main.py data/dummy .build/ndjson` writes the 6 NDJSON files to a git-ignored folder. The same pure functions as in the cloud; a few lines. |
| U   | **Phase 2's outputs point to things that do not exist yet.** The module's `outputs.tf` lists 3 outputs from phase 2, but `dataset_ids` needs the datasets (phase 4) and `function_name` the function (phase 5). `validate` would fail on the missing resources. | Each output is added in the phase that creates its resource: `data_bucket_name` in phase 2, `dataset_ids` in phase 4, `function_name` in phase 5. |

**Already folded in from earlier steps:** the phase details below include the decisions of steps 1 to 7 (for example H and I in phase 5, J in phase 7, P to S in phases 0 to 3). Each phase also adds its pre-commit hook (step 7, S).

<br>

### Final implementation: the phases

#### Phase 0: repository and dummy data

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `chore/repository-and-dummy-data`                                            |
| Files    | `.gitignore`; `data/dummy/blogs/` (4 files), `data/dummy/articles/` (2 files) |
| Files (S) | `.pre-commit-config.yaml` (general file checks); `requirements-dev.txt` with `pre-commit` |
| GitHub   | Settings: branch protection on `main` (pull requests only, no required checks); push protection |
| Done when | D3.10 check: the dummy files have the same always-empty arrays as the full dataset; `git status` shows nothing from `data/raw/` |
| Status   | Data and `.gitignore` committed; pre-commit (S) still to add; not yet pushed  |

#### Phase 1: Team A's stand-in

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `ops/bootstrap-script`                                                       |
| Files    | `bootstrap/bootstrap.sh` (step 1); `.github/workflows/shell-checks.yml` (P)  |
| Runs     | The maintainer runs the script once (Claude never does)                      |
| Done when | 11 APIs enabled; `gs://<project-id>-tfstate` exists with versioning; budget visible; ShellCheck green |

#### Phase 2: Terraform skeleton and the 2 buckets

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `feat/terraform-skeleton`                                                    |
| Files    | Deployment: all files of step 2. Module: `versions.tf`, `variables.tf`, `locals.tf`, `outputs.tf` (`data_bucket_name` only, U), `storage.tf`. `.tflint.hcl`; `.github/workflows/terraform-checks.yml` |
| Done when | `terraform output data_bucket_name` prints the name; the state file is in `gs://<project-id>-tfstate/content-pipeline/`; `destroy` then `apply` works (S8.4); Terraform checks green |

#### Phase 3: the Python cleaning, on the laptop

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `feat/python-transformation`                                                 |
| Files    | `function/main.py` (pure functions: unwrap, clean HTML, write NDJSON; command-line entry, T); `function/requirements.txt`; `tests/function/test_main.py`; `pyproject.toml`; `requirements-dev.txt` (N); `.github/workflows/python-tests.yml` (Q, R) |
| Done when | pytest green; Ruff green; `python main.py data/dummy .build/ndjson` writes 6 files with the record counts below |

**Record counts:**

| File                 | Dummy data | Full dataset |
| -------------------- | ---------- | ------------ |
| `blog_posts`         | 75         | 113          |
| `blog_categories`    | 6          | 9            |
| `blog_tags`          | 11         | 17           |
| `members`            | 5          | 8            |
| `articles`           | 12         | 18           |
| `article_categories` | 5          | 8            |

#### Phase 4: BigQuery datasets and staging tables

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `feat/bigquery-staging`                                                      |
| Files    | `bigquery.tf` (3 datasets, 6 staging tables); `schemas/*.json` (6, explicit `mode` on every column); `outputs.tf` gains `dataset_ids` (U); `scripts/data_checks.sql` |
| By hand  | For each file in `.build/ndjson/`: `bq load --replace --source_format=NEWLINE_DELIMITED_JSON content_staging.<table> <file>` (no tolerance for bad records or unknown fields) |
| Done when | Every load succeeds; `data_checks.sql`: row counts match the dummy column above, 0 for every other check |

#### Phase 5: robot accounts, the function, the upload script

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `feat/function-and-trigger`                                                  |
| Files    | `iam.tf` (3 accounts, 8 grants, M with the condition M2); `function.tf` (zip, upload, function, `depends_on`); `main.py` gains the event entry point `handle_event` (15-minute limit H, count check I); `outputs.tf` gains `function_name` (U); `scripts/upload_data.sh` (data folder as argument, default `data/dummy`) |
| First `apply` | On a permission error, run `apply` again (S5.3)                         |
| Done when | Logs: 2 working runs, 12 that end at once; `processed/` holds 6 files; `data_checks.sql` as in phase 4; no count `ERROR` in the logs; a second upload gives the same counts |

#### Phase 6: cleaned views

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `feat/cleaned-views`                                                         |
| Files    | `sql/cleaned/blog_posts.sql`, `articles.sql`, `content.sql`; the view blocks in `bigquery.tf` (`deletion_protection = false`, K) |
| Done when | Each view has one row per post or article; `content` has their sum; no empty category or tag name; `articles` has `article_text` |

#### Phase 7: analysis views

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `feat/analysis-views`                                                        |
| Files    | The 6 SQL files in `sql/analysis/`, each opening with its question and metric definition (J) |
| Done when | Each view returns rows; one number checked by hand                          |

#### Phase 8: full dataset and rebuild test

No branch, unless a fix is needed. The maintainer runs:

1. `terraform destroy`
2. `terraform apply`
3. `scripts/upload_data.sh data/raw`
4. `bq query --use_legacy_sql=false < scripts/data_checks.sql`: counts match the full dataset column above; 0 for every other check.
5. **D3.5 check:** the 2 posts without tags still appear in the cleaned views and in `content`.
6. **Save the results** of the 6 analysis views for the README (S8.8).
7. `terraform destroy`

#### Phase 9: README and docs

| Item     | Content                                                                     |
| -------- | --------------------------------------------------------------------------- |
| Branch   | `docs/readme-and-results`                                                    |
| Content  | Story (step 0); diagrams; runbook; local check commands; results from phase 8; decision summary |

<br>

### `scripts/data_checks.sql`

One query built from `UNION ALL` blocks; each returns `check_name`, `table_name`, `result`.

| Check             | Counts                                                          | Expected                    |
| ----------------- | --------------------------------------------------------------- | --------------------------- |
| Row count         | Rows per staging table                                          | The record counts above     |
| Duplicate keys    | `_id` values that appear more than once, per table              | 0                           |
| Orphan references | Post `categoryIds`, `tagIds`, `memberId` and article `category` with no match in their lookup table | 0 |
| Missing titles    | Posts and articles with an empty or `NULL` title                | 0                           |

<br>

### Runbook

| When          | Step | Command                                                                  |
| ------------- | ---- | ------------------------------------------------------------------------ |
| Once          | 1    | `gcloud auth login` and `gcloud auth application-default login`          |
| Once          | 2    | `bootstrap/bootstrap.sh`                                                 |
| Once          | 3    | Copy `terraform.tfvars.example` and `backend.hcl.example`; fill in the values |
| Once          | 4    | `terraform init -backend-config=backend.hcl` (in `terraform/deployment/`) |
| Every run     | 5    | `terraform plan`, then `terraform apply`                                 |
| Every run     | 6    | `scripts/upload_data.sh` (dummy) or `scripts/upload_data.sh data/raw` (full) |
| Every run     | 7    | Read the function's logs (command verified at implementation)            |
| Every run     | 8    | `bq query --use_legacy_sql=false < scripts/data_checks.sql`              |
| Every run     | 9    | Query the views in `content_cleaned` and `content_analysis`              |
| Every run     | 10   | `terraform destroy`                                                      |

<br>

### Optional extensions

| Extension                                              | From  |
| ------------------------------------------------------ | ----- |
| Looker Studio report on the analysis views             | D0.2  |
| The function as a Cloud Run service with a Dockerfile   | D1.8  |
| `terraform plan` on pull requests with keyless login   | S7.1  |
| `time_sleep` after the IAM grants                      | S5.3  |
| Security scanner, SQL linter                           | Step 7 |

<br>

## Review

**Fits the story:** yes. Phase 1 plays Team A once; every later phase is Team B's work on top of it.

**Fits the Data Engineering aim:** yes, strongly:

- **Test each piece alone:** the Python code on the laptop, the schemas with a manual load, then the automation.
- **Data quality checks** in SQL after every load.
- **A full rebuild** from the repository and the laptop's files (phase 8).
- **Results from the real data** in the README.

**Simplest choice:** one piece per phase, one check per phase, no automation before the manual version works.

**T. Phase 4 needs files nobody writes (accepted).** Phase 4 loads the clean files from phase 3 by hand, but phase 3 only writes them into a temporary test folder that disappears.
→ One small command: `python main.py data/dummy .build/ndjson` writes the 6 files.

**Where the 6 files land, and why:**

```text
python main.py  data/dummy        .build/ndjson
                └─ input:          └─ output:
                   dummy data         6 cleaned NDJSON files
                   (committed)        (generated, not committed)
```

- **Nothing here is secret:** the output comes from the dummy data, so it could be visible at every level.
- **Why it still stays out of git:** it is **generated output**, like the function's zip. One command recreates it at any time; a committed copy would go out of date whenever the cleaning code changes.
- **No new `.gitignore` rule:** `.build/` is already ignored.
- **Rejected option:** committing the files (for example in `data/dummy_processed/`). Visible on GitHub, but it must be regenerated and committed again after every code change.

**U. Outputs that point to nothing yet.** Phase 2 would declare all 3 outputs, but the datasets and the function arrive only in phases 4 and 5. Terraform would refuse the code.
→ Each output arrives with its resource: one in phase 2, one in phase 4, one in phase 5.

**Phase 0 is done but not delivered:** its branch still needs to be pushed and merged.
