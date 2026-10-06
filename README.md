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

The plan has two stages. The second stage builds on the first. Each stage ends in a README that summarises its results.

1. **[brainstorming/](brainstorming/):** the first discussion with Claude Code.
   - **[brainstorming/README.md](brainstorming/README.md):** all brainstorming decisions in one place.
2. **[architecture/](architecture/):** the plan for the standalone Terraform module.
   - **[architecture/README.md](architecture/README.md):** a summary of the module plan.

**Supporting folders:** `assets/` (diagrams), `dummy_data_blogs/` and `dummy_data_articles/` (the dummy dataset).
