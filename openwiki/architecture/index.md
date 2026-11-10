# Files

- [Artifact Registry, Identity, and Lineage](artifact-registry.md) - How each case study's run log combines a SQLite system of record with content-addressed filesystem artifacts, immutable lineage, resumable execution, and verified release installation.
- [Data, Imports, and Output Path Runtime](data-and-path-runtime.md) - How ML4T resolves dataset, chapter, case-study, release, experiment, and test roots, and how interpreter startup makes numbered chapter helpers importable safely.
- [Repository System and Ownership Boundaries](repository-system.md) - Explains how the teaching chapters, longitudinal case studies, shared runtime code, data tree, and generated notebook and research artifacts form one system, and where responsibility passes to the separately distributed ML4T libraries.
- [Case-Study Research Runtime](research-runtime.md) - Architecture of the shared case-study runtime that resolves declared model and strategy configurations, plans identities, dispatches adapters, isolates canonical and preview storage, and exposes verified results to notebooks.
