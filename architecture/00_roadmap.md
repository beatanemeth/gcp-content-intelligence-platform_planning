# Planning roadmap

> **Reviewed:** the [plan review](../review_plan/README.md) corrects this plan step by step. Where they differ, `review_plan/` wins.

Step-by-step plan for the final Terraform and GCP setup. Starting point: the brainstorming decisions in [brainstorming/README.md](../brainstorming/README.md).

**Status values:** _Agreed_ (confirmed by the maintainer), _Delegated_ (chosen by Claude at the maintainer's request).

<br>

## Framing: two acts

| Part             | Role                                                                                   | Built in this project                                          |
| ---------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| **A: platform**  | Shared foundation, applied once and kept. Owned by an imagined platform team.          | No. A bootstrap stand-in creates only what Part B needs to run. |
| **B: feature**   | The content pipeline as a standalone Terraform module, added to Part A without changing it. | Yes                                                          |

**Ownership rule:** every resource has exactly one owner. A resource that several features would share belongs to Part A. A resource only the pipeline uses belongs to Part B.

<br>

## What changes from the brainstorming

| Topic                                       | Brainstorming                              | Now                                                              |
| ------------------------------------------- | ------------------------------------------ | ---------------------------------------------------------------- |
| Pipeline design (D0 to D3)                  | Agreed                                     | Carried over unchanged, except the `content_` name prefix (S2.2) |
| Terraform structure                         | One root configuration, no modules (D7.3)  | A reusable module plus a thin root configuration that calls it   |
| Project, state, APIs, IAM, CI (D4.x)        | One owner: the maintainer                  | Split between Part A and Part B; re-decided in steps 1 and 4     |

<br>

## Terraform background: root configuration and module

- **Root configuration:** the folder where `terraform init`, `plan` and `apply` run. It owns the **backend** (where state is stored) and the **provider configuration** (which project, which region, which credentials).
- **Module:** a folder of `.tf` files that a root calls with a `module` block. Values go in through **variables** and come out through **outputs**.
- **The module declares, the root configures:** the module lists the providers and minimum versions it needs (`required_providers`: `google`, `archive`). It contains no `provider` or `backend` block.
- **Why:** the caller decides which project and credentials to use, and state belongs to the root (one state per root). A module with its own `provider` block cannot be used with `count` or `for_each`, and removing it from a root breaks the next `plan`.
- **Effect on the demo:** both the module and the root configuration (the `deployment` folder) belong to Part B. The feature team deploys its own module into the platform's project. The root receives Part A's facts (project ID, region, state bucket) as input values and returns what Part B creates (bucket name, dataset IDs, function name). Part A's code never changes.

<br>

## Planning steps

Each step ends with a decision log in this folder. All 8 logs are written and were reviewed on 2026-10-05. No open questions remain.

| Step | Question it answers                                                                                     | Output file                | Builds on |
| ---- | ------------------------------------------------------------------------------------------------------- | -------------------------- | --------- |
| 1    | **Platform contract:** what may Part B assume already exists?                                            | `01_platform-contract.md`  | 00        |
| 2    | **Module interface:** inputs, outputs, version constraints, the root that calls the module               | `02_module-interface.md`   | 1         |
| 3    | **Resource inventory:** every Part B resource, grouped by component, traced to D1 to D3                  | `03_resource-inventory.md` | 2         |
| 4    | **IAM:** identities and roles, split between Part A and Part B                                           | `04_iam.md`                | 1, 3      |
| 5    | **Dependencies:** creation order, first-apply timing, destroy behaviour                                  | `05_dependencies.md`       | 3, 4      |
| 6    | **Repository layout** of the implementation repository                                                   | `06_repository-layout.md`  | 2         |
| 7    | **CI:** GitHub Action with TFLint, plus `terraform fmt` and `terraform validate`                         | `07_ci.md`                 | 6         |
| 8    | **Build phases and runbook:** apply, upload, verify, destroy                                             | `08_build-phases.md`       | All       |

<br>

## Earlier decisions re-decided

Source: [brainstorming/07_claude_cross_cutting_topics.md](../brainstorming/07_claude_cross_cutting_topics.md).

| Earlier decision | Topic                                                                         | Now                                                    |
| ---------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------ |
| D4.1             | Bootstrap script creates the project, links billing, creates the state bucket | Kept as the Part A stand-in (S1.2, S1.3, S1.9)         |
| D4.2             | State in a Cloud Storage bucket (`gcs` backend) with object versioning        | Kept; the bucket belongs to Part A (S1.3)              |
| D4.3             | Terraform enables APIs with `disable_on_destroy = false`                      | Replaced: Part A enables the APIs (S1.4)               |
| D4.19            | Billing budget with email alerts                                              | Kept; owned by Part A (S1.7)                           |
| D4.20            | Artifact Registry repository with a cleanup policy                            | Replaced: Google-managed `gcf-artifacts` (S1.6)        |
| D4.6 to D4.9     | One service account per workload, smallest scope, additive IAM, no keys       | Kept; in Part B (S4.1 to S4.4)                         |
| D4.10 to D4.14   | CI tiers; `plan` on pull requests; no `apply` from CI                         | Simplified: offline checks only (S7.1)                 |
| D7.3             | One root configuration, no modules                                            | Replaced: module plus deployment folder (S6.2)         |
