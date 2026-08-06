# Upgrading the App

Here you will find any steps necessary to upgrade the App in your Nautobot environment.

## Upgrade Guide

!!! warning "Developer Note - Remove Me!"
    Add more detailed steps on how the app is upgraded in an existing Nautobot setup and any version specifics (such as upgrading between major versions with breaking changes).

When a new release comes out it may be necessary to run a migration of the database to account for any changes in the data models used by this app. Execute the command `nautobot-server post-upgrade` within the runtime environment of your Nautobot installation after updating the `nautobot-workflow-launcher` package via `pip`.

### Version Notes

- Upgrading to `2.5.0` adds AI assistant conversation, message, and capability-gap models, so the normal post-upgrade process must be completed before users access the assistant UI or assistant APIs.
- Workflows can optionally add an `ai:` metadata block after upgrading to improve assistant discovery, ranking, and plan explanations, but the metadata is not required for existing workflows to continue working.
- Upgrading to `2.4.1` restores the packaged `WorkflowAction.order` migration and re-enables deterministic action ordering within each workflow phase.
- Environments upgrading from `2.4.0` should complete the normal post-upgrade process so the restored action-order field is present before the next Git sync or workflow execution.
