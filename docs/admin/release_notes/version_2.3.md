# v2.3 Release Notes

This document describes all new features and changes in the release. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Release Overview

- Extensibility-focused release centered on repo-backed Jinja filters, shared repo-local Python helpers, and consistent filter rendering across workflow launch and execution paths

## [v2.3.0 (2026-04-09)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.3.0)

### Added

- Added support for loading custom Jinja filters from Git-synced Python modules stored under the repository's top-level `filters/` directory.
- Added a new `WorkflowFilterModule` model and sync pipeline support for storing repo-backed filter modules separately from general helper modules.
- Added the `filters_dir` plugin configuration option so deployments can override the repository path used for filter modules.
- Added sync-time validation for repo-backed filters, including `FILTERS` contract checks, duplicate-name detection, built-in filter collision detection, and clear failures for missing runtime dependencies.
- Added focused unit coverage for repo-backed filter loading, including shared `scripts/lib` imports, missing dependency handling, and restricted-manager access behavior.
- Added user, developer, and example documentation for custom filters, shared repo-local helper imports, and the expected `FILTERS` mapping contract.

### Changed

- Changed Jinja rendering to build a repo-scoped environment that combines built-in filters with repo-backed filters for the workflow's source repository.
- Changed repository package materialization to load helper modules and filter modules into a shared temporary package so custom filters can import repo-local Python code such as `scripts.lib.*`.
- Changed workflow launch-time rendering to use the same repo-backed filter resolution as execution-time rendering for dynamic choices and object filter queries.
- Changed the public documentation and README to describe repo-backed filter authoring, import conventions, runtime dependency expectations, and repository layout.

### Fixed

- Fixed custom filter availability in workflow launch forms so repo-backed filters work consistently in dynamic choice lists and rendered object lookup filters.
- Fixed repo-backed helper and filter loading to use unrestricted internal managers where available, preventing runtime lookups from failing when those support models are hidden behind restricted default managers.
- Fixed cache invalidation for repo-backed runtime artifacts so synced filter changes are reloaded with the same repository revision semantics used for script materialization.
- Fixed workflow action execution ordering so actions run in the same sequence they are declared in workflow YAML.
- Fixed repo-backed module loading to clear stale temp-backed package imports, preventing intermittent helper and filter import failures during execution and tests.
