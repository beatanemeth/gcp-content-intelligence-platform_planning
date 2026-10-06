# Answers and thoughts

**Responds to:** [02_claude_technical_challenges.md](02_claude_technical_challenges.md)

The numbers follow the questions in file 02. Under the reminders and warnings, I added my thoughts where I had some.

<br>

## Step 1: raw data is uploaded to the cloud

Okay, no heavy services. The data is tiny, so a small serverless step should be enough. I would rather show that I can choose the right size than use big tools for nothing.

1. **Upload tool:** a small Bash script with `gcloud storage cp`. Terraform could upload the files, but it should not.
   - If Terraform owns the files, `terraform destroy` deletes the data together with the bucket.
   - The content of the files could end up in the state file.
   - Terraform is for infrastructure. Data changes for other reasons.
   - The files on my laptop are the source of truth. Everything in the cloud can be deleted and rebuilt.
2. **Start of processing:** an event. Manual would be simpler, but an event shows how a real pipeline works: a file lands in the bucket, and processing starts. The upload happens only once, but the design should not depend on that.
3. **Complete groups:** I know three patterns:
   - A counter that tracks how many files have arrived.
   - A schedule that runs later, separately from the upload.
   - A marker file ("done file") uploaded last.

   I would choose the marker file. The upload script controls the order, so the marker can always come last. The blog files and the article files are two separate groups.

**My thoughts:**

- One bucket with folders (`raw/`, `processed/`) should be enough. Do we need more buckets?
- If the same file is uploaded twice, the data must not be loaded twice.

<br>

## Step 2: data transformation is done in the cloud

**Learn: NDJSON**

```json
[
  {"id": 1, "title": "First Post"},
  {"id": 2, "title": "Second Post"}
]
```

```json
{"id": 1, "title": "First Post"}
{"id": 2, "title": "Second Post"}
```

The first is standard JSON: one array, the whole file must be read at once. The second is NDJSON: one record per line, so it can be read line by line and split between workers.

1. **Plumbing or business logic:** removing the envelopes is technical plumbing. It adds no business value; it only prepares the data. Cleaning the HTML is similar: it is close to ingestion, not business logic. So Python does the envelopes and the HTML, and SQL does everything relational: the lookups of categories, tags and authors, combining posts and articles, the analysis. This is ELT.
2. **Cloud Run:**

   | Option              | Own Dockerfile?                  | Best use                      |
   | ------------------- | -------------------------------- | ----------------------------- |
   | Cloud Run functions | No (Google builds the container) | Event-driven                  |
   | Cloud Run services  | Yes                              | Event-driven or a web service |
   | Cloud Run jobs      | Yes                              | Batch, run to completion      |

   A function fits an event best and is the simplest. I would like to show Docker too, but not at the start. Could a refactor to a Cloud Run service be an optional step at the end?
3. **Streaming or load job:** a load job. The function writes NDJSON files to `processed/`, then starts the load jobs itself. Loading from Cloud Storage is free; streaming is charged. If the same file is processed twice, the load replaces the table's contents instead of adding the rows again. The NDJSON files also stay in the bucket, so I can reload without running Python again.

**On the identity warning:** the function gets its own service account, not the default one. It needs to read `raw/`, write `processed/`, load into BigQuery and run BigQuery jobs. The roles should be as small as possible: on the bucket and on the dataset, not on the whole project, where this works. Each workload should have its own account.

<br>

## Step 3: transformed data is stored in the cloud

1. **Layers:**
   - **Raw:** the uploaded files, unchanged (bucket).
   - **Processed:** NDJSON, envelopes removed, HTML cleaned (bucket).
   - **Staging:** one BigQuery table per file, a 1:1 copy.
   - **Cleaned:** the final shape, like `data_processed/` in the original project.
   - **Analysis:** the results.

   For the cleaned and analysis layers I think views are enough. The data is tiny, so a join at query time costs nothing, and a view never gets out of date. One dataset per layer?
2. **Tags:** an array inside the post row. There is no join, so BigQuery does not move data between workers. BigQuery stores data by column, so a query that does not select tags does not pay for them. The cost: renaming a tag means rewriting every row that contains it. Tags also have a `slug` and a `postCount`, so maybe an array of structs instead of an array of strings?

**On the schema reminder:** explicit schemas. The schemas live in JSON files in Git, and Terraform creates the staging tables from them. I would make loading strict: if an unexpected field or value arrives, the load should fail, not hide the problem.

**On the SQL ideas:** my first idea was simple: join articles and blog posts for a given year or month. I would build all the ideas as views:

- Content per month (`UNION ALL` of posts and articles).
- Tag frequency (`UNNEST`).
- Engagement per category.
- A running total and top posts per category (window functions).

<br>

## Step 4 (optional): visualisation in Looker Studio

A stretch goal, not part of the core. The core ends with the SQL results in BigQuery, shown in the README. If I build the report, it stays private, and the README shows screenshots.

<br>

## Requirements and open questions

### Simple and cost-effective

Keep it simple. Serverless only, nothing that charges while idle. A budget alert is a good idea, but I am not sure who should create it. Please propose something simple.

### Short-lived

The project and the Terraform state bucket must survive `destroy`. Everything else can go. A one-time `gcloud` script could create them, because Terraform cannot create the bucket that holds its own state. I think Terraform can enable the APIs, but `destroy` should not switch them off.

### Public repository

I will handle this myself; a reminder list is enough (`.gitignore`, dummy data). The project ID comes from a git-ignored `terraform.tfvars` locally. In GitHub Actions maybe as a secret, so the logs mask it?

### Local testing

Only where it is simple. Can the same checks run on my laptop before each commit and in GitHub Actions? For the end-to-end test, maybe upload the dummy data and check that the row counts match and the views return rows? Please propose the smallest test that is still meaningful.

### CI

GitHub Actions are important for me. `apply` and `destroy` run from my laptop only; a CI identity that can change everything would need almost `Owner` rights. For `plan`: Workload Identity Federation, so GitHub gets a short-lived token and no key. The CI account could be read-only.

### Step by step

The CI checks and the Python transformation with its tests come first; both work without the cloud. The first `destroy` and `apply` cycle should come early, while the configuration is still small. Please propose a build order.
