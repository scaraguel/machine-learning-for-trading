---
type: validation architecture
title: Test System and Validation Strategy
description: Layered validation for utilities, research contracts, reduced-data fixtures, Papermill notebook execution, environment-specific CI lanes, and release gates.
tags: [testing, validation, notebooks, papermill, fixtures, ci]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T03:38:57.275Z
sources:
  - id: openwiki-source-8cb5f212f3dad4676eb93d50
    resource: repo://.github/workflows/full-test.yml
  - id: openwiki-source-4f2678f93d3fd3835f9f2909
    resource: repo://.github/workflows/test.yml
  - id: openwiki-source-f0a6e7dc03522b2682f88655
    resource: repo://tests/conftest.py
  - id: openwiki-source-2e3c23e4b97bb8ab89b3d626
    resource: repo://tests/fixture_registry.py
  - id: openwiki-source-01454e27097b5306f37da3db
    resource: repo://tests/notebook_catalog.py
  - id: openwiki-source-a1e1bda80f328de3cb562469
    resource: repo://tests/notebook_worker.py
  - id: openwiki-source-7871bd05b0525f9af22bea9f
    resource: repo://tests/pm_helpers.py
  - id: openwiki-source-1d862c7dbc15167414c0a6a1
    resource: repo://tests/sample_registry_for_tests.py
  - id: openwiki-source-1cde9a5cdcd05b2ecb0dd195
    resource: repo://tests/skip_blockers.py
  - id: openwiki-source-c3ab014213c91debfb5d7cdc
    resource: repo://tests/test_case_studies.py
  - id: openwiki-source-969755044e4bb630e211fabd
    resource: repo://tests/test_chapter_notebooks.py
  - id: openwiki-source-9ee46d568b276d1b26542d53
    resource: repo://tests/test_every_fixture_file_has_a_producer.py
  - id: openwiki-source-7e8edf5412e23186428fd51a
    resource: repo://tests/test_fixture_manifest_matches_builders.py
  - id: openwiki-source-4d793b4e2edfc8f14f9a5c5f
    resource: repo://tests/test_notebook_pair_metadata_filter.py
  - id: openwiki-source-b0bab54d629d61687b74c4ae
    resource: repo://tests/test_one_producer_per_fixture_file.py
  - id: openwiki-source-3e01760cdc71fa320232d2a3
    resource: repo://tests/test_pm_helpers.py
generated: { by: "openwiki/0.9.0", at: "2026-11-10T03:38:57.275Z" }
---

# Test System and Validation Strategy

The test system has two complementary jobs. Conventional pytest tests pin small, important contracts such as artifact identity, registry behavior, loaders, fixture generation, and notebook-pair hygiene. Notebook tests then execute reader-facing `.py` notebooks through Jupytext and Papermill against a reduced but structurally representative data and artifact set. The latter are integration tests, not a replacement for focused tests: they are slower, environment-sensitive, and deliberately avoid rebuilding every upstream stage.

For artifact ownership and operational CI details, also see [Artifact Registry](../architecture/artifact-registry.md) and [Environments and CI](../operations/environments-and-ci.md). For author workflows, see [Change and Run Case Studies](../workflows/change-and-run-case-studies.md) and [Edit Notebooks](../workflows/edit-notebooks.md).

## Validation layers

| Layer | What it establishes | Representative entry points |
|---|---|---|
| Static and reader-facing guards | Formatting, import/provenance rules, paired `.py`/`.ipynb` consistency, safe paths, and notebook metadata hygiene | CI `lint` and `guards`; focused `tests/test_notebook_*` files |
| Unit tests | Pure transformations, loaders, download dispatch, path routing, environment helpers, fixture builders, and failure cases | `pytest tests/test_<area>.py` |
| Contract tests | Registry identity, immutable artifacts, supersession, research planning, retry behavior, and workspace-versus-release boundaries | [`test_research_contract_execution.py`](../../tests/test_research_contract_execution.py) and the focused registry/identity suites |
| Fixture integrity | Reduced data matches builder declarations; every fixture file has an owner; registries describe artifacts actually shipped | fixture manifest, producer, registry, and sampling tests |
| Notebook execution | The notebook source converts to an executable notebook and completes through the production code path with reduced parameters | [`test_chapter_notebooks.py`](../../tests/test_chapter_notebooks.py), [`test_case_studies.py`](../../tests/test_case_studies.py), and [`pm_helpers.py`](../../tests/pm_helpers.py) |
| Environment integration | The notebook works in the image, interpreter, service, and dependency combination assigned to it | chapter matrix, Python 3.12, Neo4j, and image-lock checks in [`test.yml`](../../.github/workflows/test.yml) |
| Release sweep | All selected chapters and all case-study matrix lanes are run manually, with an aggregate failing summary | [`full-test.yml`](../../.github/workflows/full-test.yml) |

The fast `test-unit` CI job installs a deliberately bounded dependency surface and then runs named high-value suites plus an “every other test file” sweep. The latter excludes only the explicitly justified entries in `.github/ci/unit-test-quarantine.txt`; this makes a newly added test run by default instead of depending on somebody remembering to add its filename to the workflow. Data-backed fixture checks live in `test-unit-data`, because a test that skips for missing fixture data is not evidence that the property passed.

## Notebook execution flow

```mermaid
flowchart TD
    A["Discover chapter notebooks or case-study stages"] --> B["Look up overrides and expand invocations"]
    B --> C{"Tier capability or blocker prevents this run"}
    C -->|yes| D["Report an explicit pytest skip"]
    C -->|no| E["Resolve reduced raw data"]
    E --> F["Create isolated output and release roots"]
    F --> G["Copy configs and seed intermediates and minimal results"]
    G --> H["Convert Python source to temporary ipynb with Jupytext"]
    H --> I["Inject reachable Papermill parameters"]
    I --> J["Execute in selected kernel with headless renderers"]
    J --> K{"Execution succeeded"}
    K -->|yes| L["Record pass runtime and completed stage"]
    K -->|no| M["Report failing cell and error"]
    D --> N["Append JSONL result"]
    L --> N
    M --> N
```

*Notebook discovery, fixture preparation, isolated Papermill execution, and incremental result reporting.*

### Discovery and parametrization

Chapter discovery scans top-level `NN_*` chapter directories, excludes helpers, tests, archives, and drafts, and selects notebook-like `.py` files. The chapter test also adds `data/**/dataset_card.py`. Case-study discovery scans each shipped case-study directory for numbered stages, sorts lettered producers before a bare aggregate of the same stage number, and expands an override entry’s `invocations` into independently reported pytest cases. This expansion matters when one notebook must run for several labels: each invocation receives its own test identity and result, while collection remains stage-major so all producer invocations precede the next stage.

[`notebook_catalog.py`](../../tests/notebook_catalog.py) is a separate operational inventory. It combines the same chapter discovery with numbered case-study stages, classifies them into heuristic resource lanes, and stores inventory and run status in a local SQLite catalog. [`notebook_worker.py`](../../tests/notebook_worker.py) consumes one catalog assignment in either reduced mode, through `run_notebook`, or full mode, without injected reductions. These files support managed notebook campaigns; the ordinary CI path is the two parametrized pytest modules above.

### Same code path, smaller problem

Production defaults stay in the notebook’s tagged parameters cell. [`overrides.yaml`](../../tests/overrides.yaml) changes quantities such as symbols, folds, epochs, rows, or trial counts. `run_notebook` explicitly removes legacy `TEST` and `QUICK_TEST` environment variables before launching the kernel. The tested computation therefore follows the notebook’s normal branches; only its scale and, for migrated research notebooks, its declared execution tier/workspace differ.

Papermill accepts ineffective parameters surprisingly easily. A name can be invisible to Papermill’s own parser, never read below the injection point, or unconditionally rebound before its first read. `unusable_parameters` checks all three cases. The corpus tests require every override key to name a real notebook and every declared parameter to reach its notebook on both preview and canonical fixture-generation paths. They also reject malformed multi-invocation shapes and unknown reduction vocabulary. Fix an unreachable override by making the notebook consume it or deleting it—not by adding an exception.

For research notebooks, preview routing injects `EXECUTION_TIER="preview"`, an isolated `WORKSPACE`, and a structured `PREVIEW_REDUCTIONS` mapping where supported. A reader-only downstream stage may set `research_preview: false`: it still receives the isolated workspace, but reads the seeded canonical fixture rows because no producer has populated its fresh preview registry. Canonical fixture generation strips preview-only parameters and parameters the notebook’s own canonical guard refuses.

### Execution and cleanup

`run_notebook` converts the `.py` source to a temporary `.ipynb`; it never overwrites the committed pair. Papermill executes that temporary input into a transient `_executed_*.ipynb`, saves after each cell, writes a per-process progress log under `/tmp`, reports the failing cell for `PapermillExecutionError`, and removes temporary notebooks and kernelspecs afterward. The kernel receives headless Matplotlib and Plotly settings, a repository-plus-notebook `PYTHONPATH`, thread-pool caps, and `ML4T_ENTRY_POINT`. An override may route one notebook to a separate interpreter and launcher through a temporary kernelspec.

Pytest’s report hook appends one JSON object per completed call—or setup skip—to `ML4T_RESULTS_FILE`, defaulting to `/tmp/ml4t-test-results.jsonl`. Flushing each record means partial results survive a killed process. The manual full workflow uploads the chapter result file and its final summary fails if any required lane failed.

## Data, intermediates, and isolation

### Raw reduced data is not an intermediate

`ML4T_DATA_PATH` points loaders at raw or source-shaped data. In CI this is `test-data/data` from the private `ml4t/third-edition-test-data` checkout at the deliberately moved `ci` tag. The fixture keeps schemas and useful time ranges while reducing entity counts and expensive dimensions. A missing populated fixture causes notebook tests to skip through `populated_data_dir`; CI jobs that promise data first assert that both raw data and intermediates exist so an empty checkout cannot turn into a green all-skip run.

Intermediates live beside the raw data under `test-data/intermediates`. They include case-study features, labels, evaluation data, registry rows, predictions, backtests, and selected chapter outputs. They let a downstream notebook run independently rather than paying for every producer in every test. At session setup, the harness **replaces** seeded subdirectories instead of filling missing files, preventing leftovers from an earlier run from contaminating registry identity or resolution. It then reconciles legacy prediction column names and invokes `tests/fixtures/seed_results.py` only to fill narrowly defined gaps.

The distinction guides debugging:

- a loader or source-schema failure points first to `ML4T_DATA_PATH` and the raw fixture producer;
- a missing feature, label, prediction, or registry row points first to intermediates, their sampler/generator, and registry consistency;
- a focused preview consumer may legitimately require an earlier stage in the same workspace because preview registries are intentionally not pre-seeded.

### Output and release-root isolation

`seeded_output_dir` sets `ML4T_OUTPUT_DIR` to a session/worker-specific directory, copies and reduces case-study configs, patches test model presets, and seeds intermediates there. With xdist, each worker gets its own subtree. Autouse cleanup restores process-global output routing and workspace caches after every test, preventing one test’s temporary study activation from redirecting later tests.

Isolation has a second boundary: canonical reads. A maintainer checkout can contain `case_studies/<id>/run_log` symlinks to published artifacts. The harness therefore builds a checkout-shaped `.release` tree containing source configs and tracked files but no production `run_log`, then sets `ML4T_RELEASE_ROOT` to it. A canonical-tier notebook under test can read the seeded fixture and write to its workspace without overlaying or mutating the maintainer’s published registry.

## Fixture and registry invariants

The fixture is treated as a generated product with enforceable ownership, not a bag of convenient files.

1. `tests/create_test_data.py` declares production-derived datasets, the paths each owns, and measurable reduction budgets. `manifest.json` must agree with those declarations in both directions and with the files and entity counts actually on disk.
2. `tests/generate_test_microstructure.py` is the synthetic raw-data producer. Its generated paths must not overlap paths owned by a production-derived dataset. Every production-sourced exclusion must have exactly one builder.
3. Every fixture data file must be covered by one of those producers, except the explicit shrinking `UNPRODUCED` ratchet. An exempt file that gains a producer or disappears must leave the ratchet.
4. Fixture registries must describe the fixture’s bytes. Training rows whose pinned feature or label digests are absent are stale and are pruned together with dependent prediction, metric, population, and backtest rows. Deleting only the training parent is insufficient because SQLite foreign keys are not automatically enforced for every connection.
5. Registry sampling keeps model-side rows, representative backtests, all holdouts, and complete downstream grids for retained predictions. Registry rows backed by prediction artifacts already in the fixture are captured before a resample and restored afterward, so shipped artifacts remain addressable. Pinned notebook prediction configurations bring both registry lineage and reduced parquet artifacts.
6. Generators and samplers reject an output root inside `case_studies` or one that resolves through symlinks onto a production registry.

See [Artifact Registry](../architecture/artifact-registry.md) for the production model; the tests here enforce that the reduced copy remains a coherent instance of it.

## Skips, blockers, and prerequisites

A skip is expected to remain auditable:

- `tier` routes per-commit, weekly, and on-demand notebooks; a tier mismatch is an explicit skip.
- `requires_env`, `requires_import`, and `gpu` describe runtime capabilities. GPU checks distinguish a visible Torch CUDA device from a CUDA-enabled LightGBM build.
- `skip: true` should carry a `skip_blocker`. Decidable blockers inspect the actual CI fixture path or seeded registry. When the missing file, population, candidate set, or selection appears, `honoured_skip_reason` stops honoring the skip and the blocker tests require its declaration to be retired.
- Undecidable external and fixture-shortfall blockers have stricter metadata, including tier or recent verification expectations.
- `requires_stage` is for data created only during this test session. A focused or sharded run skips the consumer and names the missing prerequisite; an in-order case-study run satisfies it.
- Case-study rebuild deferrals are checked against registry state and expire when their required input appears.

A skip caused by a fork lacking `TEST_DATA_DEPLOY_KEY` is operationally different. Data-dependent CI jobs gate their checkout and execution steps rather than omitting the job, so required check contexts can complete. Their notice explicitly says no notebook was executed; the same notebooks run after merge in a context that can read the private fixture.

## Environment assignment and CI gates

The main [`test.yml`](../../.github/workflows/test.yml) is incremental on pull requests and full-matrix on pushes to `main`, scheduled runs, post-image-publish runs, or manual `run_all`. Path filters construct chapter and case-study matrices. Changes to shared utilities, research code/config, data code, the lockfile, or notebook harness fan out broadly; a chapter or case-study-local change selects its lane. The matrix enrolls only chapter directories and numbered case-study stages present in the checkout.

Environment assignment has two parts:

- `overrides.yaml` records notebook capability metadata such as `docker_env`, `kernel_python`, `requires_import`, and `gpu`; collection applies tier/GPU/long-running markers and runtime checks capabilities.
- CI owns concrete lanes. Normal chapter and case-study tests run in the resolved `ml4t` image. An explicit Python 3.12 list runs in `ml4t-py312`; Neo4j notebooks run in dependency order against a service container; benchmark routing is currently disabled in the dynamic matrix; GPU-only coverage belongs to its dedicated tier rather than the CPU chapter matrix.

When `uv.lock`, `pyproject.toml`, or `envs/ml4t/Dockerfile` changes, CI builds and selects a candidate image from the same commit. Notebook jobs check installed distributions against `uv.lock` before execution. This prevents a dependency change from being “tested” in the older published image it replaces. See [Environments and CI](../operations/environments-and-ci.md) for image operations.

The manual [`full-test.yml`](../../.github/workflows/full-test.yml) provides `all`, `chapters-only`, and `case-studies-only` scopes. It uses per-notebook cell budgets from `overrides.yaml` and a job-level timeout rather than imposing a blanket pytest timeout shorter than legitimate notebooks. It is the pre-release breadth gate; it does not replace the always-on lint, guard, unit, data-integrity, or specialized-service checks.

## Change-to-test matrix

Use the narrow tests for fast diagnosis, then the affected integration lane. CI’s subtraction sweep and path matrix provide the broader backstop.

| Change | Minimum focused validation | Integration/breadth validation |
|---|---|---|
| Shared utility under `utils/` or `case_studies/utils/` | Its focused unit tests plus relevant contract tests; for research/registry utilities include `tests/test_research_contract_execution.py` and identity-specific suites | Run one representative consuming stage, then affected case-study and chapter lanes; CI treats these paths as shared and fans out the notebook matrix |
| Registry schema, hashes, identity, supersession, or workspace routing | Registry schema/identity/lineage tests, artifact digest/sidecar tests, and `tests/test_research_contract_execution.py` | Run producer and consumer stages against an isolated seeded output; regenerate/reconcile fixture registries if stored rows or artifacts changed |
| One case-study stage | The stage helper’s unit tests, override reachability checks, and any declared registry-write contract | `pytest tests/test_case_studies.py -v -k "<case-study> and <stage>"`; include prerequisite stages or run the whole case study when `requires_stage` applies |
| Teaching notebook | `tests/test_pm_helpers.py`, override-schema checks, import/provenance and notebook hygiene tests | `pytest tests/test_chapter_notebooks.py -v -k "<notebook>"` in its assigned image/service lane |
| Data loader or download path | Focused loader/download tests, path/data-root contracts, and synthetic edge cases | Fixture manifest/producer tests and the dependent dataset-card/chapter/case-study notebooks; regenerate reduced raw data when schema or owned paths change |
| Dockerfile, Compose mounts/ports, or dependency lock | `tests/test_compose_mounts.py`, `tests/test_compose_ports.py`, environment-lock and image-trigger tests, plus docs/compose consistency where applicable | Build the candidate image, verify it matches `uv.lock`, then run at least one notebook in each affected environment/service combination |
| Paired notebook metadata or Jupytext header | `tests/test_notebook_pair_metadata_filter.py`, pair-sync/prose/import/output hygiene tests, and provenance guards | Re-execute the exact notebook from its `.py`, sync deliberately, and confirm the committed pair remains openable and clean |

Useful focused commands are:

```bash
uv run pytest tests/test_pm_helpers.py -v --tb=short
uv run pytest tests/test_research_contract_execution.py -q --tb=short
uv run pytest tests/test_chapter_notebooks.py -v -k "<notebook>"
uv run pytest tests/test_case_studies.py -v -k "<case-study> and <stage>"
```

Notebook commands require populated reduced data and seeded intermediates, or equivalent production-shaped inputs. To reproduce CI faithfully, run them in the same resolved Docker image and service topology rather than assuming a host environment is equivalent.

## Extending the system safely

When adding a notebook, add production defaults in a tagged parameters cell, pair it correctly, and declare only reductions the source can receive. Add environment or blocker metadata only when the generic lane cannot run it. If it consumes a new intermediate, update the generator/sampler and integrity tests rather than teaching the notebook a fixture-only fallback.

When adding raw fixture data, choose exactly one producer and declare its owned path and measurable budget. When changing artifact identity or registry schema, update fixture generation and reconciliation as part of the change: a registry row without its pinned bytes, or bytes with no reachable row, is not a usable fixture. Finally, prefer a focused contract assertion for a semantic invariant and a notebook run for end-to-end wiring; relying on only one of those layers leaves either the behavior or the integration boundary untested.
