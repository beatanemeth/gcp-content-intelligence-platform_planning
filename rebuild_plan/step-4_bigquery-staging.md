# Step 4: BigQuery datasets, staging tables, data checks

**Branch:** `feature/bigquery-staging`

<br>

## In plain words

The place where the cleaned data lands, and the checks that prove it arrived intact.

- **3 datasets, one per layer** (the medallion pattern):
  - **`content_staging`:** the data as delivered, 6 tables.
  - **`content_cleaned`:** prepared data, as views (step 6).
  - **`content_analysis`:** answers, as views (step 7).
- **6 staging tables,** one per NDJSON file, each with a **strict schema** in a JSON file. A wrong type or an unknown field fails the load loudly, instead of loading bad data quietly.
- **Data checks** (`data_checks.sql`): BigQuery checks each row's shape, but not the relations between rows. One query checks row counts, duplicate IDs, broken references and missing titles.
- **The simulated check:**
  - `terraform test` checks the datasets and tables.
  - A **BigQuery emulator** in Docker loads the NDJSON files from step 3 and runs the data checks.
  - A pytest test covers the 2 rules the emulator does not enforce.

<br>

## Technical details

### Locals (module `main.tf`)

| Local            | Value                                                                             |
| ---------------- | --------------------------------------------------------------------------------- |
| `datasets`       | Map: `staging`, `cleaned`, `analysis`, each with its description                  |
| `staging_tables` | Set of the 6 table names, the same as the NDJSON file names                       |

### `bigquery.tf`

**Datasets:** `google_bigquery_dataset.layer`, with `for_each = local.datasets`.

| Argument                     | Value                                         | Why                                                       |
| ---------------------------- | --------------------------------------------- | --------------------------------------------------------- |
| `dataset_id`                 | `"${local.feature}_${each.key}"`              | `content_staging`, `content_cleaned`, `content_analysis`  |
| `location`                   | `var.region`                                  | A load job reads only from a bucket in the same location  |
| `delete_contents_on_destroy` | `true`                                        | `destroy` removes the dataset with its tables             |
| `labels`                     | `merge(local.labels, { layer = each.key })`   | The bill shows the cost per layer                         |

**Staging tables:** `google_bigquery_table.staging`, with `for_each = local.staging_tables`.

| Argument              | Value                                                   | Why                                                   |
| --------------------- | ------------------------------------------------------- | ----------------------------------------------------- |
| `dataset_id`          | `google_bigquery_dataset.layer["staging"].dataset_id`   | The reference also tells Terraform to create the dataset first |
| `table_id`            | `each.key`                                              | Same name as the NDJSON file                          |
| `schema`              | `file("${path.module}/files/schemas/${each.key}.json")` | One strict schema per table                           |
| `deletion_protection` | `false`                                                 | A Terraform-only lock, on by default; off, so `destroy` can remove the table |
| `labels`              | `local.labels`                                          |                                                       |

- **`for_each`, not `count`:** each dataset and table is addressed by name (`layer["staging"]`), not by position. Removing one item never renames the others.
- **No partitioning or clustering:** about 130 records; every query already costs the minimum.

**Output:** `dataset_ids`, a map of layer to dataset ID, in the module and passed on in `environments/dev/outputs.tf`.

### Schemas: `files/schemas/<table>.json`

| Rule                                         | Why                                                        |
| -------------------------------------------- | ---------------------------------------------------------- |
| **Wix field names, unchanged**               | Staging holds the data as delivered; renaming is the cleaned layer's job |
| **An explicit `mode` on every column**       | Nothing left to a default                                  |
| **`REQUIRED` only for `_id`**                | Every record has an ID; any other field may be missing     |
| **Arrays as `REPEATED`**                     | `categoryIds`, `tagIds`, `keywords`: lists stay lists      |
| **Always-empty arrays as `REPEATED STRING`** | Such as `translations`, `relatedPostIds`; a type is needed even if no value exists |
| **Nested objects as `RECORD`**               | Such as `metrics` (views, likes, comments) in `blog_posts`, `profile` in `members` |
| **Dates as `TIMESTAMP`**                     | Wix writes ISO 8601 in UTC (`2025-11-18T18:39:37.830Z`)    |

**Build each schema from the dummy data:** list every field of the NDJSON records, then choose its type. The same schemas must fit the full dataset in step 8.

### Data checks: `helpers/data_checks.sql`

One query, one row per check: `check_name`, `table_name`, `result`.

| Check               | Question                                                             | Expected                  |
| ------------------- | -------------------------------------------------------------------- | ------------------------- |
| `row_count`         | Did every record arrive?                                             | The record counts (dummy: 75, 6, 11, 5, 12, 5) |
| `duplicate_keys`    | Does any `_id` appear twice? (`COUNT(*) - COUNT(DISTINCT _id)`)      | 0                         |
| `orphan_references` | Does every category, tag, author and article category ID exist in its lookup table? | 0          |
| `missing_titles`    | Does every post and article have a title?                            | 0                         |

- **Orphans with `LEFT JOIN ... WHERE right._id IS NULL`,** not `NOT IN`: the intent stays clear, and a `NULL` cannot hide a miss.
- **Arrays with `CROSS JOIN UNNEST`:** each ID in `categoryIds` and `tagIds` is checked on its own.
- **Real GCP later:** `bq query --use_legacy_sql=false < helpers/data_checks.sql`.

### Simulated check: the BigQuery emulator

**Tool:** `goccy/bigquery-emulator` `0.8.1`, in Docker. It speaks the BigQuery API on `localhost:9050`.

```bash
docker run --rm -p 9050:9050 ghcr.io/goccy/bigquery-emulator:0.8.1 \
    --project=content-demo-sim --dataset=content_staging
python simulation/check_staging.py      # second terminal
```

**`simulation/check_staging.py`:**

1. Connects the BigQuery client to the emulator: anonymous credentials, `api_endpoint` set to the emulator.
2. Loads the 6 NDJSON files from `.build/ndjson/`, as a real load would: the table's schema file, `NEWLINE_DELIMITED_JSON`, `WRITE_TRUNCATE`, strict defaults.
3. Runs `data_checks.sql` and prints each result next to its expected value.
4. Exits with 1 on any mismatch. `--counts full` switches to the full dataset's counts (step 8).

**Emulator quirks found in the first build:**

| Quirk                                                   | Handling                                                              |
| ------------------------------------------------------- | --------------------------------------------------------------------- |
| Creating an existing dataset answers `500`, not `409`; the client retries for 10 minutes | The emulator creates `content_staging` at start (`--dataset`); the script never creates it |
| A repeated `--dataset` flag keeps only the last value   | Only `content_staging` at start; step 6 creates the other 2 datasets in its script |
| A stopped container looks like a slow query, not an error | When a script hangs, check `docker ps` first                        |

**What the emulator does and does not enforce:**

| Bad input                                        | Real BigQuery | Emulator           | Covered by                       |
| ------------------------------------------------ | ------------- | ------------------ | -------------------------------- |
| Wrong type, bad `TIMESTAMP`, single value in a `REPEATED` column | Load fails | Load fails | The emulator              |
| One bad row among good rows                      | Whole load fails | Same            | The emulator                     |
| Unknown field                                    | Load fails    | **Loads; dropped** | `tests/function/test_schemas.py` |
| Missing `REQUIRED` `_id`                         | Load fails    | **Loads**          | `tests/function/test_schemas.py` |

### pytest: `tests/function/test_schemas.py`

- **For each of the 6 files:** `transform` the dummy export, then compare every record with its schema file, nested `RECORD` fields included.
- **Fails on:** a field the schema does not know, and a `REQUIRED` field that is missing.
- **A second test** feeds a bad record to the checker, so the checker itself is proven to find problems.
- **Runs offline in CI,** so it also guards later changes to the function or the schemas.

### Simulated test: `tests/bigquery.tftest.hcl`

`run "datasets_and_staging_tables"`, `command = apply`, the same mocked provider. Assertions:

- The 3 dataset names; project and location; `delete_contents_on_destroy`; the 3 labels.
- 6 staging tables, all in `content_staging`, each with its own schema file, `deletion_protection = false`, the labels.
- The `dataset_ids` output maps each layer to its dataset.

### Dependencies and CI

- **`requirements-dev.txt`:** add `google-cloud-bigquery` (pinned) for `check_staging.py`. Step 5b moves it into the function's `requirements.txt`.
- **`python-tests.yml`:** add 2 paths:
  - `terraform/modules/content_pipeline/files/schemas/**`, so a schema change runs `test_schemas.py`.
  - `simulation/**`, so a change to the simulation scripts runs Ruff on them. pytest does not test them: they need the emulator.
- **The emulator never runs in CI:** it needs Docker and a long-running container. It is a laptop check.

### Notes in the files

| File                  | Note                                                                     |
| --------------------- | ------------------------------------------------------------------------ |
| `main.tf`             | `datasets`: one per layer, the key becomes the name and the label. `staging_tables`: one per NDJSON file |
| `bigquery.tf`         | Why `for_each`; why the dataset's location; `delete_contents_on_destroy`; `deletion_protection` is a Terraform-only lock |
| `data_checks.sql`     | How to run it; the expected values; why `LEFT JOIN`, not `NOT IN`        |
| `check_staging.py`    | Simulated, not real GCP; how to start the emulator; the 2 rules it misses |
| `bigquery.tftest.hcl` | The mocked provider replaces `apply` in real GCP                         |

### `simulation/README.md`

Add: the step 4 row, the emulator section (version, start command, the quirks), and the table of what it enforces.

### Docs

Each document grows with the step.

| Document            | Adds                                                                                    |
| ------------------- | --------------------------------------------------------------------------------------- |
| `README.md`         | **Run it yourself:** Docker, the emulator start command, `check_staging.py`, the table of data checks |
| Module `README.md`  | The 3 datasets and 6 staging tables; the `dataset_ids` output                           |
| `docs/decisions.md` | 3 layers (staging, cleaned, analysis); strict schemas with the Wix names; no partitioning; the simulation: mocked provider and emulator, what they prove and what they cannot |

### Commits

1. `feat(terraform): add staging table schemas`
2. `test(function): check cleaned records against the staging schemas`
3. `feat(terraform): add BigQuery datasets and staging tables`
4. `feat(terraform): add dataset_ids output`
5. `test(terraform): add simulated BigQuery test with mocked provider`
6. `feat(sql): add staging data quality checks`
7. `test: add simulated staging check with the BigQuery emulator`
8. `ops(ci): run Python tests when a staging schema changes`
9. `docs: add the staging checks and BigQuery decisions`

<br>

## Done when

- `terraform test`: 2 runs pass (buckets, datasets and tables).
- `pytest` passes, including `test_schemas.py`.
- `check_staging.py` prints `ok` for every check and exits with 0.
- **Proof that the checks can fail:** add a duplicate line to one NDJSON file; `duplicate_keys` shows 1 and the script exits with 1. Undo the change.
- `pre-commit run --all-files` and both workflows are green.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
