
# v2.1 Release Notes

This document describes all new features and changes in the release. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Release Overview

- New `listdict` (List of Dictionaries) input type for workflow forms

## [v2.1.0 (2026-02-07)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.1.0)

### Added

- New `listdict` input type that renders as a dynamic table with predefined columns in workflow forms.
- Column schema definition via `columns` field in workflow YAML, supporting `string`, `integer`, `boolean`, and `choice` column types.
- Per-column validation: `required`, `unique`, and `validators` (regex with custom error messages) on individual columns.
- Row-level duplicate warning banner when columns marked `unique: true` contain duplicate values.
- Inline cell-level visual feedback (red highlight + tooltip) for validation errors and uniqueness violations.
- New `columns` JSON field on `WorkflowInput` model to store column schema.
- Database migration `0013_workflowinput_columns` for the new field.
- Prefill support for `listdict` fields when re-running a previous workflow.
