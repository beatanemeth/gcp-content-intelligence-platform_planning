# Step 7: analysis views

**Branch:** `feature/analysis-views`

<br>

## In plain words

The last layer: 4 views in `content_analysis`, each answering one question about the content.

- **One view, one question.** Each SQL file opens with its question and the exact definition of its metric. A number without a definition cannot be checked.
- **They read only the cleaned views,** never staging. The joins and renames are done once, in step 6.
- **Still views, no new tool:** 7 views that store nothing need no Dataform or dbt. Terraform already creates them. Dataform would pay off once tables are rebuilt on a schedule, loaded incrementally or tested with its own assertions.
- **The simulated check:** `terraform test` for the view resources; the emulator creates the 4 views and checks that each returns rows; one number is checked by hand against the dummy data.

<br>

## Technical details

### The 4 views

| View                     | Question                                                                                                                            | Reads        | Technique           |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------------- |
| `content_per_month`      | How much content was published in each calendar month (for example, January 2024), per content type, and what is the running total? | `content`    | Running total       |
| `tag_frequency`          | Which tags and keywords are used most, per content type?                                                                            | `content`    | `CROSS JOIN UNNEST` |
| `top_posts_per_category` | Which 3 posts have the most views in each category, and who wrote them?                                                             | `blog_posts` | Ranking             |
| `article_text_stats`     | How long are the articles on average, per category?                                                                                 | `articles`   | Text measure        |

**One technique per view,** so each view shows something the others do not.

### Columns, metrics and SQL technique

**`content_per_month`:** `publish_month`, `content_type`, `items_published`, `running_total`.

- **Month:** `DATE_TRUNC(DATE(published_at), MONTH)`, in UTC. `publish_month` is a `DATE`, the first day of the month; each year's months are separate rows. Months without content have no row.
- **Running total:** `SUM(items_published) OVER (PARTITION BY content_type ORDER BY publish_month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`.
- **Technique:** a `WITH` step groups by month; a **window function** then adds the total. A window keeps one row per month, where `GROUP BY` would collapse them.

**`tag_frequency`:** `content_type`, `tag`, `uses`.

- **`uses`:** items that carry the tag (`COUNT(DISTINCT content_id)`). Blog posts carry tags; articles carry keywords; both sit in `content.tags`.
- **Technique:** `CROSS JOIN UNNEST(tags)`, then `GROUP BY content_type, tag`. An item without tags has nothing to count, so dropping it is right.
- **"Used most":** the reader sorts by `uses`. A view keeps no row order, so an `ORDER BY` inside it would promise nothing.
- **Tags as written:** "Massa" and "massa" are two tags.

**`top_posts_per_category`:** `category_label`, `rank_in_category`, `title`, `author_nickname`, `first_published_date`, `views`.

- **Technique:** `CROSS JOIN UNNEST(categories)` gives one row per post and category; `ROW_NUMBER() OVER (PARTITION BY category ORDER BY views DESC, first_published_date, id)` ranks them; the outer query keeps `rank_in_category <= 3`.
- **Tie-breaker:** on equal views the earlier post ranks higher, then the ID. The top 3 stays the same on every run.
- **`ROW_NUMBER`, not `RANK`:** exactly 3 rows per category, even with ties.
- **A post in two categories can rank in both.** Said in the header.
- **Author:** `author_nickname` comes from the step 6 `members` join; it is `NULL` for an unknown author.

**`article_text_stats`:** `category_title`, `articles`, `avg_word_count`, `avg_reading_minutes`.

- **Word count:** `ARRAY_LENGTH(SPLIT(article_text, ' '))`. Exact, because step 3 joins the words with single spaces.
- **Reading time:** word count / 200, rounded up (200 words per minute).
- **Technique:** a `WITH` step measures each article; the outer query averages per category with `GROUP BY`, rounded to 1 decimal.
- **Articles without a category** share one row with a `NULL` `category_title`.

### Header of each SQL file

Every `.sql.tftpl` in `templates/analysis/` opens with:

1. **Question:** one sentence.
2. **Metric:** what each number means, its unit, and its edge cases (ties, a post in two categories, `NULL` authors).
3. **Placeholders:** Terraform fills in the project and the cleaned dataset.

### `bigquery.tf`

**Module `main.tf`:** add `analysis_views`, the set of the 4 view names.

`google_bigquery_table.analysis`, with `for_each = local.analysis_views`: in `content_analysis`; `templatefile("${path.module}/templates/analysis/${each.key}.sql.tftpl", local.sql_placeholders)`; `use_legacy_sql = false`; `deletion_protection = false`; the labels.

- **`depends_on`:** both cleaned blocks, `google_bigquery_table.cleaned` and `google_bigquery_table.content`. The analysis views read both.
- **The same placeholders map** as step 6; the analysis files use only `project` and `cleaned_dataset`.
- **4 views in one block:** none reads another, so creating them in parallel is fine.

### Simulated check: `simulation/check_views.py`

Extend the step 6 script:

1. Create `content_analysis` if it is missing (get first, then create).
2. Create the 4 analysis views **after** the 3 cleaned views: Terraform's order.
3. **New check, `has_rows`:** every analysis view returns at least 1 row.
4. Both checks always run, so one report shows every failure; exit with 1 on any.
5. **`--results <folder>` flag:** after the checks, writes each analysis view to `<folder>/<view>.csv`, with a header row. Used in step 8 for the results in the README. The views exist only while the emulator runs, and this script already connects to it.

**Emulator support, probed in the first build:** `SUM() OVER` and `ROW_NUMBER()` both run.

**Hand check, once:** pick one blog post tag in `tag_frequency` and note its `uses`. Count how often that tag's `_id` appears in `data/dummy/blogs/blog_posts.json` (`grep -o`). The numbers must match.

### Simulated test: `tests/bigquery.tftest.hcl`

Add `run "analysis_views"`, `command = apply`. Assertions:

- The 4 views exist, all in `content_analysis`.
- Each view's `query` equals its rendered template.
- `use_legacy_sql = false`, `deletion_protection = false`, the labels.

### Notes in the files

| File            | Note                                                                         |
| --------------- | ---------------------------------------------------------------------------- |
| Each `.sql.tftpl` | The 3-part header; one comment on each window ("keeps one row per ...") and each `CROSS JOIN UNNEST` (why rows without a list may drop) |
| `bigquery.tf`   | The analysis views read both cleaned blocks, so they wait for both           |
| `check_views.py` | `has_rows` is the analysis criterion; both checks run before the exit; `--results` output is simulated |

### `simulation/README.md`

Add the step 7 row and the hand check.

### Docs

Each document grows with the step.

| Document            | Adds                                                                 |
| ------------------- | -------------------------------------------------------------------- |
| `README.md`         | **The 4 questions:** one row per analysis view, with its question    |
| Module `README.md`  | The 4 analysis views                                                 |
| `docs/decisions.md` | One question and one SQL technique per view, with its metric in the file header; no Dataform or dbt |

### Commits

1. `feat(sql): add the 4 analysis views, each with its question and metric`
2. `feat(terraform): create the analysis views from their templates`
3. `test(terraform): add simulated analysis view test`
4. `test: check that every analysis view returns rows`
5. `test: save the analysis views as CSV with --results`
6. `docs: add the analysis questions and decisions`

<br>

## Done when

- `terraform test`: 6 runs pass.
- `check_staging.py`, then `check_views.py`: every check `ok`, exit code 0.
- `check_views.py --results .build/results-simulated` writes 4 CSV files; `git status` shows none of them.
- The hand check matches.
- **Proof that the check can fail:** add `WHERE FALSE` to `top_posts_per_category`; `has_rows` fails. Undo the change.
- `pre-commit run --all-files` and the workflows are green.
- `progress.md` records this step: what changed, how it was checked, any deviation from the plan, the next step.
