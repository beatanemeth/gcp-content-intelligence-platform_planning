# Rules: British English

Use British English spelling (e.g., _analyse_, _optimise_, _colour_, _favour_, _capitalise_) in text written for this project. The rule depends on who defines the word, not on the file type.

**Apply to:**

- **Prose:** documentation, README, brainstorming files, commit messages.
- **Comments and docstrings** in any file, including `.tf`, `.py`, `.sql`, `.yml` and `Dockerfile`.
- **Names created in this project:** Terraform variables, outputs, locals and resource labels; Python functions, variables and classes; Dataform table and column names; script names.
- **Text values in dummy data**, such as titles and article bodies.

**Do not apply to:**

- **Syntax and keywords** defined by a language or tool: YAML keys in GitHub Actions or pre-commit, Dockerfile instructions, HCL block types.
- **Provider and API names:** Terraform resource types and arguments, GCP API fields, BigQuery functions, library identifiers.
- **Source schema field names:** dummy data keeps the field names of the real Wix exports unchanged, so the schemas stay identical.
- **Quotations and proper nouns**, such as product names and URLs.
