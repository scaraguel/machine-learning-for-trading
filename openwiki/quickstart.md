---
type: repository quickstart
title: Repository Quickstart
description: A compact orientation to the repository's notebook and case-study systems, safe startup commands, ownership boundaries, and task-specific documentation routes.
tags: [quickstart, notebooks, case-studies, development, testing]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T03:38:57.275Z
sources:
  - id: openwiki-source-cb2bbfd35b072105cdc87c22
    resource: repo://case_studies/README.md
  - id: openwiki-source-f317ee207e1653d2033c81a4
    resource: repo://CONTRIBUTING.md
  - id: openwiki-source-14586213345f63fc2ffbcb43
    resource: repo://docs/installation.md
  - id: openwiki-source-d0b13d17c20ac4458b12ac59
    resource: repo://docs/running-notebooks.md
  - id: openwiki-source-a263a0d7470a14abbc8c1861
    resource: repo://docs/what-this-is.md
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
  - id: openwiki-source-cc18f7c982b54a9643a61d74
    resource: repo://tests/README.md
generated: { by: "openwiki/0.9.0", at: "2026-11-10T03:38:57.275Z" }
---

# Repository Quickstart

This is the maintained code repository for *Machine Learning for Trading, 3rd Edition*: 27 teaching chapters plus nine longitudinal case studies. It is a teaching and demonstration system, not a turnkey research platform or production trading service. Start here to identify what you are changing; use the linked page for the actual workflow.

## Know the boundaries first

| Surface | Role and ownership |
|---|---|
| `01_process_is_edge/` … `27_systematic_edge/` | Teaching notebooks. Each notebook is a paired Jupytext `.py` source and generated `.ipynb` render; edit the `.py`, not the `.ipynb`. Most notebooks are independently runnable once their data exists. |
| `case_studies/{id}/` | Nine ordered, stateful research pipelines spanning labels, features, model families, predictions, backtests, costs, risk, and analysis. A later stage consumes artifacts from earlier stages; this is not the same execution model as a chapter notebook. |
| `case_studies/research/`, `case_studies/utils/`, and `utils/` | Repository-owned shared runtime code: planning and executing research, registry/artifact handling, model and backtest helpers, configuration, paths, CV, data quality, and notebook support. Changes here can affect many notebooks or all case studies. |
| `data/` | Download scripts, loaders, and the default dataset root. Most datasets are prerequisites rather than tracked source inputs. |
| `.ipynb`, case-study `labels/`, `features/`, `evaluation/`, and `run_log/` | Generated, downloaded, or executed state. In particular, `run_log/registry.db` and its content-addressed training, prediction, and backtest files are the case-study result record—not ad hoc JSON or notebook output. Do not treat generated state as the implementation. |
| `ml4t-data`, `ml4t-engineer`, `ml4t-models`, `ml4t-diagnostic`, `ml4t-backtest`, `ml4t-live` | External PyPI packages installed by `pyproject.toml`. This repository consumes them but does not own their implementations; determine whether a defect belongs upstream before patching local callers. |

For the larger map and import/runtime boundaries, read [Repository System and Ownership Boundaries](architecture/repository-system.md). For data roots, imports, and output redirection, read [Data, Imports, and Output Path Runtime](architecture/data-and-path-runtime.md). The persistent case-study result model is covered by [Artifact Registry, Identity, and Lineage](architecture/artifact-registry.md).

## Safe startup

Run commands from the **repository root** (the directory containing `pyproject.toml` and `docker-compose.yml`). Relative data paths, chapter imports, `matplotlibrc`, and notebook execution assume that root. On Windows, use the WSL2 Ubuntu terminal rather than native Windows Python or PowerShell.

### Local `uv`

The local environment requires Python 3.14 through the managed `uv` interpreter plus a C/C++ compiler and Python headers. Native Windows and Intel macOS are not supported local targets; use Docker there. After satisfying the platform prerequisites in [`docs/installation.md`](../docs/installation.md):

```bash
git clone https://github.com/stefan-jansen/machine-learning-for-trading.git
cd machine-learning-for-trading
cp .env.example .env
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
uv sync
uv run python scripts/verify_installation.py
ML4T_DATA_PATH="${ML4T_DATA_PATH:-$PWD/data}" uv run jupyter lab
```

Open the complete tokenized URL printed by Jupyter. Keep the `ML4T_DATA_PATH` prefix: Jupyter executes notebooks with their chapter directory as the working directory, while this absolute default keeps loaders anchored to the repository's `data/` tree.

### Docker

The default `ml4t` image covers all chapters and all case studies on CPU; a small set of notebooks requires the documented `py312`, GPU, benchmark, RAPIDS, or Neo4j environment.

```bash
git clone https://github.com/stefan-jansen/machine-learning-for-trading.git
cd machine-learning-for-trading
cp .env.example .env
docker compose pull ml4t
docker compose run --rm ml4t python scripts/verify_installation.py
docker compose up ml4t
```

Open `http://localhost:8888`. The verification command prints a `PASS` or `FAIL` for required components and exits non-zero on a required failure; run it before debugging notebook code. Environment support, compilers, specialized services, GPU setup, and image profiles belong in [Environments, Dependencies, and CI Operations](operations/environments-and-ci.md).

## Data is a separate prerequisite

Installing dependencies does not install most datasets, and opening a notebook does not fetch them. For a broad free-data start locally:

```bash
uv run python data/download_all.py --free-only
```

For a smaller first path, download only ETF market data:

```bash
uv run python data/etfs/market/download.py
```

On the Docker path, start Jupyter and run the same commands in **File → New → Terminal**, omitting `uv run`:

```bash
python data/download_all.py --free-only
```

Some datasets require API credentials or manual acquisition. A missing dataset should raise `DataNotFoundError` with its download command. Use [`data/README.md`](../data/README.md) before changing a loader merely because data is absent.

Published case-study results are a different download from raw data. To inspect the ETF analysis without retraining its model sweep:

```bash
uv run python scripts/download_artifacts.py --cs etfs
uv run python case_studies/etfs/13_model_analysis.py
uv run python case_studies/etfs/20_strategy_analysis.py
```

Downloaded run logs are accepted baseline state. Create an isolated writable experiment before changing configuration or generating alternative model/backtest results; see [Change and Run a Case Study Safely](workflows/change-and-run-case-studies.md).

## Route by task

| If the task changes… | Start here | Why |
|---|---|---|
| Teaching notebook code or prose | [Edit, Execute, and Synchronize Notebooks](workflows/edit-notebooks.md) | The `.py` is authoritative; prose synchronization, code synchronization, execution provenance, and committed outputs have different rules. |
| Shared case-study runtime in `case_studies/research/` or `case_studies/utils/` | [Case-Study Research Runtime](architecture/research-runtime.md), then [Change and Run a Case Study Safely](workflows/change-and-run-case-studies.md) | Establish planner/adapter/workspace and registry boundaries before running the affected stages. |
| Dataset download, loader, path, or output-root behavior | [Data, Imports, and Output Path Runtime](architecture/data-and-path-runtime.md) | Distinguish missing data from loader defects and preserve root/output isolation. |
| Model training, prediction, portfolio, or backtest behavior | [Change and Run a Case Study Safely](workflows/change-and-run-case-studies.md) and [Research Integrity and Evidence Boundaries](concepts/research-integrity.md) | Follow prerequisites, create new identities, and preserve validation/holdout and temporal invariants. |
| Case-study universe, labels, CV, costs, sweep, or model preset configuration | [Change and Run a Case Study Safely](workflows/change-and-run-case-studies.md) | Configuration is layered across per-case-study setup, per-label training menus, and shared presets; experiments protect the downloaded baseline. |
| Unit, contract, notebook, registry, or integration tests | [Test System and Validation Strategy](testing/test-system.md) | Choose the narrowest test that proves the behavior, while respecting Papermill overrides, seeded intermediates, output isolation, and environment shards. |
| Dependencies, Dockerfiles, Compose services, Python support, GitHub Actions, or release checks | [Environments, Dependencies, and CI Operations](operations/environments-and-ci.md) | These changes can affect several image profiles and CI jobs even when local `ml4t` passes. |

## Change discipline

Contributions fix a demonstrated defect rather than expanding the book's fixed scope. Keep one issue and one pull request per problem, include a test that fails before and passes after, and report compatibility effects on columns, artifacts, and downstream notebooks. For notebook edits, never hand-edit the generated `.ipynb`; follow the synchronization and provenance workflow above. When in doubt, run the smallest focused test first and preserve the complete failure output.
