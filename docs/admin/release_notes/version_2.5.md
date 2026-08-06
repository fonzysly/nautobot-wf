# v2.5 Release Notes

This document describes all new features and changes in the release. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Release Overview

- Added an approval-gated AI assistant that can translate natural-language requests into grounded Workflow Launcher plans backed by the synced workflow catalog.
- Added assistant conversation history, capability-gap tracking, assistant APIs, optional LLM provider integrations, and supporting UI/documentation so teams can safely discover missing automation needs without executing unsupported actions.
- Added ServiceNow change-context support, lifecycle hooks, and async model-backed refinement so change-governed teams can use the assistant without bypassing existing approval boundaries.

<!-- towncrier release notes start -->


## [v2.5.0 (2026-06-04)](https://github.com/Network-Operations/nautobot-app-workflow-launcher/releases/tag/v2.5.0)

### Added

- Added an AI assistant experience for Workflow Launcher that accepts natural-language requests, discovers matching workflows, gathers missing inputs conversationally, and executes only approved workflow-backed plans.
- Added provider-backed LLM integration roles for planner and summarizer tasks using Nautobot `ExternalIntegration` and `SecretsGroup` configuration, with support for OpenAI, Azure OpenAI, Anthropic, Ollama, and vLLM endpoints.
- Added ServiceNow change lookup, change search, related-CI enrichment, and job-driven create/update/close lifecycle hooks for assistant conversations.
- Added assistant conversation persistence, capability-gap tracking, AI API endpoints, focused tests, and UI controls for approval, change creation, and change closure.

### Fixed

- Fixed workflow action ordering, proposal naming, and related validation/test coverage so assistant-backed planning and existing workflow execution remain deterministic.
- Fixed planner grounding and retrieval so workflow ranking uses workflow metadata, schema terms, recent execution history, and attached change context instead of relying on freeform model output.
- Fixed assistant execution summaries and planner reranking to run asynchronously through the Nautobot worker path while preserving deterministic fallback responses.

### Documentation

- Updated the README, user guides, and external-interaction documentation to describe assistant behavior, approval boundaries, optional workflow `ai:` metadata, LLM configuration, ServiceNow setup, async refinement, and the assistant JSON endpoints.

### Housekeeping

- Added the assistant data-model migrations, packaging updates, and validation/lint cleanups needed to ship the new assistant feature in the supported release workflow.
