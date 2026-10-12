# Plan review

A review of the architecture plan (`architecture/`) against the story and the aim of presenting Data Engineering skills. Each file covers one planning step and holds its corrected, final implementation.

<br>

## Files

| File                                                           | Topic                          | Diagram | Built in      |
| -------------------------------------------------------------- | ------------------------------ | ------- | ------------- |
| [step-0_story-overview.md](step-0_story-overview.md)           | Team A, Team B, Part A setup   | Yes     | None          |
| [step-1_platform-contract.md](step-1_platform-contract.md)     | What Part A provides           | Yes     | Phase 1       |
| [step-2_module-interface.md](step-2_module-interface.md)       | Inputs, outputs, names         | Yes     | Phase 2       |
| [step-3.1_storage.md](step-3.1_storage.md)                     | The two buckets                | Yes     | Phase 2       |
| [step-3.2_function.md](step-3.2_function.md)                   | The function                   | Yes     | Phases 3, 5   |
| [step-3.3_bigquery.md](step-3.3_bigquery.md)                   | Datasets, tables, views        | Yes     | Phases 4 to 7 |
| [step-4_iam.md](step-4_iam.md)                                 | Robot accounts and roles       | Yes     | Phase 5       |
| [step-5_dependencies.md](step-5_dependencies.md)               | Build and removal order        | No      | Phases 2 to 8 |
| [step-6_repository-layout.md](step-6_repository-layout.md)     | Folder tree                    | No      | All phases    |
| [step-7_ci.md](step-7_ci.md)                                   | pre-commit, GitHub Actions     | Yes     | Phases 0 to 3 |
| [step-8_build-phases.md](step-8_build-phases.md)               | The 10 phases                  | No      | All phases    |

<br>

## How each file is built

1. **In plain words:** the step explained simply, with its diagram.
2. **Technical details:** the decisions and the final implementation.
3. **Review:** how the step fits the story and the Data Engineering aim.

<br>

## Diagrams

- **[diagrams/](diagrams/):** one draw.io file and one PNG per step that has a diagram.
- **[assets/gcp-intelligence-platform/](../../assets/gcp-intelligence-platform/):** the updated overview diagrams.
