# Step 1: repository, dummy data, pre-commit

**Branch:** `chore/repository-and-dummy-data`

<br>

## In plain words

The new repository gets its foundation before any cloud code exists:

- **Dummy data:** 6 export files with made-up content, in the same shape as the real Wix exports. Every later check runs on them.
- **Privacy:** git ignores the full dataset, the project-specific values and Terraform's working files from the first commit, so they can never be pushed by mistake.
- **pre-commit:** general file checks run at every `git commit`. Later steps add the Terraform, Python and shell checks.
- **Simulated Part A:** `simulation/README.md` records the values Part A would hand over. Every later step uses them.

<br>

## Technical details

### Repository

- **New GitHub repository,** with `main` as the default branch.
- **Folders created now:** `data/dummy/blogs/`, `data/dummy/articles/`, `simulation/`. Every other folder comes with the step that fills it.

### Dummy data

Add the 6 dummy files to `data/dummy/`. Keep them exactly as exported: no reformatting, no renamed fields.

| File                               | Wix wrapper                  | Records |
| ---------------------------------- | ---------------------------- | ------- |
| `blogs/blog_posts.json`            | None: a plain list           | 75      |
| `blogs/blog_categories.json`       | None: a plain list           | 6       |
| `blogs/blog_tags.json`             | None: a plain list           | 11      |
| `blogs/members.json`               | `_items`, with `_totalCount` | 5       |
| `articles/articles.json`           | `items`, with `totalCount`   | 12      |
| `articles/article_categories.json` | `items`, with `totalCount`   | 5       |

- **3 shapes in 6 files:** the Wix exports wrap their records differently. Step 3 removes the wrappers and checks each record count against the stated total.
- **Field names stay as Wix wrote them** (`_id`, `categoryIds`, `_totalCount`), so the same code and schemas fit the full dataset later.

### `.gitignore`

All entries go in now, including those for later steps. One place, decided once.

| Entry                                                    | Why                                                                            |
| -------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `data/raw/`                                              | The full dataset is private; only the dummy data is committed                  |
| `terraform.tfvars`, `backend.hcl`                        | Project-specific values; their `.example` copies are committed                 |
| `.terraform/`, `*.tfstate*`                              | Terraform's downloaded providers and state; state can hold secrets             |
| `terraform/modules/content_pipeline/.terraform.lock.hcl` | `terraform test` creates it; only the root module's lock file decides versions |
| `.build/`                                                | Build output: the function zip, NDJSON files, simulated results                |
| `__pycache__/`, `.venv/`, `.pytest_cache/`               | Python's generated files                                                       |

### Python environment

```bash
python3.14 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements-dev.txt
pre-commit install
```

- **`requirements-dev.txt`:** laptop-only tools with exact versions. For now only `pre-commit`; steps 3 and 4 add pytest, Ruff and the BigQuery client.
- **`pre-commit install`:** adds a git hook, so the checks run at every `git commit` without being called.

### pre-commit

`.pre-commit-config.yaml` with one repository for now: `pre-commit/pre-commit-hooks` (`v6.0.0`).

| Hook                      | Catches                                        |
| ------------------------- | ---------------------------------------------- |
| `trailing-whitespace`     | Spaces at line ends (fixes them)               |
| `end-of-file-fixer`       | A missing final newline (fixes it)             |
| `check-yaml`              | Broken YAML, such as the later CI workflows    |
| `check-json`              | Broken JSON, such as the data and schema files |
| `check-merge-conflict`    | Leftover `<<<<<<<` markers                     |
| `check-added-large-files` | A large file committed by mistake              |
| `detect-private-key`      | A private key committed by mistake             |

**The 2 fixers skip `data/`** (`exclude: ^data/`): they would otherwise rewrite the dummy data, and it must stay exactly as exported.

**Notes in the file:** one comment at the top ("each step adds the hooks for its own language"), one above the `exclude` lines (why `data/` is skipped).

### `simulation/README.md`

Short now; later steps add their simulated checks to it.

- **Why simulate:** no real GCP project is available; Part B needs only Part A's values, not Part A itself.
- **The simulated values:** project `content-demo-sim`, region `europe-west1`, state bucket `content-demo-sim-tfstate`.
- **Derived names:** data bucket `content-demo-sim-content-data`, source bucket `content-demo-sim-content-source`.
- **Labelling rule:** every simulated result is labelled as simulated.

### `progress.md`

The build log, kept in the repository root and updated at the end of every step.

| Section            | Content                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| Status             | One line: the date, the last merged step, the next step                 |
| Merged branches    | One row per pull request: number, branch, step                          |
| What each step changed | A few bullets per step: what was built, how it was checked, and any **deviation from this plan** with its reason |
| Open points        | Anything found but not fixed yet                                        |

- **Why:** the plan says what should happen; `progress.md` says what did. A new session (yours or Claude's) starts from it.
- **Deviations go here,** not into the plan files.

### Claude Code set-up

- **`CLAUDE.md`:** the story in short, and a pointer to this plan: `rebuild_plan/` in the planning repository (GitHub link and local clone path). The plan decides; the repository implements it. Read `progress.md` at the start of every session.
- **`.claude/settings.json`:**
  - `additionalDirectories`: the local clone of the planning repository, so Claude can read the plan.
  - Allow the offline checks (`terraform fmt`, `validate`, `test`, `tflint`, `pytest`).
  - Deny everything that changes the cloud (`terraform apply`, `destroy`, `gcloud` writes, `bq load`, the upload script and the Part A stand-in script).

### Docs: the 2 skeletons

The docs grow with the code: every later step adds its part, so they are never out of date.

- **`README.md`:** the title; one paragraph on what the project is, with the link to the original Wix project; the story (Part A and Part B, in 3 sentences); a quote block: no real GCP is used, every simulated result is labelled.
- **`docs/decisions.md`:** one table per topic with 3 columns: decision, why, what it costs. First rows: Part A and Part B; the ownership rule; simulation instead of real GCP.
- **Plain words first** in both; technical details lower down.

### Commits

1. `chore(claude): add project guide and permission settings`
2. `chore: add .gitignore`
3. `chore(data): add dummy blog and article data`
4. `ops(ci): add pre-commit with general file checks`
5. `docs: add simulation README`
6. `docs: add progress.md`
7. `docs: add README.md and decisions.md skeletons`

<br>

## Done when

- `pre-commit run --all-files` passes.
- The 6 files hold the record counts in the table above, and `pre-commit run --all-files` leaves them unchanged.
- `git check-ignore -v data/raw/test.json terraform.tfvars` names the matching `.gitignore` line for each.
- `progress.md` records step 1 and names step 2 as next.
- The pull request is merged into `main`.
