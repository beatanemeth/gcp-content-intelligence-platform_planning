# GCP content intelligence platform: Planning

This repository plans a new portfolio project. It takes an earlier local project and redesigns it as a cloud data pipeline on **Google Cloud**, built with **Terraform**.

---

## Starting point

[Wix Content Intelligence Pipeline](https://github.com/beatanemeth/Wix-Content-Intelligence-Pipeline) is a local-first Data Science project. It processed blog and article exports with Docker and Jupyter on a laptop.

The new project keeps the data structure but changes the focus:

- **From** local analysis and presentation **to** cloud infrastructure and Data Engineering.
- **From** notebooks **to** Terraform, a Cloud Run function and BigQuery.
- **From** real Wix data **to** a fictional dataset with the same structure.
