# Step 6: repository layout (which file goes where)

**Planning file:** [06_repository-layout.md](../v2_architecture/06_repository-layout.md)

**Status:** closed

**Built in:** phases 0 to 9 (each phase adds its files)

**Review:** how this step fits the story and the Data Engineering aim is at the end of this file ([Review](#review)).

<br>

## In plain words

**In one sentence:** one repository, like a filing cabinet with one drawer per kind of work.

| Drawer (folder)      | Holds                                                                 |
| -------------------- | --------------------------------------------------------------------- |
| `bootstrap/`         | The script that stands in for Team A (step 1)                         |
| `terraform/`         | The recipe (the module) and the cook (the deployment folder) (step 2) |
| `scripts/`           | The upload script and the data checks                                 |
| `tests/`             | The Python tests                                                      |
| `data/`              | The dummy data (in git) and the full dataset (laptop only)            |
| `docs/`              | Diagrams and the decision summary                                     |
| `.github/workflows/` | The automatic checks on GitHub (step 7)                               |

**The recipe is self-contained:** everything the pipeline needs sits in one folder, `terraform/modules/content_pipeline/`: the Terraform files, the Python code, the table schemas and the SQL. Pick up that folder, and you have the whole feature.

**The tests sit outside the recipe:** Terraform zips the function's folder and sends it to Google Cloud. Tests inside it would be sent too, for nothing.

**What stays on the laptop only (never in git):**

- **The full dataset:** private.
- **The real project ID and state bucket name:** private. Git holds example files with placeholders.
- **Generated files:** the function's zip, Python caches, downloaded Terraform plugins.

**One name runs through the files too:** `schemas/blog_posts.json` defines the staging table `blog_posts`; `sql/cleaned/blog_posts.sql` defines the cleaned view `blog_posts`.

<br>

## Technical details

<br>

### What this step decides

| ID   | Decision                                                                                     |
| ---- | -------------------------------------------------------------------------------------------- |
| S6.1 | One repository, one top-level folder per concern                                             |
| S6.2 | Module at `terraform/modules/content_pipeline/`; deployment at `terraform/deployment/`       |
| S6.3 | Terraform files split by topic                                                               |
| S6.4 | Python tests in `tests/`, outside the module; pytest finds the code through `pythonpath`     |
| S6.5 | The zip goes to `.build/` in the deployment folder (git-ignored); `__pycache__` excluded      |
| S6.6 | Schema and SQL files are named after their table or view; one SQL subfolder per layer         |
| S6.7 | Committed: dummy data, lock file, example files. Git-ignored: full dataset, real values, build output |
| S6.8 | The upload script creates the empty `_READY` marker itself                                   |

<br>

### Refinement

N: accepted; revisited in phase 3, when the tests are first run.

| ID  | Finding | Proposal |
| --- | ------- | -------- |
| N   | **The tests' dependencies have no home.** The Python workflow (step 7) says "install dependencies", but from where? `function/requirements.txt` lists what the function needs in the cloud; it does not list `pytest`. Tests also import the function's libraries. | Add `requirements-dev.txt` at the root: it includes the function's `requirements.txt` and adds `pytest`. The laptop's `.venv` and the CI workflow both install from it. |

**Already updated from earlier steps:** the tree below has 2 module variables (E), the `author_activity` view (J), and "standalone" instead of "reusable" (C).

<br>

### Final implementation: folder tree

```text
gcp-content-intelligence-platform_implementation/
├── README.md                              Story, architecture, runbook, results
├── CLAUDE.md                              Guide for Claude Code
├── .claude/                               Claude Code rules and settings
├── .gitignore
├── .pre-commit-config.yaml                Local checks at each commit (step 7, S)
├── .tflint.hcl                            TFLint settings (step 7)
├── pyproject.toml                         pytest settings, incl. pythonpath (S6.4)
├── requirements-dev.txt                   pre-commit; from phase 3 also the function requirements, pytest, ruff (N)
├── .github/
│   └── workflows/
│       ├── terraform-checks.yml           fmt, validate, TFLint (step 7)
│       └── python-tests.yml               pytest (step 7)
├── bootstrap/
│   └── bootstrap.sh                       Part A stand-in (step 1)
├── terraform/
│   ├── modules/
│   │   └── content_pipeline/              Part B: the standalone module
│   │       ├── README.md                  Inputs, outputs, prerequisites
│   │       ├── versions.tf                Minimum versions (>=)
│   │       ├── variables.tf               project_id, region
│   │       ├── locals.tf                  Prefix, names, labels, table and view maps
│   │       ├── outputs.tf                 data_bucket_name, dataset_ids, function_name
│   │       ├── storage.tf                 2 buckets
│   │       ├── function.tf                Zip, zip upload, function
│   │       ├── bigquery.tf                3 datasets, 6 tables, 9 views
│   │       ├── iam.tf                     3 service accounts, 8 grants
│   │       ├── function/                  Zipped and deployed
│   │       │   ├── main.py
│   │       │   └── requirements.txt
│   │       ├── schemas/                   One JSON file per staging table
│   │       │   ├── blog_posts.json
│   │       │   ├── blog_categories.json
│   │       │   ├── blog_tags.json
│   │       │   ├── members.json
│   │       │   ├── articles.json
│   │       │   └── article_categories.json
│   │       └── sql/
│   │           ├── cleaned/
│   │           │   ├── blog_posts.sql
│   │           │   ├── articles.sql
│   │           │   └── content.sql
│   │           └── analysis/
│   │               ├── content_per_month.sql
│   │               ├── tag_frequency.sql
│   │               ├── engagement_by_category.sql
│   │               ├── author_activity.sql
│   │               ├── top_posts_per_category.sql
│   │               └── article_text_stats.sql
│   └── deployment/                        Root configuration: calls the module
│       ├── versions.tf                    backend "gcs", provider versions (~>)
│       ├── providers.tf                   google provider
│       ├── main.tf                        module "content_pipeline"
│       ├── variables.tf                   project_id, region
│       ├── outputs.tf                     The module's 3 outputs
│       ├── .terraform.lock.hcl            Committed
│       ├── terraform.tfvars.example       Committed; copied to terraform.tfvars
│       └── backend.hcl.example            Committed; copied to backend.hcl
├── scripts/
│   ├── upload_data.sh                     Upload a dataset, marker last
│   └── data_checks.sql                    Row counts and data quality checks
├── tests/
│   └── function/
│       └── test_main.py                   pytest against the dummy data
├── data/
│   ├── dummy/                             Committed
│   │   ├── blogs/
│   │   └── articles/
│   └── raw/                               Git-ignored: the full dataset
└── docs/                                  Diagrams, decision summary
```

#### `pyproject.toml`

```toml
[tool.pytest.ini_options]
pythonpath = ["terraform/modules/content_pipeline/function"]
testpaths = ["tests"]
```

Tests then use `import main` directly.

#### `requirements-dev.txt` (N)

```text
pre-commit
-r terraform/modules/content_pipeline/function/requirements.txt
pytest
ruff
```

`pre-commit` from phase 0 (step 7, S); the other lines from phase 3.

Versions pinned at implementation.

#### Done when

- Each phase adds its files to this tree; nothing outside it.
- `git status` never shows a file from `data/raw/`, `terraform.tfvars`, `backend.hcl` or `.build/`.

<br>

## Review

**Fits the story:** yes. Everything in this repository is Team B's work, except `bootstrap/`, which stands in for Team A. The module folder holds the whole feature, so "Part B as a standalone module" is visible in the tree itself.

**Fits the Data Engineering aim:** yes. A reviewer opening the repository sees the pipeline's layers as files: schemas for staging, SQL for cleaned and analysis, Python for the processing, tests next to them. One name per table runs through every folder.

**Simplest choice:** one repository, standard Terraform folders, no environment folders (one deployment).

**N. The tests' shopping list is missing (accepted).** The tests need `pytest` and the function's libraries, but no file lists them.
→ One small file, `requirements-dev.txt`, used on the laptop and on GitHub.
