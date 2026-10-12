# Step 3: BigQuery

Decision log for step 3 of [03_bea_answers_and_thoughts.md](03_bea_answers_and_thoughts.md). Earlier steps: [04_claude_step1_raw_data_upload.md](04_claude_step1_raw_data_upload.md), [05_claude_step2_data_transformation.md](05_claude_step2_data_transformation.md).

**Status values:** _Agreed_ (confirmed by the maintainer).

<br>

## Summary

BigQuery has three datasets, one per layer: `staging`, `cleaned` and `analysis` (D3.1, D3.13). All of them sit in `europe-west1`, the same region as the bucket (D3.2, D3.12), and carry labels for cost tracking (D3.8).

Staging holds 6 tables, one per processed file, created by Terraform from JSON schema files (D3.4). They are exact copies of the source: Wix field names, nested records and arrays are kept (D3.3). Loading is strict: an unexpected field or value fails the load (D3.9), so the unused empty arrays stay in the schema (D3.10).

The cleaned layer is a set of views. They rename columns to snake_case (D3.6) and carry categories and tags as nested lists inside each post (D3.5). A unified content view combines blog posts and articles (D3.11). The analysis layer is also a set of views, one per question (D3.13). All view SQL lives in `.sql` files (D3.14). There is no partitioning or clustering (D3.7).

<br>

## Source structure

| Entity               | Join key to other entities                   | Nested fields                                                 |
| -------------------- | -------------------------------------------- | ------------------------------------------------------------- |
| `blog_posts`         | `categoryIds`, `tagIds` (arrays), `memberId` | `postCountInfo`, `metrics`, `media` (records); several arrays |
| `blog_categories`    | `_id`                                        | `translations` (always empty)                                 |
| `blog_tags`          | `_id`                                        | None                                                          |
| `members`            | `_id`                                        | `profile` (record)                                            |
| `articles`           | `category` (single value)                    | `keywords` (array)                                            |
| `article_categories` | `_id`                                        | None                                                          |

- **Many-to-many:** a blog post can have several categories (`categoryIds` is an array).
- **All keys resolve:** every category, tag and member ID in the posts exists in its lookup file.
- **Always-empty arrays:** several post fields (`relatedPostIds`, `pricingPlanIds`, `translations` and others) are empty in every record.
- **Dates:** ISO 8601 strings, loadable as `TIMESTAMP`.

<br>

## Decisions

| ID    | Decision                                                                                            | Status |
| ----- | --------------------------------------------------------------------------------------------------- | ------ |
| D3.1  | One dataset per layer: `staging`, `cleaned` (and `analysis`, D3.13).                                | Agreed |
| D3.2  | The datasets and the data bucket share one region.                                                  | Agreed |
| D3.3  | Staging tables are 1:1 copies of the processed files, with Wix field names and nesting kept.        | Agreed |
| D3.4  | Schemas live in JSON files, referenced from Terraform.                                              | Agreed |
| D3.5  | Cleaned views use nested and repeated fields (arrays of structs), not bridge tables.                | Agreed |
| D3.6  | Cleaned views rename columns to snake_case.                                                         | Agreed |
| D3.7  | No partitioning or clustering.                                                                      | Agreed |
| D3.8  | Every dataset carries Terraform labels (for example `project`, `layer`).                            | Agreed |
| D3.9  | Strict staging: typed columns; any unexpected value or field fails the load.                        | Agreed |
| D3.10 | Unused, always-empty arrays stay in the staging schema as `REPEATED STRING` and are never queried.  | Agreed |
| D3.11 | Unified content view: common columns, with `categories` and `tags` as string arrays for both types. | Agreed |
| D3.12 | Region `europe-west1` (Belgium) for all regional resources.                                         | Agreed |
| D3.13 | Analysis queries are views in a third dataset, `analysis`, created by Terraform.                    | Agreed |
| D3.14 | All view SQL lives in `.sql` files; Terraform reads them with `templatefile()`.                     | Agreed |

<br>

## Reasoning

### D3.1, D3.2, D3.8, D3.12: datasets, region, labels

- **Datasets, not table prefixes:** a dataset is the unit of location, access control and `delete_contents_on_destroy`. The dataset name carries the layer, so table names stay identical across layers (`staging.blog_posts`, `cleaned.blog_posts`).
- **One region:** a load job needs compatible bucket and dataset locations; the same region is the simplest rule. A dataset's location cannot change after creation. One `region` variable feeds the bucket, the datasets and the function.
- **`europe-west1`:** an EU region with every service this project needs, lower prices than most European regions and a low-CO2 grid. A multi-region (`EU`) would complicate the trigger, which must sit in the bucket's location.
- **Labels:** they appear in billing data, so cost can be filtered by project or layer.

### D3.3, D3.4, D3.9, D3.10: staging

- **Faithful copy:** camelCase Wix names stay in staging; renaming happens in the cleaned views. Arrays load as `REPEATED` columns, records as `RECORD` (`STRUCT`) columns.
- **Ownership split:** Terraform owns the table and its schema; the pipeline owns the rows. Replacing rows does not show up as Terraform drift.
- **JSON schema files:** a nested schema in HCL is long and hard to read. A JSON schema file (the format `bq` uses) is referenced with `file()` and documents the table for readers.
- **Strict loading:** unknown fields are not ignored, and no bad records are tolerated. A failed load is visible at once; a lenient load hides problems until a query returns wrong numbers.
- **Unused arrays:** strict loading rejects fields missing from the schema, so the always-empty arrays are declared as `REPEATED STRING`. An empty array fits any element type. The cleaned views do not select them.
- **Rejected alternatives:** removing the fields in Python (extra code; staging is no longer a faithful copy) and lenient loading (also hides real problems such as typos or new fields).

### D3.5, D3.6, D3.11: cleaned views

- **Nested and repeated fields:** BigQuery favours denormalised tables over many joined tables. A post carries its categories and tags as arrays of structs; `UNNEST` flattens them when needed.
- **Rejected alternative:** a `post_tags` bridge table (classic relational design). It adds a join to every query.
- **Empty arrays in SQL:** `CROSS JOIN UNNEST(tags)` drops posts without tags. Where every post must stay, use `LEFT JOIN UNNEST(tags)`. The full dataset has 2 such posts; the dummy data has none.
- **snake_case:** the cleaned views are project-defined, so project naming applies (`firstPublishedDate` becomes `first_published_date`).

**Unified content view (D3.11):**

| Column         | Blog posts                               | Articles                                |
| -------------- | ---------------------------------------- | --------------------------------------- |
| `content_type` | `'blog_post'`                            | `'article'`                             |
| `content_id`   | `_id`                                    | `_id`                                   |
| `title`        | `title`                                  | `title`                                 |
| `published_at` | `firstPublishedDate`                     | `_createdDate` (no publish date exists) |
| `categories`   | Category titles (array, often 2 or more) | Category title (array of one)           |
| `tags`         | Tag labels (array)                       | `keywords` (array)                      |

- **Arrays for both types:** `UNNEST(categories)` and `UNNEST(tags)` then work the same way for both.
- **Type-specific columns** (blog `metrics`, article `lead`) stay in the type-specific views.

### D3.7: no partitioning or clustering

- **Why not:** on-demand queries bill a minimum of 10 MB per table; at 570 KB, every query costs the minimum anyway. Views cannot be partitioned either.
- **When it matters:** large tables queried by a date range (partition by date) or by a frequent filter column (cluster).

### D3.13, D3.14: analysis views and SQL files

| Option                             | How it works                                    | Verdict  |
| ---------------------------------- | ----------------------------------------------- | -------- |
| **`.sql` files run by hand**       | Copied into the BigQuery console when needed    | Rejected |
| **Views in an `analysis` dataset** | Created by `terraform apply`; queried like tables | Chosen |

- **Always there:** after `apply` and an upload, the results exist without a manual step (D1.3). Looker Studio, if built, connects straight to these views.
- **No parameters:** a view cannot take "a given month"; that becomes a `WHERE` filter or a view grouped by month.
- **`.sql` files with `templatefile()`:** each query is a normal SQL file, easy to review on GitHub. Placeholders such as `${project}` and `${dataset}` keep the project ID out of the SQL.
- **Dependencies:** values passed into the template from other resources give Terraform implicit dependencies. Table names written as plain text inside the SQL do not.

**Planned analysis views** (from the SQL ideas in [03_bea_answers_and_thoughts.md](03_bea_answers_and_thoughts.md#step-3-transformed-data-is-stored-in-the-cloud)):

| View                   | Technique                                   |
| ---------------------- | ------------------------------------------- |
| Content per month      | `UNION ALL` content model, `GROUP BY` month |
| Tag frequency          | `UNNEST(tags)`, by month and content type   |
| Engagement by category | `UNNEST(categories)`, blog `metrics`        |
| Running total          | Window function (`SUM() OVER`)              |
| Top posts per category | Window function (`ROW_NUMBER() OVER`)       |
