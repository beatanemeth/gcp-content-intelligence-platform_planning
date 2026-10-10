# Step 3: Python cleaning, on the laptop

**Branch:** `feature/python-cleaning`

<br>

## In plain words

The function's cleaning logic, written and tested on the laptop before any cloud part exists.

- **What Python does:** only 2 things.
  - **Unwrap:** remove the Wix wrapper around the records, and check the record count against the total the export states.
  - **Clean HTML:** turn each article's HTML body into plain text.
- **What Python does not do:** joins, renames or calculations. Those are SQL, in BigQuery (steps 6 and 7), where they stay visible and testable.
- **Output:** NDJSON, one JSON record per line: the format BigQuery loads.
- **Pure functions:** no cloud clients yet. Step 5b adds the entry point that reads from and writes to Cloud Storage and BigQuery.
- **A command-line entry:** the same code writes the 6 NDJSON files on the laptop. Step 4 loads them into the BigQuery emulator.

<br>

## Technical details

### Files

| Path                                                         | Content                                         |
| ------------------------------------------------------------ | ----------------------------------------------- |
| `terraform/modules/content_pipeline/files/function/main.py`  | The cleaning functions                          |
| `terraform/modules/content_pipeline/files/function/requirements.txt` | The function's libraries: `beautifulsoup4` |
| `tests/function/test_main.py`                                | pytest tests                                    |
| `pyproject.toml`                                             | pytest and Ruff settings                        |
| `requirements-dev.txt`                                       | Adds pytest and Ruff; includes the function's `requirements.txt` with `-r` |

- **The function code sits inside the module (`files/function/`):** step 5b zips this folder and deploys it. Terraform reads it but never runs it.
- **Tests sit outside the module (`tests/`):** they must not end up in the zip.
- **`pyproject.toml` holds tool settings only:** the function is deployed as a zip, not installed as a package.

### `main.py`: the functions

| Function         | Does                                                                                           |
| ---------------- | ---------------------------------------------------------------------------------------------- |
| `log`            | Prints one JSON line with `severity` and `message`; Cloud Logging reads `severity` as the log level |
| `unwrap`         | Returns the records of one export (see the table below); logs an `ERROR` if the count differs from the total, then goes on |
| `clean_html`     | Beautiful Soup `get_text(" ")`, then all whitespace collapsed to single spaces                 |
| `transform`      | `unwrap`; for `articles` also `clean_html` on `articleText`                                    |
| `to_ndjson`      | One `json.dumps` per record, each followed by a newline; `ensure_ascii=False` keeps accented letters |
| `process_folder` | For each group and file: read, transform, write `<target>/<group>/<name>.ndjson`; logs `INFO` with the record count |

**Fixed lists in the code:**

- **`GROUPS`:** `blogs` (4 files) and `articles` (2 files). A missing file fails loudly instead of silently shrinking the group.
- **`ENVELOPES`:** the record key and its total key, `_items` and `_totalCount`, `items` and `totalCount`.

**The 3 export shapes:**

| Shape                         | Files                                           | `unwrap` returns           |
| ----------------------------- | ----------------------------------------------- | -------------------------- |
| Plain list                    | `blog_posts`, `blog_categories`, `blog_tags`    | The list; no total to check |
| `_items` with `_totalCount`   | `members`                                       | `_items`, checked          |
| `items` with `totalCount`     | `articles`, `article_categories`                | `items`, checked           |
| Anything else                 | None                                            | Raises `ValueError`        |

**Why the count mismatch logs and goes on:** a retry cannot fix the source file. The `ERROR` line makes it visible; the loaded rows show what arrived.

**Why `get_text(" ")`:** without a separator, `<strong>Lorem</strong>Ipsum` becomes `LoremIpsum`.

**Overwrite, never append:** a second run gives the same files (idempotent).

### Command line

```bash
python terraform/modules/content_pipeline/files/function/main.py data/dummy .build/ndjson
```

Writes 6 NDJSON files to the git-ignored `.build/ndjson/blogs/` and `.build/ndjson/articles/`.

### Tests: `tests/function/test_main.py`

| Test                                | Proves                                                              |
| ----------------------------------- | ------------------------------------------------------------------- |
| Unwrap each dummy export            | The record counts match step 1's table; no `ERROR` logged           |
| A cut-off export                    | Keep only 2 records of a wrapped export: `unwrap` still returns them and logs an `ERROR` with both numbers |
| An unknown shape                    | `unwrap` raises `ValueError`                                        |
| `clean_html` gives one line of words | No tags, no double spaces, no glued words                          |
| `clean_html` decodes entities       | `&amp;` becomes `&`, `&nbsp;` a space                               |
| `clean_html` on the dummy articles  | Every article has text; no `<`, `>`, line break or double space     |
| `transform` cleans only `articleText` | Other fields and other files stay unchanged                       |
| `to_ndjson`                         | One line per record, each a valid JSON object                       |
| `process_folder`                    | Writes the 6 files into a temporary folder (`tmp_path`); a second run gives the same files |

`@pytest.mark.parametrize` runs one test function for each of the 6 files.

### `pyproject.toml`

| Setting                     | Value                                              | Why                                              |
| --------------------------- | -------------------------------------------------- | ------------------------------------------------ |
| pytest `testpaths`          | `["tests"]`                                        | Where the tests are                              |
| pytest `pythonpath`         | `["terraform/modules/content_pipeline/files/function"]` | Tests import `main` from its own folder, as the runtime does |
| Ruff `target-version`       | `"py314"`                                          | The function's runtime version                   |
| Ruff `src`                  | `[".", "terraform/modules/content_pipeline/files/function"]` | Import sorting treats `main` as own code |
| Ruff lint `select`          | `["E", "F", "I", "B", "UP"]`                       | Defaults, plus import order, likely bugs, modern syntax |

### Versions

Exact versions (`==`), in both requirements files. First build: `beautifulsoup4` 4.15.0, `pytest` 9.1.1, `ruff` 0.16.10.

### pre-commit

Add `astral-sh/ruff-pre-commit` with `ruff-check` and `ruff-format`. Its `rev` matches the Ruff version in `requirements-dev.txt`, so the laptop and CI agree.

### CI: `.github/workflows/python-tests.yml`

- **Runs on:** pull requests and pushes to `main`, when the function code, `tests/`, `data/dummy/`, `pyproject.toml`, `requirements-dev.txt` or the workflow changes.
- **`permissions: contents: read`.**
- **Steps:** set up Python 3.14; `pip install -r requirements-dev.txt`; `pytest`; `ruff check`; `ruff format --check`.

### Notes in the files

| File               | Note                                                                       |
| ------------------ | -------------------------------------------------------------------------- |
| `main.py`          | `GROUPS`: a missing file fails loudly. `unwrap`: log and go on, a retry cannot fix the source. `clean_html`: why the space separator. `process_folder`: overwrite, so a second run gives the same files |
| `pyproject.toml`   | Tool settings only; why `pythonpath` points to the function folder         |
| `python-tests.yml` | Same Python as the function's runtime                                      |

### `simulation/README.md`

Add a row for step 3: already local, no simulation needed (pytest, Ruff, the command line).

### Docs

Each document grows with the step.

| Document            | Adds                                                                          |
| ------------------- | ----------------------------------------------------------------------------- |
| `README.md`         | **Run it yourself:** Python set-up (`venv`, `pip install`), pytest, Ruff, the command line |
| `docs/decisions.md` | Python only unwraps and cleans, all joins are SQL; every step overwrites its output (idempotent) |

### Commits

1. `build: add pytest and Ruff settings and pre-commit hooks`
2. `feat(function): unwrap exports and check them against their own totals`
3. `feat(function): turn article HTML into plain text`
4. `feat(function): write NDJSON files and add a command-line entry`
5. `ops(ci): add Python tests workflow`
6. `docs: add the Python checks and decisions`

<br>

## Done when

- `pytest` passes.
- `ruff check` and `ruff format --check` pass.
- The command line writes 6 NDJSON files with 75, 6, 11, 5, 12 and 5 lines (`wc -l .build/ndjson/*/*.ndjson`).
- `pre-commit run --all-files` passes.
- The `Python tests` workflow is green on the pull request.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
