---
type: research governance concept
title: Research Integrity and Evidence Boundaries
description: Decision-time alignment and evidence-governance invariants that keep features, sequence windows, backtests, selection, and holdout results auditable. Covers information availability, walk-forward geometry, economic feasibility, coverage, candidate lifecycle, and the one-time holdout boundary.
tags: [research-integrity, temporal-alignment, walk-forward-validation, holdout, selection, coverage, governance]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T04:21:32.609Z
sources:
  - id: openwiki-source-6a46902e3d4da387bcb40361
    resource: repo://13_dl_time_series/09_foundation_models.py
  - id: openwiki-source-6a8c91b0d91b182f7c9253e3
    resource: repo://13_dl_time_series/dl_sequences.py
  - id: openwiki-source-9088962bf46b12f8ff3c1aee
    resource: repo://25_live_trading/_etfs_features.py
  - id: openwiki-source-fd0df28fff6ca1736ed0233b
    resource: repo://case_studies/etfs/03_financial_features.py
  - id: openwiki-source-32aea03b7a7f7d2486d65b92
    resource: repo://case_studies/etfs/config/setup.yaml
  - id: openwiki-source-24486799f0ecbedf3af2abd7
    resource: repo://case_studies/research/comparison.py
  - id: openwiki-source-6dfb4d0e505283670818863e
    resource: repo://case_studies/research/cv.py
  - id: openwiki-source-c2bbf4cf8173608c9ae50286
    resource: repo://case_studies/research/decisions.py
  - id: openwiki-source-b3b0c71994e5a108e4cee913
    resource: repo://case_studies/research/holdout.py
  - id: openwiki-source-6bfdc295292f269153d329e7
    resource: repo://case_studies/research/models.py
  - id: openwiki-source-83ab13b6d384af5460cab9e4
    resource: repo://case_studies/research/population.py
  - id: openwiki-source-212033e664bfc9a9ff3f4cf7
    resource: repo://case_studies/research/selection_field.py
  - id: openwiki-source-5d56c3427fd0ee42a6d560b7
    resource: repo://case_studies/utils/backtest_runner.py
  - id: openwiki-source-25a7530a3fd9ae26155df353
    resource: repo://case_studies/utils/coverage.py
  - id: openwiki-source-01698c26ec608e878fb90f30
    resource: repo://case_studies/utils/registry/specs.py
  - id: openwiki-source-fe61e6b8bf78f3f70364e116
    resource: repo://case_studies/utils/strategy_analysis.py
  - id: openwiki-source-40b1eae867fb4b8a7532167f
    resource: repo://tests/test_backtest_runner_helpers.py
  - id: openwiki-source-d4ece3d35467d149b6948b42
    resource: repo://tests/test_chapter_sequence_windows.py
  - id: openwiki-source-f1e100b800dbe401504e0eb5
    resource: repo://tests/test_holdout_boundary.py
  - id: openwiki-source-b44400c97ce10b5f8fe447b7
    resource: repo://tests/test_holdout_second_look_refusal.py
  - id: openwiki-source-fe39cea477d84df5c150239d
    resource: repo://tests/test_leakage_detectors.py
  - id: openwiki-source-6ff29db61f66c255c777d152
    resource: repo://tests/test_the_live_etf_regime_features_respect_the_macro_availability_lag.py
  - id: openwiki-source-857c04e47472d3d31f1c02c6
    resource: repo://utils/cv_splits.py
generated: { by: "openwiki/0.9.0", at: "2026-11-10T04:21:32.609Z" }
---

# Research Integrity and Evidence Boundaries

Research integrity here is a property of the **whole evidence path**, not merely of a model or metric. A defensible result must preserve decision-time information, use a declared temporal geometry, cover the population and sessions it claims to cover, select only on validation evidence, and evaluate the selected configuration on a separately governed holdout. The repository turns those methodological rules into registry identities, immutable sets, lineage, executable refusals, and tests.

This implements the workflow principle that exploration and confirmation are separate evidence regimes. The strategy is an executable decision process evaluated as if it were live, and search must remain countable: what was tried, what was eligible, what won, and what remained unseen must all be recoverable. See the broader [Research Runtime](../architecture/research-runtime.md), [Artifact Registry](../architecture/artifact-registry.md), and [Change and Run Case Studies](../workflows/change-and-run-case-studies.md) pages.

## The evidence lifecycle

```mermaid
flowchart TD
    PIT["Point-in-time inputs"] --> CV["Resolved walk-forward folds"]
    CV --> V["Validation predictions and backtests"]
    V --> P["Declared sweep plans"]
    P --> A["Completed attempts and attestations"]
    A --> F["Eligible selection field"]
    F --> C["Immutable candidate set"]
    C --> S["Validation-only winner"]
    S --> R["Refit same configuration on derived holdout fold"]
    R --> L["Holdout second-look guard"]
    L --> H["One reported holdout evaluation"]
```

*Validation evidence is completed and frozen before a selected configuration is re-keyed to the holdout geometry; the second-look guard protects the meaning of “unseen.”*

The important boundary is between **selection** and **estimation**:

- Validation folds are reusable development evidence. Feature choices, labels, hyperparameters, allocation, and risk overlays may be compared there, subject to the declared field.
- The candidate set freezes exactly which complete validation backtests were eligible. Selection ranks within that immutable set, deterministically by validation Sharpe with an identity tie-break.
- Holdout is not another tuning fold. The selected validation specification is refitted on one derived holdout fold, with only fold-dependent fields recomputed. Its result estimates the performance of the already selected process.
- Observing a holdout result spends the window. Deleting or retiring its row cannot restore ignorance.

## Temporal invariants

### Point-in-time inputs

Every feature available at decision time must depend only on information available by that time. This includes less obvious transformations: backward fill, centered windows, globally decoded or smoothed regime probabilities, and randomized time-series splits can all import future information. The leakage detector suite treats new executable occurrences as violations unless they are reviewed, justified, and allowlisted; known but unaudited occurrences remain backlog rather than approval. Development notebooks must apply the configured holdout boundary before their first selection statistic, and forward labels must be purged by the **label endpoint**, not merely by the signal timestamp.

A practical rule is: timestamping a value at *t* does not prove that it was knowable at *t*. The producer must preserve publication lags, entity boundaries, fitting windows, and any other information-time semantics.

### Availability time governs macro features

The ETF yield-curve feature makes this distinction executable. Its Treasury spread row is an **observation about a date**, but the feature pipeline treats it as knowable only after the configured publication/availability delay. The ETF case study owns that contract in `config/setup.yaml`: the macro context declares its source, `alfred_initial_release_close_lagged` policy, one-day `availability_lag_days`, and `backward_asof` alignment. Both the research notebook and the live feature module read the lag from that configuration; the live module also reads the regime threshold and curve z-score window there instead of maintaining deployment-only copies.

Ordering is part of the contract:

1. Re-stamp each curve observation to the date on which it becomes available.
2. Sort that availability-dated series.
3. Build the sorted, unique ETF session axis.
4. Backward as-of join so each session receives only the latest value available by that session.
5. Compute the rolling curve z-score on the resulting **session** series, not on the source's calendar-day rows, and then broadcast the three regime columns back to the ETF panel.

Moving the lag after the join is too late: the session has already read unavailable information. Computing the rolling window before reducing to sessions is also a semantic change because weekend and holiday rows would count toward a window configured in trading sessions. The live regression test expresses knowability directly: for a probe session *t*, regime features computed from the complete curve must equal those computed after truncating publications to the configured visible-through time. This is stronger and clearer than checking only that two implementations happen to emit the same values.

### A target date does not imply one universal window boundary

For the Chapter 13 supervised sequence builders, labels are forward returns **dated on the decision row**. `create_sequences_multi_asset` and `create_patched_sequences_multi_asset` therefore create each `lookback`-row design window from `i - lookback + 1` through `i`, pair it with `targets[i]`, and stamp the sample with row `i`'s timestamp and symbol. The first valid sample ends at `lookback - 1`; an entity with exactly `lookback` rows contributes one sample. Including row `i` aligns these models with tabular design rows and other baselines that may use features known at that decision.

That rule is contextual, not “all windows include the target-dated value.” [`prepare_univariate_contexts`](../../13_dl_time_series/09_foundation_models.py) intentionally uses `[i - context_length:i]`: it gives a forecaster history ending at `i - 1`, then asks it to forecast from `i` onward. A safe review asks what the model consumes and what is being forecast:

- **Feature window:** when the target is a forward outcome dated at the decision row, include that row's already-available features.
- **Forecast context:** when the value at `i` begins the forecast path, stop at `i - 1`; including `i` would reveal the first value being forecast.

The alignment tests pin both ordinary and patched feature windows, their earliest admissible sample, exact-history entities, and equivalent row coverage. An apparently harmless one-index shift changes the information set and the sample population, so it belongs to the evidence contract rather than model plumbing.

### Walk-forward geometry is an identity

[`generate_cv_splits`](../../utils/cv_splits.py) extracts sorted unique timestamps, so panel row multiplicity does not alter fold cadence. It delegates calendar-aware rolling or expanding folds to `WalkForwardCV`, steps backward from the holdout boundary, returns folds oldest first, and asserts both chronological order and fold IDs. `CVSpec.resolve` records normalized boundaries and hashes the request plus the resolved folds; when an eligibility manifest is present, its keys and diagnostics also enter that identity.

The coding consequences are:

1. Generate folds from the panel's unique observation axis, not from raw row counts.
2. Respect the panel's cadence. A daily session buffer, an eight-hour panel, and a monthly panel cannot share arithmetic based on an assumed periods-per-day constant.
3. Treat fold IDs as semantic join keys. Reversing a list without renumbering it joins temporal artifacts to the wrong windows.
4. Use the canonical fold helpers rather than indexing a fold list when the intent is “earliest training start” or “most recent fold.”
5. Reject custom geometry that does not match fold-scoped temporal features. For holdout, require the chosen temporal fold to cover the requested training rows, every evaluation session, and the evaluation endpoint; if producer metadata says its fitting window reaches the holdout evaluation opening, refuse it as leakage.

### Buffers, embargoes, and the widest shared seal

The train/validation gap prevents a training label's future outcome from overlapping validation. `generate_cv_splits` interprets session-unit day buffers as counts of market sessions when a calendar exists; treating `21D` as 21 calendar days would leave only about 15 trading sessions. Validation signals whose outcome reaches the holdout are purged separately using the outcome horizon.

The derived holdout fold ends training one label buffer before the holdout opens. It counts backward on the panel's observed timestamps and measures the panel's own cadence; it does not subtract a generic calendar duration. For shared fold-scoped temporal artifacts, [`widest_label_buffer`](../../case_studies/research/holdout.py) chooses the largest buffer among the case study's primary and variant labels. A narrower shared fold is unsafe for a longer-horizon label because its features may have been fitted on observations beyond that label's allowed training boundary. Mixed session and calendar units are refused rather than compared as if they meant the same thing.

### Panel-aware decision cadence

The expected axis is the set of actual observations at which a model could decide, not a synthetic daily calendar. Coverage can be narrowed by a feature-panel decision axis, but never widened beyond non-null label timestamps. This distinction matters when an input feed is absent while prices and labels remain available: the model was blind, so the label file alone overstates feasible prediction coverage.

For intraday panels, the configured holdout end date includes the final observed session on that date. For all panels, buffer sizes are converted to observation counts using measured cadence, with a periods-per-year fallback for non-fixed durations such as months.

## Coverage is absolute, not peer-relative

Peer comparisons cannot detect a defect shared by every candidate. If a join drops the same week from all models, they still have equal coverage. The coverage gate therefore compares predictions or backtest inputs against the declaration: fold boundaries from configuration and sessions actually present with non-null labels.

A complete report requires all of the following:

- the observed fold IDs equal the declared fold IDs and timestamps stay within each fold;
- every expected session in every declared fold has at least one observation or prediction;
- the folds account for the complete declared validation window;
- where a fold subset is intentionally requested, it is a non-empty subset of declared folds;
- the checked label matches the artifact's own label when that evidence is available.

A missing file, missing time or fold column, empty frame, or unavailable declaration is an error, not a passing empty report. Coverage means observation availability, not trading activity: a flat target is still a covered decision.

**Fold coverage must also be comparable.** Candidate sets compare result protocols and reject undeclared differences. A comparison contract may explicitly name fields permitted to vary; everything else remains common protocol. Do not “fix” a coverage mismatch by intersecting candidates onto common support after the fact unless that support is itself the declared estimand—doing so changes the question and can hide a shared upstream loss.

## Declared populations, plans, and supersession

A registry row being complete and on the current identity schema does not mean its producer still publishes it. That governance statement belongs to an [`OfficialPopulation`](../../case_studies/research/population.py): a named, immutable snapshot of training, prediction, or backtest identities.

Population rules prevent common bypasses:

- Members are non-empty and unique, have the declared kind, and cannot be preview results. Canonical populations cannot admit backtests made from exploratory decision artifacts.
- A changed snapshot under an existing name must explicitly supersede the current generation. Older generations remain addressable evidence; they are not deleted.
- Resolution by name expects exactly one unsuperseded head. A fork is ambiguous and must be repaired, not resolved by recency.
- Retirement is computed **per population name and per member**. A new generation that changes three of ten identities retires those three, while unchanged identities remain live. An unrelated stale snapshot cannot restore identities their own publisher superseded.
- Positive membership is stronger than “not retired.” An experimental result that no official population ever published is not automatically eligible merely because nobody retired it.

Sweep plans are populations published before execution. They define the expected grid, making interruption visible: rows that happen to exist cannot prove the sweep finished. Each run opens a generation-specific attempt, and success publishes an attestation for that attempt. Readers require the latest attempt to be attested. Downstream generation identity includes upstream plan identities, so superseding a baseline plan invalidates attestations for allocation and risk grids derived from its old ranking.

```mermaid
stateDiagram-v2
    [*] --> Planned
    Planned --> AttemptOpen: open sweep attempt
    AttemptOpen --> Unfinished: interruption or failure
    AttemptOpen --> Attested: full successful execution
    Attested --> FieldEligible: complete plan covers pool in force
    FieldEligible --> CandidateFrozen: freeze immutable candidate set
    CandidateFrozen --> Selected: rank validation members
    Selected --> HoldoutLocked: derive refit and check spent window
    HoldoutLocked --> Reported: evaluate once
    Planned --> Superseded: declared grid changes
    Attested --> Superseded: upstream or population changes
    CandidateFrozen --> Superseded: explicit replacement binding
    Unfinished --> AttemptOpen: rerun
```

*Plans, attempts, attestations, immutable selection, and explicit supersession make incomplete or changed research states visible instead of silently rankable.*

## Selection fields and candidate sets

The **selection field** answers “what may this holdout selection choose from?” It spans the `signal`, `allocation`, and `risk_overlay` validation stages for every declared label. Every label must reach a rankable signal baseline; absence means the run is unfinished, because freezing an immutable set before that point would permanently exclude the late label.

Where sweep plans exist, they control membership and must be complete, attested, and consistent with the prediction pool in force. A missing stage plan is not permission to rank historical rows. The field drops null-Sharpe rows and refuses incomplete planned members before coverage is assessed. The winning label is read from the winner's training lineage; downstream code must not substitute `labels.primary` when a variant label won.

A [`CandidateSet`](../../case_studies/research/comparison.py) then freezes complete canonical **validation** predictions or backtests under a comparison contract. Changed membership under the same name requires an explicit supersession edge, and resolving a name with multiple live heads refuses. `open_selection_field` prefers the recorded immutable set. Its live reconstruction path exists for registries with no recorded set, but reports that weaker source explicitly and applies the same field-building rule.

Do not bypass the field by querying a catalog and sorting whatever rows are present. That loses declared-label coverage, planned-grid membership, latest-attempt attestation, supersession, protocol compatibility, and the immutable record of what selection saw.

## Decisions and state transitions

Predictions are not yet trading decisions. `DecisionArtifact.publish` validates a decision frame, records exact prediction lineage and parameters, and content-addresses the normalized artifact. `target_positions` and `orders` must declare a [`StateTransitionPolicy`](../../case_studies/research/decisions.py) for both fold boundaries and temporal gaps, choosing `continue`, `reset`, or `liquidate`. This prevents portfolio state from leaking implicitly across disconnected validation windows.

Canonical promotion is stricter than exploratory publication: it requires current complete canonical prediction lineage, an importable source identity, exact declared prediction inputs, deterministic or seeded execution, and a clean-replay digest equal to the published artifact. Decision state is therefore explicit and auditable rather than hidden in notebook control flow.

## Economic feasibility is an evidence boundary

The simplified vectorized backtest computes period gross return as `sum(weight * y_true)`. It has rebalance-date weights and forward outcomes, but no broker, maintenance-margin enforcement, liquidation fill, or position state inside the holding period. That simplification is acceptable only while the resulting period remains economically representable by the path.

For a short position, `weight * y_true` is unbounded below when the underlying rises. If aggregation produces `gross_ret < -1.0`, `_run_vectorized` refuses the run with the worst period and largest position contributions rather than registering the result. This is a **hard evidence boundary**, not simulated liquidation. Clipping the period to -100% would invent a liquidation price and timing that this path does not possess; continuing to compound through zero would no longer describe an account return. The corrective choices are to bound the short leg before this calculation or use the engine path, which holds and prices positions through time.

This refusal is deliberately checked on `gross_ret`, before post-hoc risk overlays and metric computation. A vectorized risk rule cannot repair evidence that the underlying return construction could not produce. The same boundary-oriented design appears in adjacent refusals: selected nonzero-weight positions must have usable outcomes, position-level stops and `daily_loss` cannot silently run on a close-to-close vectorized series, and funding or explicit state transitions require richer execution state.

The ordinary ruin-stop semantics elsewhere in the runner serve a different purpose for representable paths: they stop compounding once an account is exhausted and make risk-adjusted metrics unrankable. They must not be read as authorization to transform an impossible vectorized period into a simulated liquidation.

## Holdout lock and second looks

[`build_holdout_cv`](../../case_studies/research/holdout.py) derives exactly one fold from the selected validation specification and the case study's configured holdout window. Training begins at the earliest validation training start unless fold-scoped features impose a later availability floor; it ends before the holdout by the widest buffer. [`build_holdout_training_spec`](../../case_studies/research/holdout.py) replaces the CV and dispatches to the model family to recompute eligibility, effective per-fold parameters, and other fold-derived fields. A family without that hook cannot run a holdout fit. This is a refit of the selected configuration, not an opportunity to alter its mechanics.

The **holdout lock** is operationally enforced by inspecting all prediction generations already registered against that window:

- an existing refit of a different configuration is a spent-window second look and is refused by default;
- proceeding deliberately requires naming exactly the superseded prediction hashes to retire, and the retired rows are returned for visible reporting;
- stale or absent override names refuse, so an old override cannot silently authorize the next generation;
- rows whose training specification records no CV split, or a non-holdout split, are not overridable because the registry cannot establish that they were valid holdout refits.

The override is disclosure and lifecycle control, **not restoration of out-of-sample status**. Once a result was observed, a later selection may have been influenced by it.

## Hashes are identities, not runtime proof

Configuration and training hashes are cache and lineage identities. `training_hash_from_spec` hashes a canonical projection of the resolved specification; it proves which computation was *declared* and lets equivalent requests collide on one identity. It does **not** prove that a runtime override actually reached the estimator, that the library honored it, or that a device/seed/model parameter took effect.

Consequently, do not use “the hash changed” as the sole runtime attestation. Validate the executed result against the resolved spec, record effective per-fold parameters where applicable, retain source/runtime provenance, and use clean replay or explicit execution attestations when the claim is about what actually ran. A hash can faithfully identify a wrong handoff.

## Safe-change checklist

Before changing evaluation, selection, or holdout code, verify:

1. Inputs remain point-in-time; macro data is ordered by configured publication/availability time rather than assumed knowable from its observation timestamp, and label endpoints—not signal dates—define the leakage seal.
2. A supervised feature window includes the target-dated decision row when its features are then available, while a forecasting context stops before the first value it is meant to forecast.
3. Fold generation still uses unique panel timestamps, declared cadence/calendar semantics, chronological IDs, and the correct label buffer unit.
4. Shared temporal artifacts use geometry safe for the widest label buffer and cover holdout training plus the full evaluation endpoint.
5. Coverage is checked against declarations and cannot turn missing evidence into a pass.
6. Every declared label and planned stage is complete; the latest generation-specific attempt is attested.
7. The selection field contains only complete, rankable, protocol-compatible validation results from published populations.
8. Population and candidate-set changes use explicit supersession rather than deletion, recency, or silent filtering.
9. The winner's own label and immutable candidate set flow into the holdout refit.
10. Holdout-derived fields are re-keyed to one holdout fold, and no selection decision reads holdout results.
11. A spent window is refused unless the exact prior generation is deliberately named and disclosed.
12. Decision artifacts declare fold/gap state transitions, and canonical promotion carries deterministic replay evidence.
13. A vectorized period loss below -100% is refused rather than clipped or compounded; configurations needing liquidation, funding, stops, or intra-period state move to the engine path.
14. Runtime behavior is independently attested; configuration hashes are not treated as proof of override delivery.

## Focused regression gates

The most relevant executable specifications are:

- [`tests/test_leakage_detectors.py`](../../tests/test_leakage_detectors.py): default-deny AST ratchets for global decoding, random temporal splits, and future-aware fills.
- [`tests/test_holdout_boundary.py`](../../tests/test_holdout_boundary.py): configured holdout scoping before development statistics and endpoint-based forward-label purging.
- [`tests/test_coverage_gate.py`](../../tests/test_coverage_gate.py): absolute session/fold coverage and failures when coverage cannot be evaluated.
- [`tests/test_population_supersedes.py`](../../tests/test_population_supersedes.py): current-generation resolution and explicit population lineage.
- [`tests/test_holdout_second_look_refusal.py`](../../tests/test_holdout_second_look_refusal.py): default refusal, exact per-generation override, and non-overridable unattributable/non-out-of-sample rows.
- [`tests/test_chapter_sequence_windows.py`](../../tests/test_chapter_sequence_windows.py): decision-row inclusion for ordinary and patched supervised windows, earliest complete windows, and exact-history entities.
- [`tests/test_the_live_etf_regime_features_respect_the_macro_availability_lag.py`](../../tests/test_the_live_etf_regime_features_respect_the_macro_availability_lag.py): live macro features agree with an availability-truncated source, and deployment reads the case study's lag and threshold.
- [`tests/test_backtest_runner_helpers.py`](../../tests/test_backtest_runner_helpers.py): vectorized runs refuse missing outcomes and short-driven period losses below -100%, while a bounded long-only fixture remains admissible.
