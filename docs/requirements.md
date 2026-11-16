# projgen — Requirements Specification

| Field   | Value                    |
|---------|--------------------------|
| Status  | Draft                    |
| Version | 0.1.0                    |
| Date    | 2026-11-16               |
| Related | [Tech Stack](./tech-stack.md), [Architecture](./architecture.md) |

## 1. Introduction

### 1.1 Purpose

projgen is a Python command-line interface that automates the initial setup of software
repositories. It generates standardized documentation and strict issue / pull request
templates so that quality assurance and organization are enforced from day one.

### 1.2 Scope

**In scope**

- Generating repository scaffolding files (README, issue templates, PR template,
  contribution/documentation files, CI workflows).
- Interactive and non-interactive (flag-driven) operation.
- Configuration of generation defaults via files, environment variables and flags.
- Safe handling of already-existing files (no silent overwrites).

**Out of scope (for the initial versions)**

- Creating the remote repository on GitHub/GitLab (API integration).
- Generating application source code or language-specific project skeletons.
- Hosting or serving documentation.

### 1.3 Definitions

| Term       | Meaning |
|------------|---------|
| Template   | A Jinja2 file that renders into one generated artifact. |
| Preset     | A named, versioned collection of templates and default settings (e.g. `default`, `minimal`, `strict`). |
| Artifact   | A single file produced by the generator. |
| Target dir | The directory where artifacts are written (default: current working directory). |
| Dry run    | An execution that computes and reports changes without writing to disk. |

### 1.4 Requirement conventions

- **FR-xxx**: functional requirement. **NFR-xxx**: non-functional requirement.
- Priority uses MoSCoW: **Must**, **Should**, **Could**, **Won't (now)**.

## 2. Functional Requirements

### 2.1 Project initialization

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-001 | The CLI shall provide an `init` command that generates the full scaffolding in the target directory. | Must |
| FR-002 | The `init` command shall accept the target directory as an optional argument, defaulting to the current directory. | Must |
| FR-003 | The CLI shall collect project metadata: project name, short description, author, license, default branch, and repository URL (optional). | Must |
| FR-004 | When run in an interactive terminal and metadata is missing, the CLI shall prompt the user for it with sensible defaults. | Must |
| FR-005 | The CLI shall support a non-interactive mode (`--no-input` / `--yes`) in which all values come from flags, configuration or defaults, and fail with a clear error if a required value is missing. | Must |
| FR-006 | The CLI shall allow the user to select which artifact groups to generate (e.g. `--only issue-templates,pr-template` or `--skip ci`). | Should |
| FR-007 | The CLI shall infer defaults from the environment where possible (directory name as project name, `git config user.name` as author, existing `git remote` as repository URL). | Should |

### 2.2 Generated artifacts

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-010 | The CLI shall generate a `README.md` populated with the project metadata. | Must |
| FR-011 | The CLI shall generate GitHub issue templates under `.github/ISSUE_TEMPLATE/`: bug report, feature request and task. | Must |
| FR-012 | Generated issue templates shall follow a strict, consistent structure with the sections *Objective*, *Action Items*, *Dependencies* and *Additional Context*. | Must |
| FR-013 | The CLI shall generate a pull request template at `.github/pull_request_template.md` containing an objective, a review checklist and a related-issues section. | Must |
| FR-014 | The CLI shall generate a CI workflow enforcing branch flow (e.g. only `development` may open PRs into `main`). | Should |
| FR-015 | The CLI shall generate additional documentation stubs (`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `LICENSE`) on request. | Should |
| FR-016 | The CLI shall support selecting the license from a built-in list of common SPDX identifiers (MIT, Apache-2.0, GPL-3.0, BSD-3-Clause, …). | Should |
| FR-017 | The CLI shall generate a `docs/` skeleton (requirements, architecture) on request. | Could |
| FR-018 | The CLI shall generate a `.gitignore` for a selected language/ecosystem. | Could |

### 2.3 Presets and templates

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-020 | The CLI shall ship with at least one built-in preset (`default`) and expose it via `--preset`. | Must |
| FR-021 | The CLI shall provide a `list` command (`projgen presets list`) that shows available presets and the artifacts each one contains. | Should |
| FR-022 | The CLI shall allow users to supply a custom template directory (`--templates-dir`) that overrides or extends built-in templates. | Should |
| FR-023 | Templates shall be rendered with a documented context (variables) and strict undefined-variable handling, so a missing variable is an error and not an empty string. | Must |
| FR-024 | The CLI shall validate presets and custom templates before rendering (`projgen validate`), reporting every problem found. | Could |

### 2.4 Safety and file handling

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-030 | The CLI shall never overwrite an existing file without explicit consent (interactive confirmation or `--force`). | Must |
| FR-031 | The CLI shall provide `--dry-run`, showing exactly which files would be created, skipped or overwritten, without touching the disk. | Must |
| FR-032 | The CLI shall provide a per-file conflict strategy: `skip`, `overwrite` or `backup` (writes `<file>.bak`). | Should |
| FR-033 | The CLI shall show a diff of the changes when a file already exists and would be modified. | Could |
| FR-034 | The CLI shall guarantee atomic writes: a failure mid-generation shall not leave a half-written file. | Should |
| FR-035 | The CLI shall refuse to write outside the target directory (path traversal protection). | Must |

### 2.5 Configuration

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-040 | Configuration shall be resolved with the precedence: CLI flags > environment variables (`PROJGEN_*`) > project config file (`.projgen.toml`) > user config file > built-in defaults. | Must |
| FR-041 | All configuration shall be validated against typed models; invalid values shall produce an actionable error naming the offending key and the accepted values. | Must |
| FR-042 | The CLI shall provide a `config` command to show the effective configuration and where each value came from. | Should |
| FR-043 | The CLI shall provide `projgen config init` to write a starter `.projgen.toml`. | Could |

### 2.6 User interface and feedback

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-050 | The CLI shall provide `--help` for every command and subcommand, with usage examples. | Must |
| FR-051 | The CLI shall provide `--version`. | Must |
| FR-052 | At the end of a run, the CLI shall print a summary listing created, skipped, overwritten and failed files. | Must |
| FR-053 | The CLI shall provide `--verbose` / `--quiet` flags to control output detail. | Should |
| FR-054 | The CLI shall use meaningful, documented exit codes (0 success, 1 generic failure, 2 usage error, 3 validation error, 4 conflict/aborted by user). | Must |
| FR-055 | The CLI shall support machine-readable output (`--output json`) for the summary, to ease scripting. | Could |
| FR-056 | The CLI shall support shell completion for bash, zsh and fish. | Could |

### 2.7 Git integration

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-060 | The CLI shall optionally run `git init` (with the configured default branch) when the target directory is not a repository (`--git-init`). | Should |
| FR-061 | The CLI shall optionally create an initial commit with the generated files (`--commit`). | Could |
| FR-062 | The CLI shall optionally create the `development` branch so the generated branch-protection workflow is usable immediately. | Could |

### 2.8 Extensibility

| ID     | Requirement | Priority |
|--------|-------------|----------|
| FR-070 | Adding a new artifact type shall require only registering a new generator/template, with no changes to the CLI layer. | Should |
| FR-071 | Third-party presets distributed as Python packages shall be discoverable via entry points. | Won't (now) |
| FR-072 | Support for GitLab (`.gitlab/`) and Bitbucket templates. | Won't (now) |

## 3. Non-Functional Requirements

### 3.1 Performance

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-001 | A full default `init` shall complete in under 1 second on commodity hardware, excluding user input time. | Should |
| NFR-002 | CLI start-up (`projgen --help`) shall take under 300 ms; heavy imports shall be lazy-loaded. | Should |

### 3.2 Reliability and correctness

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-010 | Generation shall be deterministic: the same inputs shall produce byte-identical output (no timestamps or random values unless requested). | Must |
| NFR-011 | Generation shall be idempotent: running `init` twice with the same inputs shall result in no changes on the second run. | Must |
| NFR-012 | Generation shall be transactional in effect: if any artifact fails to render, no file is written. Rendering is completed in memory before the first write. | Should |
| NFR-013 | Unexpected errors shall never show a raw stack trace by default; a `--debug` flag shall enable it. | Must |

### 3.3 Usability

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-020 | Error messages shall state what went wrong, why, and how to fix it. | Must |
| NFR-021 | Commands shall follow common CLI conventions (POSIX-style flags, `--help`, respect for `NO_COLOR`). | Must |
| NFR-022 | Output shall degrade gracefully in non-TTY environments (no colors, no prompts, no spinners). | Must |
| NFR-023 | Generated templates shall be understandable and usable without reading the projgen documentation. | Should |

### 3.4 Portability and compatibility

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-030 | The CLI shall run on Linux, macOS and Windows. | Must |
| NFR-031 | The CLI shall support all currently maintained CPython versions (initially 3.13). | Must |
| NFR-032 | Generated files shall use UTF-8 and LF line endings regardless of host OS (configurable). | Should |
| NFR-033 | Path handling shall use `pathlib` exclusively; no hard-coded separators. | Must |

### 3.5 Maintainability

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-040 | The code base shall be fully type-annotated and pass strict static type checking. | Must |
| NFR-041 | The code base shall pass linting and formatting checks with zero warnings in CI. | Must |
| NFR-042 | Business logic shall be independent of the CLI framework (CLI layer is a thin adapter). | Must |
| NFR-043 | Public functions, models and templates variables shall be documented. | Should |
| NFR-044 | Commit messages and branches shall follow a documented convention (Conventional Commits). | Should |

### 3.6 Testability and quality

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-050 | Automated tests shall be implemented with pytest and run on every PR. | Must |
| NFR-051 | Line coverage shall be ≥ 90 % overall and 100 % for the file-writing and path-safety logic. | Should |
| NFR-052 | Generated output shall be covered by snapshot/golden-file tests. | Should |
| NFR-053 | Tests shall not touch the real file system outside temporary directories nor require network access. | Must |
| NFR-054 | CLI commands shall be tested end to end through the CLI test runner, including exit codes. | Must |

### 3.7 Security

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-060 | User-supplied values shall be validated and never evaluated as code; templates shall run in a sandboxed Jinja2 environment. | Must |
| NFR-061 | Output paths shall be normalized and confined to the target directory. | Must |
| NFR-062 | The tool shall not perform network calls during generation; any future network feature must be opt-in. | Must |
| NFR-063 | Dependencies shall be pinned via a lock file and scanned for known vulnerabilities in CI. | Should |
| NFR-064 | The tool shall not collect telemetry. | Must |

### 3.8 CI/CD and release

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-070 | GitHub Actions shall run lint, type-check and tests on every PR across the supported OS/Python matrix. | Must |
| NFR-071 | The existing branch-flow workflow (`development` → `main` only) shall remain enforced. | Must |
| NFR-072 | Releases shall be built and published automatically from tagged commits; versioning follows SemVer. | Should |
| NFR-073 | Every release shall have a changelog entry. | Should |
| NFR-074 | The project shall be installable via `pip` and `pipx`/`uvx` from PyPI. | Should |

### 3.9 Documentation

| ID      | Requirement | Priority |
|---------|-------------|----------|
| NFR-080 | The repository shall contain up-to-date requirements, tech-stack and architecture docs under `/docs`, written in English. | Must |
| NFR-081 | The README shall include installation, quick start and a command reference. | Must |
| NFR-082 | All project documentation, code comments, CLI messages and identifiers shall be written in English. | Must |
| NFR-083 | Internationalization of CLI messages is not required for now. | Won't (now) |

## 4. Traceability

Each issue and PR should reference the requirement IDs it implements (e.g. `Implements FR-030, FR-031`).
Each test should carry a marker or docstring referencing the requirement it verifies.

## 5. Open questions

1. Should presets be versioned independently of the CLI?
2. Should `projgen` support an `update` command to re-sync an existing repository with newer template versions?
3. Is a plugin mechanism (FR-071) desirable before v1.0?
4. Which license will projgen itself use?
