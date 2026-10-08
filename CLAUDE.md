# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository. General working rules live in `.claude/rules/`.

<br>

## Project context

- **Purpose:** a portfolio project: a hands-on **Terraform** and **GCP** build. It rebuilds a local Docker and Jupyter project ([Wix Content Intelligence Pipeline](https://github.com/beatanemeth/Wix-Content-Intelligence-Pipeline)) as a cloud data pipeline.
- **Planning only:** this repository holds the plan. Do not write Terraform, Python, SQL or CI code here.

<br>

## The story

Imagine a company that already runs a GCP platform. This project adds a **new feature** to it: the content pipeline, delivered as a **standalone Terraform module**.

The story has two acts:

- **Part A: the _platform_.**
  - Owned by **Team A** (the platform team); created once and kept.
  - It holds everything that more than one feature uses.
  - In the story, Team A built it with Terraform. This project does not build it: a stand-in script (`bootstrap/bootstrap.sh`) creates its elements for the demo.
  - Its only job is to tell Part B what it can assume already exists.
  - Keep it **_minimal_**: assume only the basics Part B needs, so Part B stays simple too.
- **Part B: the _feature_.**
  - Owned by **Team B**. The maintainer is Team B only.
  - The content **pipeline** as a standalone module, added without changing Part A's code or removing anything from it; it only adds permissions for its own service accounts.
  - It holds everything that only the pipeline uses.

<br>

## Project goals

- **Focus:** _Data Engineering_.
- **Practice:** puts Google Professional Data Engineer knowledge into practice with _Terraform_ on _GCP_.
- **SQL:** the transformations and data quality checks.
- **CI:** pre-commit on the laptop; _GitHub Actions_ workflows run ShellCheck, Terraform checks (`fmt`, `validate`, TFLint) and Python checks (pytest, Ruff).
- **Scope:** simple but _production-minded_; the maintainer can explain every part.
- **Method:** planned step by step.
