# Contributing to RepoLens

Thank you for contributing! This guide explains how our **Continuous Integration (CI)**
pipeline works, how to run checks locally, and what must pass before a PR can be merged.

---

## Table of Contents

1. [CI Overview](#ci-overview)
2. [Trigger Rules](#trigger-rules)
3. [Jobs Explained](#jobs-explained)
4. [Running Checks Locally](#running-checks-locally)
5. [Branch Protection & Merging](#branch-protection--merging)
6. [Adding Tests](#adding-tests)
7. [Troubleshooting](#troubleshooting)

---

## CI Overview

RepoLens uses **GitHub Actions** (`.github/workflows/ci.yml`) to automatically
validate every change before it lands on a protected branch.

```
PR / Push
   │
   ▼
┌─────────────────────────┐
│  Job 1 · Ruff Lint+Fmt  │  ← fails fast; no test run if this fails
└──────────┬──────────────┘
           │  success
           ▼
┌─────────────────────────┐
│  Job 2 · pytest         │  ← skipped gracefully if no tests exist
└──────────┬──────────────┘
           │  (always, on PRs only)
           ▼
┌─────────────────────────┐
│  Job 3 · PR Comment     │  ← posts / updates a CI summary comment
└─────────────────────────┘
```

---

## Trigger Rules

| Event | Branches |
|-------|----------|
| **Pull Request** (opened, synchronised, reopened) | `develop`, `main` |
| **Push** (direct commits to protected branches) | `develop`, `main`, `integration/**`, `release/**` |
| **Manual** (`workflow_dispatch`) | any branch |

> **Note:** A single in-progress run is cancelled automatically when a new
> commit is pushed to the same PR / branch (`concurrency: cancel-in-progress: true`).

---

## Jobs Explained

### 1 · Ruff – Format & Lint (`lint`)

| Step | Command | Purpose |
|------|---------|---------|
| Format check | `ruff format --check .` | Ensures code is Black-compatible formatted |
| Lint check | `ruff check .` | Catches style, import-order, and bug-prone patterns |

Configuration lives in [`pyproject.toml`](pyproject.toml) under `[tool.ruff]`.

**Fail behaviour:** If this job fails, the `test` job is skipped automatically
(via `needs: lint`), saving runner minutes.

---

### 2 · pytest (`test`)

| Step | Detail |
|------|--------|
| Install deps | `pip install pytest pytest-cov` + `requirements.txt` (if present) |
| Detect tests | Searches for `test_*.py` / `*_test.py` files |
| Run tests | `pytest --tb=short -q` |
| Skip gracefully | If no test files are found the step exits 0 with an informational notice |

Configuration lives in [`pyproject.toml`](pyproject.toml) under `[tool.pytest.ini_options]`.

---

### 3 · PR Comment (`pr-comment`)

Runs **only on pull-request events** (not on direct pushes). Posts (or updates)
a markdown table in the PR timeline showing the result of every job.

Example comment:

> ## 🟢 RepoLens CI Report
> **✅ All checks passed**
>
> | Job | Result |
> |-----|--------|
> | 🎨 Ruff Format & Lint | ✅ `success` |
> | 🧪 pytest             | ✅ `success` |

---

## Running Checks Locally

Install the development tools once:

```bash
pip install ruff pytest pytest-cov
```

Then run the same commands the CI runs:

```bash
# Format check (read-only)
ruff format --check .

# Auto-fix formatting
ruff format .

# Lint (read-only)
ruff check .

# Auto-fix lint issues
ruff check --fix .

# Tests
pytest
```

---

## Branch Protection & Merging

Once **branch protection rules** are configured on `develop` and `main` via
*Settings → Branches → Branch protection rules*, the following required status
checks will block merging:

| Required check | Job |
|----------------|-----|
| `Ruff · Format & Lint` | `lint` |
| `pytest` | `test` |

> **What this means:** A PR cannot be merged while any of these checks are
> failing or pending — GitHub will display a red ✗ next to the check name
> and grey out the merge button.

To configure branch protection yourself:
1. Go to **Settings → Branches**.
2. Click **Add rule** for `main` (repeat for `develop`).
3. Enable **"Require status checks to pass before merging"**.
4. Search for and add `Ruff · Format & Lint` and `pytest`.
5. Enable **"Require branches to be up to date before merging"** (recommended).

---

## Adding Tests

Place test files inside the `tests/` directory following the naming convention
`test_<module>.py`. pytest discovers them automatically.

```
tests/
├── test_smoke.py          # always-on sanity check
└── test_<your_module>.py  # your new tests here
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `ruff format --check` fails | Run `ruff format .` locally, commit the diff |
| `ruff check` fails | Run `ruff check --fix .` to auto-fix, then review remaining issues |
| `pytest` fails | Read the short traceback in the Actions log; fix the failing test |
| PR comment not posted | Check `pull-requests: write` permission is present in `ci.yml` |
| CI not triggered on push | Confirm the branch name matches a pattern in `on.push.branches` |
