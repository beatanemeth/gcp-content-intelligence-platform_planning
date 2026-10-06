# Step 2: data transformation

Decision log for step 2 of [03_bea_answers_and_thoughts.md](03_bea_answers_and_thoughts.md). Step 1 decisions: [04_claude_step1_raw_data_upload.md](04_claude_step1_raw_data_upload.md).

**Status values:** _Agreed_ (confirmed by the maintainer).

<br>

## Summary

The Cloud Run function is the only place where Python runs. On a group's `_READY` marker (D1.9), it removes the envelope of each raw file, cleans the HTML in articles and writes one NDJSON file per raw file to `processed/` (D2.1, D2.2, D2.5). Then it starts one BigQuery load job per file (D2.3). Everything relational happens in BigQuery SQL: the lookups of categories, tags and authors, combining posts and articles, and the analysis (D2.2). This pattern is **ELT**.

The final shape, **cleaned**, is not stored as data. It is a set of BigQuery views, defined in Terraform, that combine the staging tables at query time (D2.4).

<br>

## Data flow

| Resting place           | Content                                                   | Arrives by                          |
| ----------------------- | --------------------------------------------------------- | ----------------------------------- |
| Laptop                  | Raw JSON files (source of truth)                          | Already there                       |
| Bucket `raw/`           | The same raw files, unchanged                             | `gcloud storage cp` (upload script) |
| Bucket `processed/`     | 6 NDJSON files: envelopes removed, HTML cleaned           | Cloud Run function (Python)         |
| BigQuery staging tables | 6 tables, one per file                                    | Load jobs started by the function   |
| BigQuery cleaned views  | `blog_posts`, `articles`: categories, tags and authors combined | SQL (D2.4)                    |

**Mapping to the original project:**

| Original project (laptop)                          | This project (cloud)          |
| -------------------------------------------------- | ----------------------------- |
| `data_raw/`                                        | Bucket `raw/`                 |
| ETL notebooks: envelope removal                    | Cloud Run function (Python)   |
| `data_temp/` (intermediate JSON)                   | Bucket `processed/` (NDJSON)  |
| ETL notebooks: combining categories, tags, authors | BigQuery SQL                  |
| `data_processed/blog_posts.json`, `articles.json`  | BigQuery cleaned views        |
| Data science notebooks                             | BigQuery SQL analysis         |

The final shape stays the same. What changes: steps run by hand become steps chained by events.

<br>

## Decisions

| ID   | Decision                                                                                             | Status |
| ---- | ---------------------------------------------------------------------------------------------------- | ------ |
| D2.1 | The function writes NDJSON files to `processed/`; BigQuery loads from those files.                   | Agreed |
| D2.2 | Python does only what SQL cannot do well (reshaping, HTML cleaning); SQL does all relational work.   | Agreed |
| D2.3 | The function starts the BigQuery load jobs itself, after writing a group's NDJSON files.             | Agreed |
| D2.4 | The cleaned layer is a set of BigQuery views defined in Terraform.                                   | Agreed |
| D2.5 | `processed/` mirrors `raw/`: same group folders, same base names, `.ndjson` extension, fixed names.  | Agreed |
| D2.6 | Raw files follow one naming pattern: `<group>_<entity>.json`, snake_case, plural, no `_data` suffix. | Agreed |

<br>

## Reasoning

### D2.1: NDJSON files, then a load job

- **Inspectable:** a wrong table can be traced to either the Python step or the load step.
- **Replayable:** BigQuery can be reloaded without re-running the Python.
- **Cheaper than streaming:** batch load jobs from Cloud Storage are free; streaming inserts (`insertAll`) are billed per MB.

### D2.2: Python and SQL split

| Task                                         | SQL suitable?                                  | Layer      |
| -------------------------------------------- | ---------------------------------------------- | ---------- |
| Unwrap envelopes into NDJSON                 | No: BigQuery cannot load the envelopes as rows | **Python** |
| Clean HTML in `articleText`                  | No: regular expressions break on nested markup | **Python** |
| Lookups (category, tag, member IDs to names) | Yes: plain joins                               | **SQL**    |
| Combine blogs and articles, analysis         | Yes: `UNION ALL`, `UNNEST`, window functions   | **SQL**    |

- **Per-file Python:** each raw file is cleaned on its own; the function holds no cross-file logic. It stays small and unit-testable on the dummy data.
- **Change from the original project:** the notebooks did the lookups in Python; here they are SQL joins, visible next to the data.
- **Effect on D1.9:** the marker stays, but its reason changes. Python no longer needs a group's files together; the SQL joins do. Without a complete group, a join silently returns empty names.

### D2.3: the function starts the load

| Option                                | How it works                                                     | Verdict  |
| ------------------------------------- | ---------------------------------------------------------------- | -------- |
| **Same function**                     | On `_READY`: write NDJSON, then one load job per file            | Chosen   |
| **Second trigger on `processed/`**    | Each NDJSON file fires an event that starts a loader             | Rejected |
| **External tables**                   | No load; BigQuery reads the files at query time                  | Rejected |

- **Why the same function:** it already knows the group is complete, so it loads the whole group in one run. Each load uses `WRITE_TRUNCATE`, which matches D1.10.
- **Why not a second trigger:** one event per file again; the grouping problem returns.
- **Why not external tables:** every query parses the files again; native tables are faster and cheaper to query.
- **Trade-off:** the function needs permission to write to BigQuery. A failed load re-runs the whole group, which is safe because every step overwrites.

### D2.4: cleaned layer as views

| Aspect             | View                                       | Table rebuilt after each load                  |
| ------------------ | ------------------------------------------ | ---------------------------------------------- |
| **Stored**         | Only the SQL                               | The result rows                                |
| **Freshness**      | Always current                             | Stale if a rebuild is missed                   |
| **Extra step**     | None                                       | Something must run the rebuild                 |
| **Query cost**     | The join runs on every query               | Joined once                                    |

- **Why views:** at 570 KB the join cost is negligible, and there is no rebuild step to forget. `apply` creates the whole cleaned layer.
- **When tables win:** large data, expensive logic, or a dashboard that queries constantly.
- **Consequence for step 3:** BigQuery checks a view's query when the view is created, so the staging tables must exist first. Terraform therefore creates the staging tables too.

### D2.5 and D2.6: file names

- **One name through all layers:** `raw/blogs/blog_posts.json`, then `processed/blogs/blog_posts.ndjson`, then the staging table `blog_posts`.
- **Fixed names, no timestamps:** a re-run overwrites the same files (D1.10); timestamped names would pile up.
- **Pattern:** snake_case, plural, a group prefix where the entity belongs to one group: `blog_posts`, `blog_categories`, `blog_tags`, `members`, `articles`, `article_categories`.
- **`members` without prefix:** Wix members are site-wide; they sit in the blog group because posts reference them as authors.
- **Field names unchanged:** only file names follow the pattern; fields stay identical to the Wix exports.
- **Dummy files:** this repository adds a `dummy_` prefix (`dummy_data_blogs/`, `dummy_data_articles/`); the implementation repository drops it.
