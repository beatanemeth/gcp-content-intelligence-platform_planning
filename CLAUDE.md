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
  - The shared foundation, applied once and kept.
  - It holds everything that more than one feature uses.
  - Part A stays **_imaginary_**: it is not built now, but may be built later.
  - Its only job is to tell Part B what it can assume already exists.
  - Keep it **_minimal_**: assume only the basics Part B needs, so Part B stays simple too.
- **Part B: the _feature_.**
  - The content **pipeline** as a standalone module, added to Part A without changing it.
  - It holds everything that only the pipeline uses.

<br>

## Project goals

- **Focus:** _Data Engineering_.
- **Practice:** puts Google Professional Data Engineer knowledge into practice with _Terraform_ on _GCP_.
- **SQL:** the transformations and data quality checks.
- **CI:** a _GitHub Actions_ workflow runs TFLint.
- **Scope:** simple but _production-minded_; the maintainer can explain every part.
- **Method:** planned step by step.
