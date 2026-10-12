# Step 6: cleaned views

**Branch:** `feature/cleaned-views`

<br>

## In plain words

The first SQL layer: the staging tables, as delivered, become data that is easy to query.

- **3 views in `content_cleaned`:**
  - **`blog_posts`:** one row per post, with the author's nickname and the names of its categories and tags.
  - **`articles`:** one row per article, with its category name and its plain text.
  - **`content`:** posts and articles in one list, with the same columns for both.
- **What "cleaned" means here:** IDs replaced by names (joins), Wix's camelCase names turned into snake_case, only the useful columns kept. The data itself is not changed.
- **Views, not tables:** a view stores no data; it runs its SQL on every query. At about 130 records every query costs the minimum, and a view is always up to date after a new load.
- **Terraform creates the views** from SQL template files. The SQL stays readable as normal BigQuery SQL; only the project and dataset names are filled in.
- **The simulated check:** `terraform test` for the view resources; the BigQuery emulator creates the views from the same files and checks their rows against the staging tables.

<br>

## Technical details

### Files

| Path                                             | Content                                  |
| ------------------------------------------------ | ---------------------------------------- |
| `templates/cleaned/blog_posts.sql.tftpl`         | The `blog_posts` view                    |
| `templates/cleaned/articles.sql.tftpl`           | The `articles` view                      |
| `templates/cleaned/content.sql.tftpl`            | The `content` view                       |
| `simulation/check_views.py`                      | Creates and checks the views in the emulator |

**Placeholders, the only ones:** `${project}`, `${staging_dataset}`, `${cleaned_dataset}`. Table and view names are written in the SQL itself, such as `` `${project}.${staging_dataset}.blog_posts` ``.

**Editor tip:** link `*.sql.tftpl` to SQL in VS Code (`files.associations`), so the files keep SQL colouring.

### The views and their columns

**`blog_posts`** (from `blog_posts`, `members`, `blog_categories`, `blog_tags`):

| Column                                       | Source                                    |
| -------------------------------------------- | ----------------------------------------- |
| `id`, `title`, `slug`, `excerpt`             | `_id`, `title`, `slug`, `excerpt`         |
| `first_published_date`, `last_published_date` | `firstPublishedDate`, `lastPublishedDate` |
| `minutes_to_read`                            | `minutesToRead`                           |
| `author_nickname`                            | `members.profile.nickname`, by `memberId` |
| `views`, `likes`                             | `metrics.views`, `metrics.likes`          |
| `categories`                                 | List of `id` and `label`, from `categoryIds` |
| `tags`                                       | List of `id` and `label`, from `tagIds`   |

- **Categories and tags stay nested lists** (`ARRAY` of `STRUCT`), one row per post. BigQuery's native model: no bridge tables.
- **One `ARRAY(SELECT AS STRUCT ... FROM UNNEST(...) JOIN ...)` subquery per list,** not a join in the main `FROM`:
  - A post without tags keeps its row, with an empty list.
  - No `GROUP BY` is needed to rebuild one row per post.
  - Two lists joined in one `FROM` would multiply the rows (posts × categories × tags).
- **The category name comes from `label`, not `title`:** the full dataset has categories with an empty `title` but a filled `label`. The dummy data hides this; every dummy category has both.
- **`LEFT JOIN members`:** a post with an unknown author keeps its row, with no nickname.

**`articles`** (from `articles`, `article_categories`):

| Column           | Source                                       |
| ---------------- | -------------------------------------------- |
| `id`, `title`, `lead` | `_id`, `title`, `lead`                  |
| `created_date`   | `_createdDate`                               |
| `category_title` | `article_categories.title`, by `category`    |
| `keywords`       | `keywords` (a list)                          |
| `article_text`   | `articleText`, already plain text (step 3)   |

- **`LEFT JOIN article_categories`:** an article without a category keeps its row.
- **Article categories keep `title`:** it is filled in every export.

**`content`** (from the 2 views above, not from staging):

| Column         | Blog post                     | Article                                     |
| -------------- | ----------------------------- | ------------------------------------------- |
| `content_type` | `'blog_post'`                 | `'article'`                                 |
| `content_id`   | `id`                          | `id`                                        |
| `title`        | `title`                       | `title`                                     |
| `published_at` | `first_published_date`        | `created_date`: articles have no publish date |
| `categories`   | The category labels           | A list of one, or empty if no category      |
| `tags`         | The tag labels                | `keywords`                                  |

- **`UNION ALL`,** not `UNION`: no duplicate check is needed, and it is cheaper.
- **An empty list, not `[NULL]`:** BigQuery rejects a `NULL` inside an array. `IF(category_title IS NULL, ARRAY<STRING>[], [category_title])`.

### `bigquery.tf`

**Placeholders local** (in `bigquery.tf`, next to its only user): `sql_placeholders`, a map of `project`, `staging_dataset` and `cleaned_dataset`, each from the variable or the dataset resource.

**Module `main.tf`:** add `cleaned_views`, the set `blog_posts`, `articles`. `content` is not in it; it has its own block.

| Resource                             | Notes                                                       |
| ------------------------------------ | ----------------------------------------------------------- |
| `google_bigquery_table.cleaned`      | `for_each = local.cleaned_views`; in `content_cleaned`      |
| `google_bigquery_table.content`      | One block; in `content_cleaned`                             |

**Each view block:**

| Argument              | Value                                                                       | Why                                              |
| --------------------- | --------------------------------------------------------------------------- | ------------------------------------------------ |
| `view.query`          | `templatefile("${path.module}/templates/cleaned/${each.key}.sql.tftpl", local.sql_placeholders)` | The SQL file with its names filled in |
| `view.use_legacy_sql` | `false`                                                                     | The API's default is legacy SQL                  |
| `deletion_protection` | `false`                                                                     | A view is a `google_bigquery_table`; same lock as the tables |
| `labels`              | `local.labels`                                                              |                                                  |
| `depends_on`          | `cleaned`: the staging tables. `content`: the `cleaned` block               | See below                                        |

**Why `depends_on`:** BigQuery checks a view's SQL when it creates the view, so every table the SQL reads must already exist. Terraform cannot see table names inside a text string; it only sees references in HCL. `depends_on` tells it the order: staging tables, then `blog_posts` and `articles`, then `content`.

**Why `content` has its own block:** inside one `for_each` block, Terraform creates all items in parallel. `content` reads the other two views, so it must wait for them.

### Simulated check: `simulation/check_views.py`

Run after `check_staging.py`, with the emulator still running: it keeps the loaded tables in memory.

1. **Creates `content_cleaned`** if it is missing: get first, create only on `NotFound` (the emulator's `500` quirk from step 4).
2. **Renders each template** with Python's `string.Template(...).substitute(...)`: the same `${name}` syntax as `templatefile()`, and it also fails on a missing placeholder. The values match `bigquery.tf`.
3. **Creates the views in Terraform's order:** `blog_posts`, `articles`, then `content`. Each view is deleted and created again on every run, so a changed SQL file takes effect.
4. **Checks each view against staging,** not against fixed numbers, so the same script works for the full dataset in step 8:

| Check                  | View                     | Expected                                       |
| ---------------------- | ------------------------ | ---------------------------------------------- |
| `row_count`            | `blog_posts`, `articles` | The staging table's row count                  |
| `distinct_ids`         | `blog_posts`, `articles` | The staging table's row count: no ID twice     |
| `row_count`            | `content`                | Posts + articles                               |
| `empty_names`          | `content`                | 0 empty category or tag names                  |
| `missing_article_text` | `articles`               | 0                                              |

5. Exits with 1 on any mismatch.

**Emulator support, probed in the first build:** views created through the API, `ARRAY(SELECT AS STRUCT ...)` over `UNNEST`, and `UNION ALL` of arrays all run.

### Simulated test: `tests/bigquery.tftest.hcl`

Add `run "cleaned_views"`, `command = apply`. Assertions:

- The 3 views exist, all in `content_cleaned`.
- Each view's `query` equals the rendered template (`templatefile(...)` with the expected values).
- `use_legacy_sql = false`, `deletion_protection = false`, the labels.

### CI

No change: `terraform-checks.yml` covers `templates/` (under `terraform/**`); `python-tests.yml` runs Ruff on `check_views.py` (under `simulation/**`, step 4).

### Notes in the files

| File                  | Note                                                                        |
| --------------------- | --------------------------------------------------------------------------- |
| Each `.sql.tftpl`     | Header: what one row is; Terraform fills in the placeholders. `blog_posts`: why `ARRAY` subqueries; why `label`. Each `LEFT JOIN`: which row it keeps. `content`: the article stand-ins; why an empty list |
| `bigquery.tf`         | The only placeholders; the API's default is legacy SQL; a view has the same lock as a table; why `depends_on`; why `content` has its own block |
| `check_views.py`      | Simulated, not real GCP; run after `check_staging.py`; `substitute` fails like `templatefile()`; expected values come from staging |

### `simulation/README.md`

Add the step 6 row and the views section: order, same SQL as Terraform, expected values from staging.

### Docs

Each document grows with the step.

| Document            | Adds                                                                         |
| ------------------- | ---------------------------------------------------------------------------- |
| `README.md`         | **Run it yourself:** `check_views.py`, after `check_staging.py`              |
| Module `README.md`  | The 3 cleaned views                                                          |
| `docs/decisions.md` | Views, not tables; categories and tags as nested lists; SQL in templates with only project and dataset placeholders, and `depends_on` per layer |

### Commits

1. `feat(sql): add cleaned blog_posts and articles views`
2. `feat(sql): add cleaned content view`
3. `feat(terraform): create the cleaned views from their templates`
4. `test(terraform): add simulated cleaned view test`
5. `test: add simulated view check with the BigQuery emulator`
6. `docs: add the view check and view decisions`

<br>

## Done when

- `terraform test`: 5 runs pass.
- `check_staging.py`, then `check_views.py`: every check `ok`, exit code 0.
- **Proof that the checks can fail:** join `tagIds` in the main `FROM` instead of the subquery (flattened tags); `row_count` for `blog_posts` is much higher than 75. Undo the change.
- **Known gap:** no dummy post lacks tags, so "a post without tags keeps its row" waits for step 8.
- `pre-commit run --all-files` and the workflows are green.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
