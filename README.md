# GCP content intelligence platform: planning

This repository plans a new portfolio project. It takes an earlier local project and redesigns it as a cloud data pipeline on **Google Cloud**, built with **Terraform**.

<br>

## Starting point

[Wix Content Intelligence Pipeline](https://github.com/beatanemeth/Wix-Content-Intelligence-Pipeline) is a local-first Data Science project. It processed blog and article exports with Docker and Jupyter on a laptop.

The new project keeps the data structure but changes the focus:

- **From** local analysis and presentation **to** cloud infrastructure and Data Engineering.
- **From** notebooks **to** Terraform, a Cloud Run function and BigQuery.
- **From** real Wix data **to** a fictional dataset with the same structure.

<br>

## Planning flow

The plan has four stages. Each stage builds on the one before. Each stage ends in a README that summarises its results.

1. **[history/v1_brainstorming/](history/v1_brainstorming/):** the first discussion with Claude Code.
   - **[history/v1_brainstorming/README.md](history/v1_brainstorming/README.md):** all brainstorming decisions in one place.
2. **[history/v2_architecture/](history/v2_architecture/):** the plan for the standalone Terraform module.
   - **[history/v2_architecture/README.md](history/v2_architecture/README.md):** a summary of the module plan.
3. **[history/v3_review_plan/](history/v3_review_plan/):** the review of each architecture step, with refinements and the final implementation details.
   - **[history/v3_review_plan/README.md](history/v3_review_plan/README.md):** an index of the reviewed steps.
4. **[rebuild_plan/](rebuild_plan/):** a clean, step-by-step build plan for the pipeline, with the lessons of a first build already included. It needs no other file in this repository.
   - **[rebuild_plan/README.md](rebuild_plan/README.md):** the project in short, how to use the plan, and the steps.

**Supporting folders:** `assets/` (diagrams), `dummy_data_blogs/` and `dummy_data_articles/` (the dummy dataset).
