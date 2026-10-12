# Technical challenges and questions

**Source:** [01_bea_original_vs_current_project_background.md](01_bea_original_vs_current_project_background.md)  
**Status:** brainstorming; no decisions made yet.

Responds to file 01: first the four steps of "How it is supposed to work", then "Requirements and open questions". Questions cover only the core design choices; everything else is a reminder or a warning.

<br>

## Raw data profile

| Group    | Files                        | Shape                                                           |
| -------- | ---------------------------- | --------------------------------------------------------------- |
| Blogs    | posts, categories, tags      | bare JSON arrays                                                |
| Blogs    | members                      | wrapper object, records under `_items`                          |
| Articles | articles, article categories | wrapper object, records under `items`, plus pagination metadata |

- **Size:** the full dataset is about 570 KB; the dummy data holds 2/3 of each file.
- **Relationships:** held as ID arrays (`categoryIds`, `tagIds`, `memberId`, `relatedPostIds`).
- **Nested objects:** `metrics`, `media`, `postCountInfo`, `profile`.
- **Embedded HTML:** `articleText` holds HTML markup (`<p class="font_8">…`).

<br>

## Step 1: raw data is uploaded to the cloud

- **No API:** the original project fetched the data with a JWT/FastAPI microservice. The data is now fictional, so there is no API to call. Something must replace the fetch.
- **Tiny data:** 570 KB, loaded once. None of the heavy services (Dataproc, Dataflow, Composer) is justified here, on cost or on engineering judgement. The design must justify itself by pattern, not by scale.
- **Start of processing:** the upload happens once, but something must start the processing after it: a person or an event.

**Questions:**

1. What is the simplest tool to copy files from the laptop to Cloud Storage? Could Terraform do it, and should it? (Hint: what happens to the data on `terraform destroy`, and what ends up in the state file?)
2. Should processing start by hand or by an event? What does each choice show a reviewer?
3. The files arrive one by one, but a join needs every file of its group. How does the system know that a group is complete?

<br>

## Step 2: data transformation is done in the cloud

- **Inconsistent envelopes:** bare arrays, `items`, `_items`.
- **Loading format:** BigQuery loads JSON only as newline-delimited JSON (one record per line), so a reshaping step is needed.
- **HTML:** the article text must be cleaned before analysis, as in the original notebooks.
- **Python or SQL:** SQL inside the warehouse is cheap and easy to review, but handles HTML and envelopes poorly. Python handles them well, as in the original project, but adds a service to provision and pay for. Both can coexist only if each has a clear, non-redundant role.

**Hint:** the two patterns are **ETL** (transform in compute, then load) and **ELT** (load first, then transform with SQL in the warehouse). Real-world pipelines often mix them.

**Warning:** every workload runs as an identity. The default service account has broad rights; a workload with its own account and the smallest roles is safer.

**Questions:**

1. Is removing envelopes business logic or technical plumbing? Is cleaning HTML? Which work belongs to Python, and which to SQL?
2. GCP offers Cloud Run functions, Cloud Run services and Cloud Run jobs. Which one runs on an event? Which one needs your own Dockerfile?
3. Should Python stream rows into BigQuery, or write files to Cloud Storage and start a load job? (Hint: compare the cost, and what happens if the same file is processed twice.)

<br>

## Step 3: transformed data is stored in the cloud

- **Queryable with SQL:** the stored data must answer analysis questions with SQL.
- **Relationships:** posts point to categories, tags and authors by ID. The joins that the notebooks did with DataFrames must happen in the warehouse.

**Reminder:** autodetected schemas suit exploration, not pipelines. A schema that changes silently breaks every query built on it.

**SQL ideas:** a unified content model (`UNION ALL` of posts and articles), tag frequency (`UNNEST`), engagement per category, running totals and top posts (window functions).

**Questions:**

1. Which layers does the data pass through between the raw files and the final shape? Should the final shape be stored as tables or computed as views?
2. A post has many tags. Should tags be a separate table joined by ID, or an array inside the post row? Compare storage, query cost and join complexity.

<br>

## Step 4 (optional): visualisation in Looker Studio

- **No Terraform:** Looker Studio has no Terraform provider; a report is built by hand in the browser.

**Warning:** a publicly shared report exposes its data.

<br>

## Requirements and open questions

### Tech stack

- Steps 1 to 4 cover Terraform, GCP, Python and SQL. GitHub Actions is covered under CI.
- Terraform manages infrastructure. Not everything in the project is infrastructure: data files, reports and build output have their own lifecycles.

### Simple and cost-effective

**Reminder:** the cost risk is idle resources, not data volume. Serverless services scale to zero; a budget alert warns before costs grow.

### Short-lived

**Warning:**

- Non-empty buckets and tables with deletion protection can block `terraform destroy`.
- Terraform keeps its state in a bucket, and that bucket must exist before Terraform runs. Some things must survive `destroy`.

### Public repository

- **Answer to the question in file 01:** code is written against the **schema**, not the content. The dummy data keeps the full structure (envelopes, nesting, field names), so code that works on the dummy data works on the full dataset.

**Reminder:** nothing private reaches Git: the full dataset, Terraform state, `*.tfvars`, project IDs.

### Local testing

**Reminder:** no official BigQuery emulator exists, and Terraform has no local `apply`. Static checks (`fmt`, `validate`, TFLint) and Python unit tests on the dummy data need no cloud; the real proof is an `apply` and a small end-to-end test.

### CI

**Warning:** no service account keys in a public repository. CI that runs `plan` needs keyless authentication; CI that runs `apply` needs near-`Owner` rights.

### Step by step

**Reminder:** test locally before the cloud, and run the first `destroy` and `apply` cycle early, while the configuration is small.
