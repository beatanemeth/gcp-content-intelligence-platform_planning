# Step 8: full dataset run

**Branch:** `docs/full-dataset-run`, for the records and the docs only; no code changes. If a check fails, fix it on its own branch first, then run again.

<br>

## In plain words

- **Why:** dummy data can hide cases that real data has. This run is the only test of the whole local chain on the real exports.
- **No code:** the exports go in the git-ignored `data/raw/`; the same commands from steps 3 to 7 run on them.
- **Publishing rule:** only analysis results, labelled as simulated, with author names replaced ("Author 1", "Author 2"). Never the records.

<br>

## Checklist

1. **Copy** the 6 exports into `data/raw/blogs/` and `data/raw/articles/`; `git status` shows nothing under `data/raw/`.
2. **Clean:** `python terraform/modules/content_pipeline/files/function/main.py data/raw .build/ndjson`.
   - Expected: 6 `INFO` lines, and 1 `ERROR` for `members` (8 records, stated total 9).
3. **Count:** `wc -l .build/ndjson/*/*.ndjson`.
   - Expected: 113 posts, 9 categories, 17 tags, 8 members, 18 articles, 8 article categories.
4. **Fresh emulator:** stop the old container and start a new one, so no dummy rows remain.
5. **Staging:** `python simulation/check_staging.py --counts full`.
   - Expected: every check `ok`.
6. **Views:** `python simulation/check_views.py --results .build/results-simulated`.
   - Expected: every check `ok`; 4 CSV files.
7. **Confirm the hidden cases:**
   - 2 posts without tags in `data/raw/blogs/blog_posts.json`; `blog_posts` still has 113 rows.
   - `empty_names` is 0: the category names come from `label`.
8. **Record** the date and each step's result in `simulation/README.md` and `progress.md`.
9. **Results in the README:** a "Results (simulated)" section; each of the 4 questions with a small table from the CSV files. Shorten where it helps (content per year, top 5 tags); replace author names.
10. **Final docs pass:**
    - `simulation/README.md` reads as one whole: each step's real check next to its simulated check.
    - `CLAUDE.md`: the repository layout and the check commands.
    - Every link and path opens; every name matches the code; every number matches a CSV file; every simulated result says "simulated"; no record and no real author name appears.
11. **Commit** on the branch and open the pull request.

<br>

## Done when

- Every step gives its expected result.
- The 4 CSV files are in `.build/results-simulated/`, and git ignores them.
- The README renders on GitHub with the diagram, the results and working links.
