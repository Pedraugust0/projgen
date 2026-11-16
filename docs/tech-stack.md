# projgen — Technology Stack

| Field   | Value                    |
|---------|--------------------------|
| Status  | Draft                    |
| Version | 0.1.0                    |
| Date    | 2026-11-16               |
| Related | [Requirements](./requirements.md), [Architecture](./architecture.md) |

This document lists the technologies chosen for projgen and the reasoning behind each. Everything
in section 1 is part of the decided stack. Section 2 lists tools that were considered and
deliberately left out.

## 1. Decided stack

### 1.1 Core

| Concern              | Choice              | Role in projgen |
|----------------------|---------------------|-----------------|
| Language             | Python (3.13)       | Implementation of the CLI. Wide availability, excellent text/templating ecosystem. |
| Testing              | pytest              | Unit, integration and end-to-end tests. Fixtures (`tmp_path`, `monkeypatch`) fit file-generating tools well. |
| Model validation     | Pydantic v2         | Typed domain models (project metadata, presets, generation plans) with validation and clear errors. |
| Settings management  | pydantic-settings   | Layered configuration (env vars, `.env`, TOML) validated by the same models. Satisfies FR-040/FR-041. |
| CI/CD                | GitHub Actions      | Lint, type-check, test, release; existing branch-flow workflow. |

### 1.2 Runtime dependencies

| Library | Purpose | Why | Trade-offs / alternatives |
|---------|---------|-----|---------------------------|
| **Typer** | CLI framework | Built on Click; builds commands from type hints, which pairs naturally with Pydantic and a fully typed code base. Auto-generated help and shell completion (FR-050, FR-056). `typer.testing.CliRunner` simplifies E2E tests (NFR-054). | Adds Click as a transitive dependency. Alternatives: **Click** (more control, more boilerplate), **argparse** (zero deps, verbose, no completion). |
| **Rich** | Terminal output | Tables, panels, colored summaries, diffs and progress for the generation report (FR-052, FR-033). Respects `NO_COLOR` and non-TTY (NFR-021/022). Installed with Typer's `[all]` extra, or on its own. | Slightly heavier import; mitigate with lazy imports (NFR-002). |
| **Jinja2** | Template engine | De-facto standard for text templating. `StrictUndefined` gives strict variable handling (FR-023); `SandboxedEnvironment` protects against template injection (NFR-060). Custom template directories map to `FileSystemLoader` (FR-022). | Expression-heavy templates can become hard to read — keep logic in Python, not templates. Alternative: `string.Template` (too limited). |
| **questionary** (or **InquirerPy**) | Interactive prompts | Select lists, multi-select checkboxes, confirmations and validators for the interactive flow (FR-004, FR-006, FR-030). Built on prompt_toolkit. | Extra dependency tree. Alternative: Typer's built-in `typer.prompt`/`confirm` for a leaner start (recommended for v0.1, swap later; the `Prompter` protocol makes the swap local). |
| **tomli-w** | TOML writing | Python reads TOML in the stdlib (`tomllib`) but cannot write it. Needed for `projgen config init` (FR-043). | Only needed if FR-043 is implemented; a Jinja2 template could generate the small starter file instead. |
| **platformdirs** | OS-correct paths | Locates the user-level config directory on Linux/macOS/Windows (`~/.config/projgen`, `%APPDATA%`, …) for the user config file (FR-040, NFR-030). | Tiny, widely used; no practical alternative needed. |
| **`importlib.resources`** (stdlib) | Resource access | Ship built-in templates inside the wheel and read them reliably after installation. No extra dependency. | — |

### 1.3 Tooling and quality

| Tool | Purpose | Why |
|------|---------|-----|
| **uv** | Package/env manager, lock file, runner | Very fast, produces a lock file (NFR-063), manages Python versions, and enables `uv tool install` / `uvx projgen` distribution (NFR-074). Alternatives: Poetry, Hatch, pip-tools. |
| **Hatchling** | Build backend (PEP 517) | Packages the code and bundled templates into an installable wheel (required for the `projgen` terminal command). Simple, `pyproject.toml`-only, `src/` layout aware. Alternatives: `uv_build`, setuptools. |
| **Ruff** | Linter + formatter | One fast tool replacing flake8, isort, pyupgrade and black (NFR-041). |
| **mypy** (strict) or **pyright** | Static type checking | Needed for NFR-040. Pydantic ships a mypy plugin; pyright is faster and has stronger inference. *Open: pick one.* |
| **pre-commit** | Local git hooks | Runs Ruff/mypy before commit, catching issues before CI. Complements the PR checklist in the existing templates. |
| **Conventional Commits tooling** (e.g. `commitizen`) | Commit linting, changelog, version bump | Automates NFR-044, NFR-072 and NFR-073. Optional; `python-semantic-release` is the heavier alternative. |
| **pip-audit** | Dependency vulnerability scan | Satisfies NFR-063 in CI. Dependabot (config only, no code) is a good complement for update PRs. |

### 1.4 Testing extensions

| Library | Purpose | Why |
|---------|---------|-----|
| **pytest-cov** | Coverage | Enforces NFR-051 with `--cov-fail-under`. |
| **syrupy** | Snapshot testing | Compares rendered artifacts against stored snapshots (NFR-052); `--snapshot-update` makes intentional changes easy. Alternative: `pytest-regressions`, or plain golden files compared manually. |
| **pytest-mock** | Mocking helper | Cleaner than raw `unittest.mock` for faking `subprocess` (git integration) and prompts. |
| **Hypothesis** | Property-based testing | Ideal for path-safety (FR-035) and name-sanitization logic: generates hostile inputs (`../`, absolute paths, odd Unicode). Use sparingly, only for the security-critical functions. |
| **pytest-xdist** | Parallel tests | Optional; useful once the suite grows. |
| **tox** or **nox** / GitHub matrix | Multi-OS testing | Verifies NFR-030. A plain GitHub Actions matrix is enough at first. |

### 1.5 Documentation

| Tool | Purpose | Why |
|------|---------|-----|
| **MkDocs Material** | Rendered documentation site | Reads the Markdown already in `/docs`, no rewrite required. Optional, later; used on the project itself, not a projgen feature. |
| **mermaid** (in Markdown) | Diagrams | Architecture docs render natively on GitHub; diagrams stay as editable text. |

## 2. Considered and not adopted (for now)

| Item | Reason |
|------|--------|
| **GitPython** | Heavy dependency for the three git operations required (`init`, `config --get`, `commit`). Calling `git` through `subprocess` behind a small adapter is simpler and easier to mock. |
| **Cookiecutter / Copier** | They solve a similar problem, but projgen's value is a strict, opinionated generator with its own UX and validation. Copier's *update* feature is worth studying if the `update` command (open question #2) is adopted. |
| **PyYAML / ruamel.yaml** | Not needed unless the tool must *edit* existing YAML. Workflow files are rendered from templates as text. |
| **Plugin frameworks (pluggy, stevedore)** | Entry points (stdlib `importlib.metadata`) are enough if FR-071 is ever adopted. |
| **Async frameworks** | The workload is local, small and synchronous. |

## 3. Suggested baseline `pyproject.toml`

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "projgen"
requires-python = ">=3.13"
dependencies = [
  "typer>=0.12",
  "rich>=13",
  "jinja2>=3.1",
  "pydantic>=2.7",
  "pydantic-settings>=2.3",
  "platformdirs>=4",
]

[project.scripts]
projgen = "projgen.cli.app:main"

[dependency-groups]
dev = [
  "pytest>=8",
  "pytest-cov",
  "pytest-mock",
  "syrupy",
  "hypothesis",
  "ruff",
  "mypy",
  "pre-commit",
  "pip-audit",
]
```

Versions are indicative floors; pin exact versions through the lock file.

## 4. Repository conventions

| Topic | Decision |
|-------|----------|
| Layout | `src/projgen/` (src layout prevents accidental imports from the working directory), `tests/` mirroring the package. |
| Entry point | `[project.scripts] projgen = "projgen.cli.app:main"`. |
| Built-in templates | Stored under `src/projgen/presets/` and included in the wheel. |
| Branching | `feature/*` → `development` → `main` (enforced by `check-branch.yml`). |
| Commits | Conventional Commits (`feat:`, `fix:`, `ci:`, `docs:` …) — the history already follows this. |
| Python versions | 3.13. |

## 5. Installation (primary usage: terminal)

projgen is meant to be used as a terminal command (`projgen init`).

| Scenario | Command |
|----------|---------|
| Development (editable, changes apply immediately) | `uv tool install --editable .` or `pipx install --editable .` |
| After publishing to PyPI | `pipx install projgen` or `uv tool install projgen` |
| One-off run without installing | `uvx projgen init` |
| Fallback without the command on PATH | `python -m projgen init` |

Avoid `pip install` directly into the system Python on modern Linux distributions
(`externally-managed-environment`).

## 6. Proposed CI pipeline

| Job | Tools | Trigger |
|-----|-------|---------|
| Branch flow | existing `check-branch.yml` | PR → `main` |
| Lint & format | `ruff check`, `ruff format --check` | every PR |
| Type check | `mypy --strict` | every PR |
| Tests | `pytest --cov` on Ubuntu/macOS/Windows, Python 3.13 | every PR |
| Security | `pip-audit` | every PR + weekly schedule |
| Release | `uv build` + PyPI trusted publishing | tag `v*` on `main` |

## 7. Decision log

| # | Decision | Status |
|---|----------|--------|
| 1 | Python + pytest + Pydantic + pydantic-settings | Accepted |
| 2 | Typer + Rich for CLI/UX | Accepted |
| 3 | Jinja2 (sandboxed, strict) for templates | Accepted |
| 4 | uv + Hatchling + Ruff + mypy/pyright tooling | Accepted (mypy vs pyright still open) |
| 5 | syrupy snapshot tests | Accepted |
| 6 | No GitPython; use `subprocess` adapter | Accepted |
