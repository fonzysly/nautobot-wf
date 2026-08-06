
# v2.2 Release Notes

This document describes all new features and changes in the release. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Release Overview

- Workflow UX release with improved list views, run history filtering, and listdict/table ergonomics

## [v2.2.5 (2026-03-24)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.5)

### Changed

- Published the March 24 workflow UX follow-up release as `v2.2.5`.
- Refined client-side workflow launch validation styling to align with Bootstrap 5 form layouts.

### Fixed

- Fixed client-side validation cleanup so repeat submit attempts remove stale inline errors before revalidating.
- Fixed launch-form error summaries to target only client-side validation banners and scroll users to the active summary or first invalid field.
- Fixed JavaScript submit exception handling to show a visible in-page error instead of failing silently.

## [v2.2.4 (2026-03-24)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.4)

### Changed

- Published the March 24 workflow UX release as `v2.2.4`.
- Rebuilt package artifacts so the released wheel and source distribution match the final tagged code.

### Fixed

- Fixed `listdict` submission handling to synchronize edited table rows back into the hidden JSON field before validation and submit.
- Fixed launch form submission flow to use a single guarded submit path while still recording form duration.
- Fixed grouped workflow list rendering to avoid repeating the category label inside each row.

## [v2.2.3 (2026-03-24)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.3)

### Added

- Added grouped workflow list interactions with per-category collapse/expand controls and a global “Collapse All Groups” toggle.
- Added workflow list display-mode switching between grouped list and tile views.
- Added workflow list metadata columns for last run timestamp, last run user, and last run status.
- Added run history pagination with configurable page sizes and previous/next navigation controls.
- Added persisted run history filters and page-size preferences in user session state (with explicit reset support).

### Changed

- Changed workflow list rendering to shared partial templates and dedicated static JavaScript for grouped-section behavior.
- Changed category filtering links to preserve display mode and simplify clearing category filters.
- Changed run history header count to display total matching runs instead of the current page item count.
- Changed launch form/listdict layout to use responsive table wrapping and improved column-width handling for mixed input types.

### Fixed

- Fixed run history filter state loss when refining search criteria by preserving active filter values across form submissions.
- Fixed listdict validation schema resolution by using a consistent columns lookup path with backward-compatible fallbacks.
- Fixed listdict row usability in constrained layouts by preventing actions/buttons columns from wrapping and keeping cells readable.

## [v2.2.2 (2026-03-06)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.2)

### Added

- Added multi-phase workflow action orchestration with support for `pre_actions`, `actions` (main), and `post_actions`.
- Added per-phase execution tracking on workflow runs via `phase_results`, including per-action status and timestamps.
- Added scheduled-run preparation task flow that executes pre-actions before creating the final Nautobot scheduled job.
- Added run-detail warning banner and live-progress payload flag for post-action failures while preserving core run outcomes.

### Changed

- Changed workflow sync parsing to support phase aliases (for example `pre-actions`, `before_actions`, `post-actions`) while keeping `actions` backward compatible.
- Changed immediate and scheduled launch lifecycle so runs enter `running` status earlier, improving UI progress consistency and reducing polling race conditions.
- Changed scheduled workflow UX to redirect immediately to run details while asynchronous pre-action preparation proceeds in the background.

### Fixed

- Fixed unsupported or unknown action type handling during datasource sync with explicit validation errors by phase.
- Fixed context propagation across phases so previously completed action results are available consistently to later actions.

## [v2.2.2 (2026-03-06)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.2)

### Added

- Added multi-phase workflow action orchestration with support for `pre_actions`, `actions` (main), and `post_actions`.
- Added per-phase execution tracking on workflow runs via `phase_results`, including per-action status and timestamps.
- Added scheduled-run preparation task flow that executes pre-actions before creating the final Nautobot scheduled job.
- Added run-detail warning banner and live-progress payload flag for post-action failures while preserving core run outcomes.

### Changed

- Changed workflow sync parsing to support phase aliases (for example `pre-actions`, `before_actions`, `post-actions`) while keeping `actions` backward compatible.
- Changed immediate and scheduled launch lifecycle so runs enter `running` status earlier, improving UI progress consistency and reducing polling race conditions.
- Changed scheduled workflow UX to redirect immediately to run details while asynchronous pre-action preparation proceeds in the background.

### Fixed

- Fixed unsupported or unknown action type handling during datasource sync with explicit validation errors by phase.
- Fixed context propagation across phases so previously completed action results are available consistently to later actions.

## [v2.2.1 (2026-02-16)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.1)

### Added

- Added first-class `canceled` workflow run status and scheduled-run cancellation support from the run detail page.
- Added live run progress endpoint and in-place UI updates for run detail, replacing hard page refresh behavior.
- Added progress bar UX for live run monitoring with status-aware styling and reconnect handling.
- Added scheduled execution timestamp visibility in run detail and run history UI for scheduled runs.

### Changed

- Changed scheduled datetime input behavior to use user local time in the UI and convert to UTC at submit time.
- Changed progress estimation to use historical runtime of recent workflow runs (median with outlier trimming), instead of manual duration estimates.
- Changed run detail polling to surface `user_log` output continuously while a script is executing.

### Fixed

- Fixed scheduled job creation compatibility across Nautobot versions by filtering unsupported `ScheduledJob` fields (including `approved_by_user`).
- Fixed PostgreSQL `FOR UPDATE` error when canceling scheduled runs by avoiding nullable outer-join locking.
- Fixed AWX progress serialization crash by removing ordering on a non-existent `AwxRun.created` field.
- Fixed immediate runs incorrectly saving `scheduled_for` values.
- Fixed run history UX so unscheduled rows do not display placeholder scheduled timestamps.

## [v2.2.0 (2026-02-07)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.2.0)

### Fixed

- Fixed `listdict` fields rendering as uneditable rows when clicking "Add Row". The root cause was Nautobot/Bootstrap form enhancers interfering with the hidden storage input. Both `listdict` and `list` fields now use Django's `HiddenInput` widget to prevent framework JS from altering the storage element.
- Fixed `listdict` table container placement using robust DOM insertion via `group_<name>` div lookup instead of fragile `nextSibling` navigation.
- Added `.list-dict-field-container` to all validation skip checks so dynamically created table cell inputs are not incorrectly flagged by the form-level validator.
- Added `data-label` attribute to `list` and `listdict` widgets so the JavaScript enhancer displays the proper human-readable label instead of the field key slug.
- Added diagnostic `console.debug` and `console.error` logging in the `listdict` JS enhancer to aid troubleshooting when columns data is missing or malformed.

### Added

- Comprehensive user documentation for all nine workflow input types (`string`, `integer`, `boolean`, `choice`, `multichoice`, `object`, `multiobject`, `list`, `listdict`) in the [Using the App](../../user/app_use_cases.md) guide.
- Detailed `listdict` column properties reference table, column type descriptions, and data format documentation.
- Documentation for conditional visibility (`show_if`), cascading dependencies (`depends_on`), input validation (`validators`), and workflow actions.
