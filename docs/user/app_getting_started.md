# Getting Started with the App

This document provides a step-by-step tutorial on how to get the App going and how to use it.

## Install the App

To install the App, please follow the instructions detailed in the [Installation Guide](../admin/install.md).

## First steps with the App

After installation and a successful Git sync:

1. Open **Apps → Workflow Launcher**.
2. Verify that at least one workflow appears in the catalog.
3. Launch a simple workflow and confirm it reaches the run detail page.
4. Open **Run History** to review execution status and logs.

If you want to use the AI assistant:

1. Open **Apps → Workflow Launcher → AI Assistant**.
2. Enter a natural-language request such as `Create VLAN 150 in Amsterdam`.
3. Review the returned plan and provide any missing required fields.
4. Approve the plan to execute it through the existing workflow engine.

If you want model-backed ranking in the assistant, configure a Nautobot `ExternalIntegration` and set `planner_llm_integration` in `PLUGINS_CONFIG`. Store the API key in the linked secrets group's generic `password` secret and set `extra_config.provider` and `extra_config.model` on the integration.

If you want the assistant to use ServiceNow change records as planning context, configure `servicenow_external_integration_name` and then reference a change number such as `CHG0012345` in your request. The assistant will read the change details server-side and use the implementation text to improve workflow selection.

If you also configure ServiceNow job mappings in `ExternalIntegration.extra_config.ai_assistant`, the assistant can create changes from approved plans, post execution updates through the configured Nautobot job, and close successful changes through an explicit API action.

## What are the next steps?

After the first successful workflow run, the usual next steps are:

- Add richer workflow metadata with the optional `ai:` block so the assistant can rank and explain workflows more accurately.
- Group workflows by category and assign category-scoped object permissions.
- Add `manual_duration_minutes` so reporting can quantify time savings.
- Review capability gaps created by unsupported AI assistant requests to guide future automation work.

You can check out the [Use Cases](app_use_cases.md) section for more examples.

## Cancel a scheduled workflow run

You can cancel a workflow run only while it is still in **Scheduled** status.

1. Open **Apps → Workflow Launcher → Run History**.
2. Open the scheduled run you want to stop.
3. Click **Cancel Scheduled Run** and confirm.

After canceling, the run status becomes **Canceled** and the scheduled execution is not started.

!!! note
    Running workflows cannot be interrupted. Cancellation is available only before execution begins.
