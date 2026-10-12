# Rebuild plan

A step-by-step plan to build the content pipeline again, in a new repository. It is based on the first build: what worked is kept, what was learned is built in from the start, and the folder layout follows Google's Terraform guidance.

The plan is self-contained: it needs no other file in this repository.

> **Authorship note:** Plan and decisions are mine. Claude Code helped with research and wording.

<br>

## The project in short

- **What:** blog and article exports from a Wix site are uploaded to GCP, cleaned, loaded into BigQuery and turned into answers to 4 content questions.
- **How:** one Terraform module creates everything: 2 buckets, a function, its trigger, 3 service accounts, BigQuery datasets, tables and views.
- **The 3 service accounts:** one identity per job, so each job can do only its own work.
  - **`content-build`:** turns the function's source code into a running service.
  - **`content-trigger`:** delivers the "file uploaded" event to the function.
  - **`content-runtime`:** the function runs as it; reads uploads, writes the cleaned files, loads BigQuery.
- **The story:** a company already runs a GCP platform (Part A, Team A). The pipeline is a new feature on it (Part B, Team B), added without changing Part A.
- **Simulation only:** no real GCP is used. Every cloud check runs on the laptop. A real deployment comes later.

<br>

## How to use it

- **Start with step 0:** it explains the story, the folder layout and the tools.
- **One step at a time:** each step is one branch and one pull request in the new repository.
- **Each step file has 3 parts:**
  1. **In plain words:** what the step builds and why.
  2. **Technical details:** resources, settings, commands, and the notes for the Terraform files.
  3. **Done when:** the check that proves the step works.
- **The plan stays as written.** What really happened, and any change from the plan, goes into `progress.md` in the new repository.

<br>

## The steps

| Step | File                                                               | What it builds                                                                                                             |
| ---- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| 0    | [step-0_overview.md](step-0_overview.md)                           | Nothing: the story, the layout, the tools                                                                                  |
| 1    | [step-1_repository.md](step-1_repository.md)                       | The repository, dummy data, pre-commit, `progress.md`                                                                      |
| 2    | [step-2_terraform-skeleton.md](step-2_terraform-skeleton.md)       | The Terraform module and root module; the 2 buckets (data and source)                                                      |
| 3    | [step-3_python-cleaning.md](step-3_python-cleaning.md)             | The Python cleaning (unwrap, clean HTML), tested on the laptop                                                             |
| 4    | [step-4_bigquery-staging.md](step-4_bigquery-staging.md)           | 3 datasets (`content_staging`, `content_cleaned`, `content_analysis`), 6 staging tables, data checks (`data_checks.sql`)   |
| 5a   | [step-5a_service-accounts.md](step-5a_service-accounts.md)         | 3 service accounts (`content-build`, `content-trigger`, `content-runtime`); 7 of the 8 grants                              |
| 5b   | [step-5b_function-and-trigger.md](step-5b_function-and-trigger.md) | The function (`content-transform`), its trigger, the 8th grant (`run.invoker`), the upload script with the `_READY` marker |
| 6    | [step-6_cleaned-views.md](step-6_cleaned-views.md)                 | 3 cleaned views: `blog_posts`, `articles`, `content`                                                                       |
| 7    | [step-7_analysis-views.md](step-7_analysis-views.md)               | 4 analysis views, one per question: `content_per_month`, `tag_frequency`, `top_posts_per_category`, `article_text_stats`   |
| 8    | [step-8_full-dataset.md](step-8_full-dataset.md)                   | The whole chain on the real exports; the results in the README                                                             |
| 9    | [step-9_real-deployment.md](step-9_real-deployment.md)             | The runbook now; the real deployment                                                                                       |
