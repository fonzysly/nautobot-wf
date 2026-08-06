# v2.4 Release Notes

This document describes all new features and changes in the release. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Release Overview

- Enhanced the `listdict` workflow input so tabular data can be imported from CSV files without introducing a separate input type.
- Added a CSV template download flow and strengthened client/server validation for imported tabular data, including choice and boolean normalization.

## [v2.4.1 (2026-05-15)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.4.1)

### Fixed

- Restored `WorkflowAction.order` after it was dropped from the packaged model so action ordering remains part of the persisted workflow definition.
- Restored the missing `0017_workflowaction_order` migration for upgrade compatibility with environments expecting that migration in the app package.
- Fixed repository sync to persist per-phase workflow action order from YAML definitions when rebuilding actions.
- Fixed phase execution ordering to honor `WorkflowAction.order` and fall back to primary key ordering when existing rows share the same default value.

## [v2.4.0 (2026-05-14)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.4.0)

### Added

- Added CSV import support to `listdict` inputs so users can populate tabular workflow data from uploaded CSV files and then review/edit the rows inline before submission.
- Added per-field CSV template download for `listdict` inputs, generating headers from the configured column schema.
- Added focused form tests covering imported `listdict` data coercion, duplicate detection, choice normalization, and invalid row handling.

### Changed

- Changed `listdict` documentation and README guidance to describe CSV upload/template download as part of the existing tabular input model.
- Changed CSV choice-column handling so uploaded values can match either the configured choice value or the user-facing choice label.

### Fixed

- Fixed imported choice values not binding correctly to `listdict` dropdown columns when the CSV used the choice label instead of the stored value.
- Fixed malformed or ambiguous CSV input handling for `listdict` uploads with clearer row-aware feedback for column mismatches and invalid boolean values.