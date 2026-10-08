# Step 5: dependencies (in which order things are built and removed)

**Planning file:** [05_dependencies.md](../architecture/05_dependencies.md)

**Status:** closed

**Built in:** phases 2 to 8 (every `apply` and `destroy`)

**Review:** how this step fits the story and the Data Engineering aim is at the end of this file ([Review](#review)).

<br>

## In plain words

**In one sentence:** Terraform works out the building order by itself; the plan only adds two hints it cannot see, and says what to do if the first build trips.

**Building order, like a house:** walls before the roof. Terraform reads the code and sees what needs what. For example, the function names the code box, so the code box is built first. Things that need nothing from each other are built at the same time.

**Two hints Terraform cannot see:**

1. **The function needs the robots' badges to be handed out first.** The function names the robots, but not their badges. Without a hint, Terraform could build the function while the badges are still being handed out, and the build would fail.
2. **The views need their tables first.** A view's SQL mentions tables by name, but Terraform does not read SQL. Without a hint, it could create a view before the table it reads, and BigQuery would refuse the view.

These hints are called `depends_on`. They are used only in these two places.

**If the first build trips:** a new badge can take a few minutes to start working. If the first `terraform apply` fails with a permission error, simply run `terraform apply` again. Terraform finishes whatever is missing.

**Upload only after building:** the messenger must exist before the first file lands, or nobody hears "a new file arrived". So: first `terraform apply`, then the upload script.

**Removing everything:** `terraform destroy` removes all of Part B in reverse order, with every file and table in it, in one command. Three switches make this possible:

| Switch                              | On                 | Says                                          |
| ----------------------------------- | ------------------ | --------------------------------------------- |
| `force_destroy = true`              | Buckets            | Delete the box even when it holds files       |
| `delete_contents_on_destroy = true` | Datasets           | Delete the room even when it holds tables     |
| `deletion_protection = false`       | Tables and views   | No safety lock (K, step 3.3)                  |

**What stays after `destroy`:** Part A (Team A's project, state bucket, services), plus a few things Google keeps: old builds of the function's program (a few MB), logs (deleted after 30 days), build history (free).

<br>

## Technical details

<br>

### What this step decides

| ID   | Decision                                                                               |
| ---- | -------------------------------------------------------------------------------------- |
| S5.1 | Order comes from references; `depends_on` only for the function and the view blocks    |
| S5.2 | The function waits for the build account's 3 grants and the trigger account's `eventReceiver` grant |
| S5.3 | No `time_sleep` at first; a permission error on the first `apply` is solved by running `apply` again |
| S5.4 | Data is uploaded only after `apply` has finished                                       |
| S5.5 | The `run.invoker` grant is created after the function; the short gap is harmless       |
| S5.6 | `destroy` removes everything in one command; Google-managed leftovers are accepted     |

<br>

### Refinements

None. The step is consistent with the decisions of steps 1 to 4:

- **M and M2 (step 4):** the build account's `objectViewer` grant moves to the project, with a condition; it stays in the function's `depends_on`.
- **K (step 3.3):** this step already says `deletion_protection = false` for tables **and** views.
- **H (step 3.2):** the 15-minute age limit does not change any order.

<br>

### Final implementation

#### `depends_on` (the only two places)

| Resource                    | `depends_on`                                                                  | Hidden need                                       |
| --------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------- |
| Function                    | `content-build`: `storage.objectViewer`, `artifactregistry.writer`, `logging.logWriter`; `content-trigger`: `eventarc.eventReceiver` | The build and the trigger use the grants, but the function references only the accounts |
| `cleaned` views             | Staging tables                                                                | Table names inside SQL text                       |
| `content` view              | `cleaned` views                                                               | Same                                              |
| `analysis` views            | `cleaned` views, `content` view                                               | Same                                              |

- **Each `depends_on` gets a comment** naming the hidden need (for example `# S5.2: Cloud Build uses these grants`).
- **Not in `depends_on`:** the runtime account's grants. They are used only when the function runs, after the upload.

#### Creation order (simplified)

| Wave | Created                                                                  |
| ---- | ------------------------------------------------------------------------ |
| 1    | 3 service accounts, 2 buckets, 3 datasets; the zip is built locally      |
| 2    | 7 grants (all except `run.invoker`), zip upload, 6 staging tables        |
| 3    | Cleaned views `blog_posts`, `articles`                                   |
| 4    | `content` view; the function with its trigger                            |
| 5    | 6 analysis views; the `run.invoker` grant                                |

#### First `apply` (runbook)

1. `terraform apply`
2. If it fails with a permission error from Cloud Build or Eventarc: `terraform apply` again. The failed function is marked "tainted" and rebuilt.
3. If this happens more than once: add a `time_sleep` of about 60 seconds after the grants (`hashicorp/time` provider).
4. Only then: the upload script.

#### What a later `apply` does after a change

| Change                                    | Effect                                                     |
| ----------------------------------------- | ---------------------------------------------------------- |
| Function code                             | New zip name; function rebuilt in place                    |
| View SQL                                  | View updated in place                                      |
| Staging schema: new column                | Table updated in place                                     |
| Staging schema: column removed or retyped | Table replaced; rows lost; upload again                    |
| Region                                    | Buckets, datasets and function replaced                    |

#### Done when

- **Phase 2:** `destroy`, then `apply`, works on the 2 buckets.
- **Phase 8:** `destroy`, `apply`, upload and checks work on the full system; a final `destroy` leaves only Part A and the accepted leftovers.

<br>

## Review

**Fits the story:** yes. `destroy` removes only Part B. Team A's project, state bucket and services stay, exactly as the contract says. The old builds in `gcf-artifacts` are Team A's to clean up, because that repository is shared.

**Fits the Data Engineering aim:** yes. The whole pipeline can be removed and rebuilt from the repository and the laptop's files with three commands: `apply`, upload, check. A reproducible pipeline is a core Data Engineering habit, and phase 8 proves it.

**Simplest choice everywhere:**

- **Only 2 hints (`depends_on`):** everything else comes from references.
- **No artificial wait:** run `apply` again if needed.
- **One command removes everything.**

**No refinements.** One correction to the review itself: in step 3.3, K said the plan forgets the views' safety lock. That was too strong. Step 5 of the plan already says "tables and views"; only the step 3 plan file misses it. K now says the two plan files disagree.
