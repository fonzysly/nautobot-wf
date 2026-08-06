# External Interactions

This document describes external dependencies and prerequisites for this App to operate, including system requirements, API endpoints, interconnection or integrations to other applications or services, and similar topics.

## External System Integrations

### From the App to Other Systems

- **Git repositories**: workflow YAML, helper modules, and filter modules are synced from Nautobot-managed Git repositories.
- **AWX / Controller**: AWX job templates may be launched and monitored as workflow actions through a Nautobot `ExternalIntegration`.
- **AI providers**: the assistant can call model providers through a Nautobot `ExternalIntegration` when `planner_llm_integration` is configured.
- Supported providers: `openai`, `azure_openai`, `anthropic`, `ollama`, and `vllm`.
- The provider API key is read from the attached secrets group's generic `password` secret.
- The provider metadata lives in `ExternalIntegration.extra_config`, including `provider`, `model`, and optional fields such as `deployment`, `api_version`, `temperature`, and `max_tokens`.
- Planner reranking and execution summarization are dispatched asynchronously through Celery when the corresponding LLM roles are configured.
- **ServiceNow**: the assistant can read change context through a Nautobot `ExternalIntegration` when `servicenow_external_integration_name` is configured.
- ServiceNow job mappings live in `ExternalIntegration.extra_config.ai_assistant` and resolve to Nautobot jobs by class path.
- ServiceNow create, update, and close actions are routed through configured Nautobot jobs; the assistant never writes directly to ServiceNow.
- When a conversation has a change number and `update_change_job` is configured, execution can publish structured work notes as part of the approval lifecycle.
- When a fetched change is not in an execution-ready state, the assistant warns the user and blocks approval/execution until the change is ready.
- Related CIs are normalized from configured ServiceNow change fields and included in assistant change context when available.
- The assistant also supports ServiceNow change search by keyword, requested user, assignment group, and time range through the AI API when the deployment exposes those fields.

### From Other Systems to the App

- Users and future chat integrations can call the AI assistant JSON endpoints over the Nautobot web application.
- Workflow Launcher may also be used by other systems through normal Nautobot authentication and permissions.

### AI Grounding Sources

- Workflow catalog metadata from synced workflow definitions.
- Workflow input schema details such as input keys, labels, types, and choice values.
- Recent successful workflow execution inputs when available.
- ServiceNow change text when a configured change number is supplied in the conversation.

## Nautobot REST API endpoints

### AI Assistant Endpoints

These endpoints follow the existing Nautobot authentication and permission model.

#### `GET /api/workflow-launcher/ai/catalog`

Returns the normalized workflow catalog exposed to the assistant.

Example response excerpt:

```json
{
    "workflows": [
        {
            "key": "create_vlan",
            "title": "Create VLAN",
            "description": "Create VLAN on switching infrastructure",
            "risk_level": "medium",
            "required_inputs": ["site", "vlan_id", "change_ticket"]
        }
    ]
}
```

#### `POST /api/workflow-launcher/ai/conversations`

Creates a new assistant conversation and optionally plans from the first message.

Example request:

```json
{
    "message": "Create VLAN 150 in Amsterdam"
}
```

#### `GET /api/workflow-launcher/ai/conversations`

Lists persisted assistant conversations for the authenticated user.

#### `GET /api/workflow-launcher/ai/conversations/<id>`

Returns one conversation with message history and current plan state.

This read path also reconciles any completed async planner reranking or execution summarization results into conversation state.

#### `POST /api/workflow-launcher/ai/conversations/<id>/messages`

Adds a user message, updates collected inputs, and rebuilds the grounded workflow plan.

#### `POST /api/workflow-launcher/ai/conversations/<id>/approve`

Approves and executes the current plan. This requires workflow execution permission and only works when the plan has no missing required inputs.

When a ServiceNow change is attached and `update_change_job` is configured, this execution path can also send structured work notes through the configured Nautobot job.

#### `POST /api/workflow-launcher/ai/conversations/<id>/create-change`

Builds a ServiceNow change draft from the current workflow plan and triggers the configured `create_change_job`. The returned change number is stored in conversation state.

#### `POST /api/workflow-launcher/ai/conversations/<id>/close-change`

Closes the associated ServiceNow change through the configured `close_change_job`. This only succeeds after all workflow runs in the conversation have completed successfully.

#### `POST /api/workflow-launcher/ai/servicenow/search`

Searches ServiceNow changes through the configured `ExternalIntegration` using optional keyword, requested user, assignment group, and start/end filters. Results return normalized change context, including related CIs when available.

#### `GET /api/workflow-launcher/ai/gaps`

Lists recorded capability gaps for unsupported requests.

#### `GET /api/workflow-launcher/ai/metrics`

Returns aggregate assistant usage and capability-gap counts.

### Security and Execution Boundaries

- The assistant never executes arbitrary code.
- Every executable plan step must map to a synced workflow.
- Any configured planner LLM is limited to reranking already-retrieved workflow candidates.
- Any configured summarizer LLM is limited to turning workflow run results into an operator-facing summary.
- Approval is required before execution.
- Unsupported requests create capability gaps and non-executable proposals instead of fabricated actions.
