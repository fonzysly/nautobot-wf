# App Overview

This document provides an overview of the App including critical information and important considerations when applying it to your Nautobot environment.

!!! note
    Throughout this documentation, the terms "app" and "plugin" will be used interchangeably.

## Description

Nautobot Workflow Launcher is a Git-synced workflow execution platform for Nautobot that renders dynamic forms, executes Nautobot scripts or AWX jobs, tracks run history, and reports automation time savings.

It also now includes an AI assistant layer that helps users describe outcomes in natural language, discover matching workflows, gather required inputs conversationally, request approval, and then execute the selected workflows through the existing Workflow Launcher runtime.


## Audience (User Personas) - Who should use this App?

- Network operations engineers launching repeatable operational workflows.
- Automation teams publishing Git-backed workflow catalogs for other users.
- Change-managed operations teams that want approval-gated workflow execution.
- Platform owners who need visibility into workflow demand, usage, and automation gaps.

## Authors and Maintainers

This app is maintained in the repository by the project owners and contributors listed in the source control history.

## Nautobot Features Used

The app uses Nautobot plugin extensibility for UI pages, navigation, persisted models, Git repository sync, scheduled jobs, and external integrations.

Primary features exposed by the app:

- Workflow catalog and launch UI.
- Dynamic workflow form rendering.
- Workflow run history and reporting dashboards.
- Git-synced workflow/action/input definitions.
- AI assistant conversations, capability gaps, and AI-facing workflow catalog APIs.

### Extras

The app integrates with Nautobot Extras objects such as:

- `GitRepository` for workflow and helper/filter source material.
- `ExternalIntegration` for AWX and optional AI provider configuration.
- `ScheduledJob` for delayed workflow execution.

The current AI assistant implementation is intentionally conservative:

- It plans and explains.
- It can optionally use a configured LLM to rerank already-grounded workflow matches.
- Planner reranking and post-execution summarization run asynchronously through Nautobot's Celery worker path when those roles are configured.
- It grounds retrieval in workflow catalog metadata, workflow input schema details, and recent successful execution inputs when available.
- It never invents executable automation.
- It only executes existing workflows after approval.
- Unsupported requests become capability gaps and non-executable proposals.
