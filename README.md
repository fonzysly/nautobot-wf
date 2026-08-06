# Nautobot Workflow Launcher

A general-purpose workflow launcher for Nautobot 2.4+ that provides a flexible framework for defining and executing workflows through a web interface.

## Features

- **Git Integration**: Syncs workflow definitions from Git repositories (YAML files)
- **Modular Scripts**: Support for helper modules and shared code in Git repositories
- **Custom Jinja Filters**: Load repo-backed Jinja filters from synced Python modules without rebuilding the plugin
- **Dynamic Forms**: Renders dependency-aware input forms with native Nautobot object lookups and Jinja-templated choice fields
- **Multi-Platform Execution**: Executes actions against Nautobot or external orchestrators like Ansible AWX
- **Return Value Handling**: Scripts can return values accessible to subsequent actions
- **Deterministic Action Ordering**: Pre, main, and post actions retain the order defined in workflow YAML during sync and execution
- **Job Monitoring**: Monitors AWX jobs and surfaces run status/log links in Nautobot
- **Flexible Inputs**: Supports strings, integers, booleans, choices, multi-choices, object references, multi-object selections, lists, and list dictionaries (`listdict`) with optional CSV import/template download for tabular data
- **Custom Regex Input Validators**: Supports full regex, and error message customization
- **Conditional Logic**: Execute actions based on conditional expressions with access to previous action results
- **Workflow Management**: Enable/disable workflows while preserving run history and data integrity
- **Workflow Reporting**: Comprehensive dashboard and execution history with filtering and statistics
- **Time Savings Tracking**: Track manual duration estimates and calculate time saved through automation
- **Category Organization**: Group workflows by category with filtering capabilities
- **AI Assistant**: Discover workflows from natural language, collect missing inputs conversationally, request approval, execute through Workflow Launcher, and record unsupported requests as capability gaps

## Installation

### 1. Install the Plugin

```bash
pip install nautobot-workflow-launcher
```

### 2. Configure Nautobot

Add to your `nautobot_config.py`:

```python
PLUGINS = [
    # ... other plugins
    "nautobot_workflow_launcher",
]

PLUGINS_CONFIG = {
    "nautobot_workflow_launcher": {
        # Directory within Git repository containing workflow YAML files
        "workflow_dir": "workflows",  # default

        # Directory within Git repository containing custom Jinja filter modules
        "filters_dir": "filters",  # default
        
        # ExternalIntegration name for AWX (optional)
        "awx_external_integration_name": "AWX",  # default

        # Optional ServiceNow ExternalIntegration used for change lookup and job mapping.
        "servicenow_external_integration_name": "ServiceNow Primary",

        # Optional LLM integrations backed by Nautobot ExternalIntegration records.
        # The planner role is used today to rerank already-grounded workflow candidates.
        "planner_llm_integration": "OpenAI Primary",
        "summarizer_llm_integration": "OpenAI Primary",
        "embedding_llm_integration": "",
        
        # Polling interval for AWX job monitoring (seconds)
        "awx_poll_seconds": 5,  # default

        # Timeout for outbound LLM HTTP calls (seconds)
        "llm_timeout_seconds": 30,

        # Show disabled Workflows, useful for development systems
        "show_disabled": False,

        # Show "Create New XXXX" in object inputs
        # Example, Contact will have a first option for "Create New Contact"
        # Value is always "__create__" for add new item when selected
        "show_add_new_item": ['extras.Contact']
    }
}
```

### 3. Run Migrations

```bash
nautobot-server migrate
```

If you are upgrading an existing deployment, apply migrations after updating the package so the AI assistant conversation and capability-gap models are created.

### 3a. Optional: Configure an LLM Integration for the AI Assistant

If you want model-backed ranking inside the AI assistant, create a Nautobot `ExternalIntegration` and point `planner_llm_integration` at it.

Recommended `ExternalIntegration` fields:

- `remote_url`: provider API base URL such as `https://api.openai.com/v1`
- `verify_ssl`: enabled unless you are targeting a trusted local endpoint such as Ollama or vLLM
- `secrets_group`: store the API key in the generic `password` secret; `username` is optional
- `extra_config`: include at least `provider` and `model`

Supported provider values in `extra_config`:

- `openai`
- `azure_openai`
- `anthropic`
- `ollama`
- `vllm`

Example `extra_config` for an OpenAI-compatible endpoint:

```json
{
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "temperature": 0.2,
  "max_tokens": 4000
}
```

The current assistant uses the planner role conservatively: the LLM may rerank workflow candidates that were already retrieved from the synced workflow catalog, but it cannot invent executable workflows or bypass approval.

When a planner model is configured, that reranking now runs asynchronously through the Nautobot Celery worker path. The assistant returns the grounded plan immediately and reconciles any model-backed ranking refinement on later conversation reads.

If `summarizer_llm_integration` is configured, the assistant may also produce a concise post-execution summary from completed workflow runs. If no summarizer is configured, it falls back to a deterministic server-side summary.

When a summarizer model is configured, the assistant stores the deterministic execution summary first and refines it asynchronously through the same Celery-backed path.

### 3b. Optional: Configure ServiceNow Change Context

If you want the assistant to read ServiceNow change records from messages such as `Use CHG0012345 as the basis for this work`, create a Nautobot `ExternalIntegration` and point `servicenow_external_integration_name` at it.

Recommended `ExternalIntegration` fields:

- `remote_url`: your ServiceNow instance base URL, such as `https://example.service-now.com`
- `verify_ssl`: enabled unless you have a trusted internal test endpoint
- `secrets_group`: store either username/password for basic auth or the API token in the generic `password` secret
- `extra_config`: optionally define API path, fields, and AI-assistant job mappings

Example `extra_config`:

```json
{
  "table_api_path": "/api/now/table",
  "change_table": "change_request",
  "change_fields": [
    "number",
    "short_description",
    "description",
    "implementation_plan",
    "risk",
    "impact",
    "state",
    "assignment_group",
    "start_date",
    "end_date"
  ],
  "ai_assistant": {
    "create_change_job": "my_plugin.jobs.CreateServiceNowChange",
    "update_change_job": "my_plugin.jobs.UpdateServiceNowChange",
    "close_change_job": "my_plugin.jobs.CloseServiceNowChange"
  }
}
```

Current ServiceNow behavior stays inside Workflow Launcher guardrails:

- the assistant can load change context for planning
- create, update, and close operations only occur through configured Nautobot jobs
- execution updates can post structured work notes through `update_change_job` when a change number is attached to the conversation
- change creation and closure remain explicit API actions rather than automatic direct writes to ServiceNow

### 4. Setup Git Repository

1. Create or configure a **Git Repository** in Nautobot Admin
2. Ensure the repository contains workflow YAML files in the configured directory (default: `/workflows/`)
3. Add custom Jinja filter modules in the configured filter directory (default: `/filters/`) when you need repo-specific filters
4. Place shared helper code under paths such as `/scripts/lib/` and import it from filters with standard Python imports like `from scripts.lib.network_utils import helper_name`
5. Run **Sync** on the repository to import workflows, helper modules, and filter modules


## Permissions

### Reporting
> **Object**: Workflow Run  
**Permisison**: Can View  
**Effect**: User can see reports about previous workflow runs that were executed

### Launching
> **Object**: Workflow  
**Permission**: Can View  
**Effect**: User can see the workflows

> **Object**: Workflow Run  
**Permission**: Can Add  
**Effect**: User can launch workflows

### AI Assistant
> **Object**: Workflow  
**Permission**: Can View  
**Effect**: User can open the AI Assistant, discover workflows, read the AI workflow catalog, create conversations, and review capability gaps/metrics

> **Object**: Workflow Run  
**Permission**: Can Add  
**Effect**: User can approve an AI-generated plan and execute the selected workflows

### Cancel Scheduled Runs
> **Object**: Workflow Run  
**Permission**: Can Change  
**Effect**: User can cancel runs that are still in **Scheduled** state


### Permission Scenarios

- **View Only**: Users with `view_workflow` can browse workflows, but cannot launch them.
- **Launch Workflows**: Users with `add_workflowrun` can execute workflows they have permission for.
- **View Run History**: Users with `view_workflowrun` can see stats and history for workflow runs.
- **Cancel Scheduled Runs**: Users with `change_workflowrun` can cancel runs before they start.

### Category-Based Scoping

Permissions can be limited to workflows in specific categories using constraints in the Nautobot admin UI. For example, a permission constraint like:

Restrict access to only workflows in the "IPAM" category:
```json
{"category": "IPAM"}
```

Permissions for multiple categories:
```json
{"category__in": ["IPAM","Network"]}
```


### Example: Assigning Permissions

1. **Navigate to**: Admin > Users or Groups > Object Permissions
2. **Select Object Type**: `Workflow` or `WorkflowRun`
3. **Choose Action**: `view`, `add`, or `change`
4. **Set Constraints** (optional):  

    - To restrict by category, use: `{"category": "Network"}`

5. **Save**: The user/group will now only see or launch workflows in the specified category.

### Summary Table

| Permission         | Effect                                      | Can be Scoped by Category? |
|--------------------|---------------------------------------------|:-------------------------:|
| view_workflow      | See workflows in UI/API                     | Yes                       |
| add_workflowrun    | Launch workflows                            | Yes                       |
| change_workflowrun | Cancel scheduled workflow runs              | Yes                       |
| view_workflowrun   | View workflow run history and stats         | Yes                       |



## Usage

1. Navigate to **Apps → Workflow Launcher** 
2. Select a workflow from the list (optionally filter by category)
3. Fill in the required inputs
4. Click **Run** to execute the workflow
5. View execution status and logs in the workflow run detail page

### AI Assistant

1. Navigate to **Apps → Workflow Launcher → AI Assistant**.
2. Describe the desired outcome in natural language.
3. Review the assistant's grounded workflow plan.
4. Provide any missing values requested by the assistant.
5. Approve the plan to execute it through Workflow Launcher.

Current assistant behavior:

- The assistant only recommends workflows that exist in the synced workflow catalog.
- If `planner_llm_integration` is configured, the assistant may rerank already-matched workflow candidates using the configured provider.
- If `servicenow_external_integration_name` is configured and the message includes a change number such as `CHG0012345`, the assistant loads the change context and uses it to enrich planning.
- When a fetched ServiceNow change is not in an execution-ready state, the assistant warns the user, blocks approval, and refuses execution until the change is ready.
- ServiceNow change context now includes related CIs when those fields are available from the configured ServiceNow table API.
- Retrieval is grounded in workflow metadata, workflow input schema details, and recent successful execution inputs when available.
- If a ServiceNow change is attached and `update_change_job` is configured, execution can post structured work notes through the configured Nautobot job.
- Missing required inputs are gathered conversationally and stored server-side in conversation state.
- Approval is required before execution.
- ServiceNow change creation and closure are available through explicit assistant API actions backed by configured Nautobot jobs.
- If `summarizer_llm_integration` is configured, the assistant may generate a concise result summary after execution.
- Unsupported requests are stored as capability gaps and returned with a non-executable workflow proposal.

### Canceling a Scheduled Run

- Open the workflow run detail page for a run in **Scheduled** status.
- Click **Cancel Scheduled Run** and confirm.
- The run status changes to **Canceled**.

> Note: Running workflow executions cannot be interrupted. Cancellation is only available before execution starts.

### Reporting and History

- **Dashboard**: Navigate to **Apps → Workflow Launcher → Dashboard** for execution statistics and recent runs
- **Run History**: Navigate to **Apps → Workflow Launcher → Run History** to view all workflow executions with filtering options
- **Individual Run Details**: Click on any run to view detailed execution logs, inputs, and AWX job status
- **Time Savings Reporting**: When workflows include `manual_duration_minutes`, the dashboard shows total time saved through automation over the last 30 days

## Workflow YAML Syntax

### Basic Structure

```yaml
key: my-workflow
name: "My Sample Workflow"
description: "A sample workflow that demonstrates basic functionality"
category: "Network Operations"  # Workflows are grouped by category in the UI
enabled: true                   # Optional: whether workflow is available for execution (default: true)
manual_duration_minutes: 30     # Time in minutes this workflow would take if performed manually (optional, for time savings reporting)
ai:                             # Optional AI metadata used by the assistant for discovery and planning
  category: provisioning
  aliases:
    - add vlan
    - provision vlan
  examples:
    - Create VLAN 150 in Amsterdam
  tags:
    - switching
    - provisioning
  risk_level: medium
  estimated_runtime_minutes: 5
  composable: true

inputs:
  - key: device_name
    label: "Device Name"
    type: string
    required: true
    
  - key: interface_name
    label: "Interface Name" 
    type: string
    required: true

  - key: address
    label: IPv4 CIDR
    type: string
    required: true
    validators:
      - regex: '^(?:(?:25[0-5]|2[0-4]\d|1?\d?\d)\.){3}(?:25[0-5]|2[0-4]\d|1?\d?\d)/(?:3[0-2]|[12]?\d)$'
        message: "Enter IPv4/CIDR, e.g., 192.0.2.10/24."

pre_actions:
  - type: nautobot
    name: "Create Change Ticket"
    script: "scripts/create_change_ticket.py"
    with:
      requested_by: "{{ user.username }}"
      device: "{{ inputs.device_name }}"

actions:
  - type: nautobot
    name: "Add Interface"
    script: "scripts/add_interface.py"
    with:
      device: "{{ inputs.device_name }}"
      interface: "{{ inputs.interface_name }}"

post_actions:
  - type: awx
    name: "Notify Completion"
    template_id: 99
    extra_vars:
      run_status: "{{ last.status if last else 'unknown' }}"
```

### AI Workflow Metadata

The `ai` block is optional. When present, it improves assistant ranking, composition, and explanation.

```yaml
ai:
  category: decommissioning
  aliases:
    - retire switch
    - remove switch
  examples:
    - Decommission switch AMS-SW-102
  tags:
    - decommissioning
    - lifecycle
  risk_level: high
  estimated_runtime_minutes: 20
  composable: true
```

Supported fields:

- `category`: broad operational grouping used during intent filtering.
- `aliases`: alternative phrases users may type.
- `examples`: natural-language prompts that should match the workflow.
- `tags`: extra retrieval terms.
- `risk_level`: `low`, `medium`, `high`, or `critical`.
- `estimated_runtime_minutes`: used when the assistant explains the plan.
- `composable`: whether the workflow can be included in multi-step plans.

Workflows without an `ai` block remain discoverable by `key`, `name`, and `description`.

## AI Assistant API

The assistant exposes internal JSON endpoints that future chat integrations can reuse:

- `GET /api/workflow-launcher/ai/catalog`
- `POST /api/workflow-launcher/ai/conversations`
- `GET /api/workflow-launcher/ai/conversations`
- `GET /api/workflow-launcher/ai/conversations/<id>`
- `POST /api/workflow-launcher/ai/conversations/<id>/messages`
- `POST /api/workflow-launcher/ai/conversations/<id>/approve`
- `POST /api/workflow-launcher/ai/conversations/<id>/create-change`
- `POST /api/workflow-launcher/ai/conversations/<id>/close-change`
- `GET /api/workflow-launcher/ai/gaps`
- `GET /api/workflow-launcher/ai/metrics`

Execution safety rules:

- The browser never talks directly to an LLM provider.
- The assistant only executes after approval.
- Every executable step must map to a synced workflow.
- Unsupported requests generate proposals and capability gaps, not execution.

### Custom Jinja Filters

Custom filter modules live under the repository's top-level `filters/` directory by default. Each module must define a `FILTERS` dictionary that maps the filter name exposed to Jinja to a callable object.

```python
# filters/string_filters.py
from scripts.lib.text_helpers import normalize_hostname


def normalize_for_dns(value):
    """Normalize a user-provided hostname for DNS-safe usage."""
    return normalize_hostname(value).replace("_", "-")


FILTERS = {
    "normalize_for_dns": normalize_for_dns,
}
```

Once synced, repo-backed filters can be used anywhere the workflow launcher renders Jinja, including action parameters, `when` expressions, input `filter.query`, and dynamic choice definitions.

```yaml
actions:
  - type: nautobot
    name: "Prepare Hostname"
    script: "scripts/create_device.py"
    with:
      hostname: "{{ inputs.hostname | normalize_for_dns }}"
```

Dependency rules for custom filters:

- Use Python standard-library modules freely.
- Repo-local helper modules are supported. Shared helpers under `scripts/lib` should be imported with absolute package imports such as `from scripts.lib.text_helpers import normalize_hostname`.
- Third-party packages must already be installed in the Nautobot runtime. If a filter imports a package that is missing, Git sync fails early with an import error.

### Action Phases

Workflows support three optional action phases:

- `pre_actions` — runs before main execution. For scheduled workflows, these run at schedule-creation time.
- `actions` — main workflow actions (backward-compatible existing key).
- `post_actions` — always run after main actions, even when main actions fail.

Within each phase, actions execute in the same order they are defined in YAML. This ordering is preserved when workflows are synced from Git and reused during runtime.

Execution behavior:

- If any `pre_actions` fail, the workflow execution is blocked.
- `post_actions` are non-blocking for final run status. If post-actions fail, the run keeps its main status and displays a warning.

### Input Types

#### String Input
```yaml
- key: hostname
  label: "Hostname"
  type: string
  required: true
  validators:
    - regex: '^[a-z]+'
      message: "Must be a-z"
```

#### Integer Input
```yaml
- key: vlan_id
  label: "VLAN ID"
  type: integer
  required: true
```

#### Boolean Input
```yaml
- key: enable_port
  label: "Enable Port"
  type: boolean
  required: false
```

#### Choice Input
```yaml
- key: device_role
  label: "Device Role"
  type: choice
  required: true
  choices:
    - value: "switch"
      label: "Switch"
    - value: "router" 
      label: "Router"
    - value: "firewall"
      label: "Firewall"
```

#### Multi-Choice Input
```yaml
- key: protocols
  label: "Protocols"
  type: multichoice
  required: false
  choices:
    - value: "bgp"
      label: "BGP"
    - value: "ospf"
      label: "OSPF"
    - value: "isis"
      label: "ISIS"
```

#### Multi-Object Input

Multi-object inputs allow users to select multiple Nautobot objects using checkboxes. Similar to the `object` input type but allows multiple selections, this is useful for bulk operations.

```yaml
- key: devices
  label: "Devices"
  type: multiobject
  model: dcim.Device
  required: true
  depends_on: ["site"]
  filter:
    query: {"site_id": "{{ inputs.site.id }}"}

- key: manufacturers
  label: "Manufacturers"
  type: multiobject
  model: dcim.Manufacturer
  required: false

- key: interfaces
  label: "Interfaces"
  type: multiobject
  model: dcim.Interface
  required: true
  depends_on: ["device"]
  filter:
    query: {"device_id": "{{ inputs.device.id }}"}
```

**Using Multi-Object Inputs in Scripts:**

Multi-object inputs are received as a list of object IDs:

```python
def run(*, devices, **kwargs):
    """Process multiple devices selected by user."""
    from nautobot.dcim.models import Device
    
    # devices is a list of IDs: [1, 2, 3, ...]
    device_objects = Device.objects.filter(id__in=devices)
    
    results = []
    for device in device_objects:
        # Perform operation on each device
        print(f"Processing {device.name}")
        results.append({"device": device.name, "status": "completed"})
    
    return {"processed": len(device_objects), "results": results}
```

**Key Differences:**
- **object**: Single selection, returns one object ID
- **multiobject**: Multiple selection with checkboxes, returns list of object IDs
- **choice**: Single selection from static list, returns one value
- **multichoice**: Multiple selection from static list with checkboxes, returns list of values

#### List Input
```yaml
- key: hostnames
  label: "Server Hostnames"
  type: list
  required: true
  validators:
    - regex: '^[a-zA-Z0-9][a-zA-Z0-9\-]{0,61}[a-zA-Z0-9]?$'
      message: "Hostname must be valid (2-63 characters, alphanumeric and hyphens, cannot start/end with hyphen)"

- key: ip_addresses
  label: "IP Addresses"
  type: list
  required: false
  validators:
    - regex: '^(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$'
      message: "Must be a valid IPv4 address (e.g., 192.168.1.1)"

- key: vlan_ids
  label: "VLAN IDs"
  type: list
  default: ["100", "200"]
  validators:
    - regex: '^(?:[1-9]|[1-9][0-9]|[1-9][0-9][0-9]|[1-3][0-9][0-9][0-9]|40[0-8][0-9]|409[0-4])$'
      message: "VLAN ID must be between 1 and 4094"
```

The list input type allows users to build a list of values by:
- Entering a value in a text field
- Clicking the "+" button to add it to the list
- Removing items by clicking the "×" button next to each item
- Each value is validated using the same regex validator system as string fields
- Duplicate values are automatically prevented
- The field stores and submits the list as a JSON array

**List Input with Conditional Visibility:**
```yaml
- key: deployment_type
  label: "Deployment Type"
  type: choice
  required: true
  choices:
    - value: "single"
      label: "Single Server"
    - value: "cluster" 
      label: "Cluster"

- key: server_hostnames
  label: "Server Hostnames"
  type: list
  required: true
  show_if:
    deployment_type: "cluster"
  validators:
    - regex: '^[a-zA-Z0-9][a-zA-Z0-9\-]{0,61}[a-zA-Z0-9]?$'
      message: "Hostname must be valid"
```

#### List Dictionary Input (`listdict`)

The `listdict` input type lets users build a table-style list of structured rows.
Each row is validated against a `columns` schema.

```yaml
- key: students
  label: "Students"
  type: listdict
  required: true
  columns:
    - key: name
      label: "Name"
      type: string
      required: true
      unique: true
      validators:
        - regex: "^[A-Za-z\\s]+$"
          message: "Name must contain only letters and spaces"
    - key: age
      label: "Age"
      type: integer
      required: true
    - key: enrolled
      label: "Enrolled"
      type: boolean
      required: false
```

Behavior:
- Users can add and remove rows dynamically.
- Users can download a CSV template with pre-populated headers derived from the column schema.
- Users can upload a CSV file to populate the table, then review and edit rows before submission.
- Field-level `required: true` means at least one row is required.
- Column-level `required: true` enforces a value in each row for that column.
- Column-level `unique: true` enforces uniqueness for that column across all rows.
- Column validators are applied per row value.
- CSV uploads match headers against column `key` and `label` values, and choice columns accept either the configured choice value or label.

Example CSV for the `students` schema above:

```csv
Name,Age,Enrolled
Alice,21,true
Bob,19,false
```

In scripts/actions, `listdict` is provided as a list of dictionaries:

```python
def run(params):
    students = params["students"]
    # Example: [{"name": "Alice", "age": 21, "enrolled": True}, ...]

    adults = [row for row in students if int(row.get("age", 0)) >= 18]
    return {
        "total_students": len(students),
        "adult_students": len(adults),
    }
```

#### Object Input (Nautobot Model References)

Object inputs allow you to reference Nautobot models and access their properties in templates:

```yaml
- key: site
  label: "Site"
  type: object
  model: "dcim.site"
  required: true

- key: device
  label: "Device" 
  type: object
  model: "dcim.device"
  required: true
  depends_on: ["site"]
  filter:
    query:
      site: "{{ inputs.site.id }}"

- key: tenant
  label: "Tenant"
  type: object
  model: "tenancy.tenant"
  required: true

- key: vrf
  label: "VRF"
  type: object
  model: "ipam.vrf"
  required: true
  depends_on: ["tenant"]
  filter:
    query:
      tenant: "{{ inputs.tenant.id }}"
```

**Object Property Access**: When using object inputs in templates, you can access object properties directly:

```yaml
actions:
  - type: awx
    name: "configure_device"
    template_id: 123
    extra_vars:
      # Access object properties using dot notation
      device_name: "{{ inputs.device.name }}"
      device_id: "{{ inputs.device.id }}"
      site_name: "{{ inputs.site.name }}"
      site_slug: "{{ inputs.site.slug }}"
      tenant_name: "{{ inputs.tenant.name }}"
      # String operations work on object properties
      tenant_short: "{{ inputs.tenant.name.split(':')[-1] }}"
      # Conditional access with safe defaults
      description: "{{ inputs.device.description or 'No description' }}"
```

**Available Properties**: Object inputs provide access to common model properties:
- `id` - Primary key of the object
- `name` - Name field (if available)
- `slug` - Slug field (if available)  
- `description` - Description field (if available)
- Plus any other attributes available on the original object

**Dependent Object Filtering**: Use object properties in filter queries for dependent fields:

```yaml
- key: controller
  label: "Controller"
  type: object
  model: "dcim.controller"
  required: true

- key: tenant
  label: "Tenant"
  type: object
  model: "tenancy.tenant"
  required: true
  depends_on: ["controller"]
  filter:
    query:
      # Filter tenants by controller name
      tenant_group__name: "{{ inputs.controller.name }}"

- key: vrf
  label: "VRF"
  type: object
  model: "ipam.vrf"
  required: true
  depends_on: ["tenant"]
  filter:
    query:
      # Use tenant ID for filtering
      tenant_id: "{{ inputs.tenant.id }}"
```

#### Input Validation

Add regex validation to string inputs:

```yaml
- key: hostname
  label: "Hostname"
  type: string
  required: true
  validators:
    - regex: "^[a-zA-Z0-9-]+$"
      message: "Hostname must contain only letters, numbers, and hyphens"
      flags: ["IGNORECASE"]

- key: ip_address
  label: "IP Address"
  type: string
  required: false
  validators:
    - regex: "^(?:[0-9]{1,3}\\.){3}[0-9]{1,3}$"
      message: "Must be a valid IPv4 address"

- key: vlan_id
  label: "VLAN ID"
  type: integer
  required: true
  validators:
    - regex: "^([1-9]|[1-9][0-9]|[1-9][0-9][0-9]|[1-3][0-9][0-9][0-9]|40[0-8][0-9]|409[0-4])$"
      message: "VLAN ID must be between 1 and 4094"
```

#### Conditional Field Visibility

Control field visibility based on other field values using `show_if`:

```yaml
inputs:
  - key: deployment_type
    label: "Deployment Type"
    type: choice
    required: true
    choices:
      - value: "physical"
        label: "Physical Device"
      - value: "virtual"
        label: "Virtual Machine"

  - key: request_type
    label: "Request Type"
    type: choice
    required: true
    choices:
      - value: "permanent"
        label: "Permanent Assignment"
      - value: "temporary"
        label: "Temporary Assignment"
      - value: "move"
        label: "Move Device"
      - value: "reservation"
        label: "Reserve for Future"

  - key: enable_routing
    label: "Enable Routing"
    type: boolean
    required: false

  - key: gateway_address
    label: "Gateway Address"
    type: string
    required: true
    depends_on: ["enable_routing"]
    show_if:
      enable_routing: true  # Field appears when checkbox is checked
    validators:
      - regex: '^(?:(?:25[0-5]|2[0-4]\\d|1?\\d?\\d)\\.){3}(?:25[0-5]|2[0-4]\\d|1?\\d?\\d)(?:/(?:3[0-2]|[12]?\\d))?$'
        message: "Enter IPv4 with CIDR notation (e.g., 192.0.2.1/24)."

  - key: rack_unit
    label: "Rack Unit"
    type: integer
    required: true
    show_if:
      deployment_type: "physical"  # Only show for physical deployments

  - key: hypervisor
    label: "Hypervisor"
    type: object
    model: "virtualization.cluster"
    required: true
    show_if:
      deployment_type: "virtual"  # Only show for virtual deployments

  - key: cpu_cores
    label: "CPU Cores"
    type: integer
    required: false
    show_if:
      deployment_type: "virtual"
```

**Multiple Condition Visibility**: For complex scenarios, you can specify multiple conditions that ALL must be satisfied:

```yaml
inputs:
  - key: location_method
    label: "Location Selection Method"
    type: choice
    required: true
    choices:
      - value: "manual"
        label: "Manual Selection"
      - value: "auto"
        label: "Automatic Assignment"

  - key: request_type
    label: "Request Type"
    type: choice
    required: true
    choices:
      - value: "permanent"
        label: "Permanent"
      - value: "temporary"
        label: "Temporary"
      - value: "move"
        label: "Move"
      - value: "reservation"
        label: "Reservation"

  - key: rack
    label: "Rack"
    type: object
    model: "dcim.rack"
    required: true
    depends_on: ["location"]
    show_if:
      # ALL conditions must be satisfied for field to be visible
      location_method: manual
      request_type:
        - permanent
        - move
        - reservation  # Field shows when request_type is any of these values
    filter:
      query:
        location: "{{ inputs.location.id }}"

  - key: maintenance_window
    label: "Maintenance Window"
    type: choice
    required: false
    show_if:
      # Multiple conditions with different types
      deployment_type: "physical"
      request_type: "permanent"
      enable_monitoring: true
    choices:
      - value: "business_hours"
        label: "Business Hours"
      - value: "after_hours"
        label: "After Hours"
```

**Boolean Field Visibility**: For boolean fields, use `true`/`false` values in show_if conditions:

```yaml
- key: create_backup
  label: "Create Backup"
  type: boolean
  required: false

- key: backup_location
  label: "Backup Location"
  type: string
  required: true
  show_if:
    create_backup: true  # Shows when backup checkbox is checked
```

### Actions

#### Nautobot Script Action
```yaml
- type: nautobot
  name: "Create Interface"
  script: "scripts/create_interface.py"
  when: "{{ inputs.create_interface }}"  # optional condition
  with:
    # Pass object IDs and properties to scripts
    device_id: "{{ inputs.device.id }}"
    device_name: "{{ inputs.device.name }}"
    site_id: "{{ inputs.site.id }}"
    site_name: "{{ inputs.site.name }}"
    interface_name: "{{ inputs.interface_name }}"
    description: "Created by workflow for {{ inputs.device.name }} at {{ inputs.site.name }}"
```

#### AWX/Ansible Action
```yaml
- type: awx
  name: "Configure Device"
  template_id: 123
  when: "{{ inputs.configure_device }}"  # optional condition
  extra_vars:
    # Access object properties directly
    target_host: "{{ inputs.device.name }}"
    device_id: "{{ inputs.device.id }}"
    site_name: "{{ inputs.site.name }}"
    site_slug: "{{ inputs.site.slug }}"
    # String manipulation on object properties
    tenant_name: "{{ inputs.tenant.name.split(':')[-1] }}"
    # Use object properties in conditionals
    environment: "{% if 'prod' in inputs.site.name.lower() %}production{% else %}development{% endif %}"
    # Safe property access with defaults
    device_description: "{{ inputs.device.description or 'No description available' }}"
    interface_name: "{{ inputs.interface_name }}"
    vlan_id: "{{ inputs.vlan_id }}"
  monitor: true  # default: true
```

### Workflow Management

#### Enabling and Disabling Workflows

Workflows can be disabled to prevent execution while preserving their run history. This is useful for:
- Temporarily disabling workflows during maintenance
- Removing workflows from Git while keeping historical data
- Testing new versions without deleting the old ones

**Explicit Disabling in YAML:**
```yaml
key: maintenance-workflow
name: "Maintenance Workflow"
description: "Currently disabled for updates"
enabled: false  # This workflow won't appear in the launcher
```

**Automatic Disabling:**
- When workflows are removed from Git repositories, they are automatically disabled during sync (if delete option is enabled)
- Disabled workflows preserve all run history, actions, and inputs
- Only enabled workflows appear in the workflow launcher UI
- Disabled workflows can still be viewed in the admin interface

**Re-enabling Workflows:**
- Add the workflow back to Git to automatically re-enable it
- Or set `enabled: true` in the YAML file
- Or manually enable via Django admin interface

### Advanced Features

#### Object Property Access in Templates

When using object inputs in workflow templates, you have full access to object properties using dot notation. This powerful feature allows you to access any attribute of Nautobot objects in your template expressions.

**Basic Property Access:**
```yaml
actions:
  - type: awx
    name: "configure_device"
    template_id: 100
    extra_vars:
      # Basic object properties
      device_id: "{{ inputs.device.id }}"           # Primary key
      device_name: "{{ inputs.device.name }}"       # Name field
      device_slug: "{{ inputs.device.slug }}"       # Slug field (if available)
      description: "{{ inputs.device.description }}" # Description field

      # Related object properties
      site_name: "{{ inputs.device.site.name }}"
      tenant_name: "{{ inputs.device.tenant.name }}"
      device_type: "{{ inputs.device.device_type.model }}"
```

**String Operations on Properties:**
```yaml
extra_vars:
  # String manipulation
  tenant_short: "{{ inputs.tenant.name.split(':')[-1] }}"
  site_upper: "{{ inputs.site.name.upper() }}"
  device_serial: "{{ inputs.device.serial.replace('-', '') }}"
  
  # Conditional string formatting
  environment: "{% if 'prod' in inputs.site.name.lower() %}production{% else %}development{% endif %}"
```

**Safe Property Access:**
```yaml
extra_vars:
  # Use defaults for potentially None values
  device_desc: "{{ inputs.device.description or 'No description available' }}"
  primary_ip: "{{ inputs.device.primary_ip4.address if inputs.device.primary_ip4 else '' }}"
  rack_position: "{{ inputs.device.position or 'Not racked' }}"
  
  # Check if property exists before using
  has_location: "{{ inputs.device.location is not none }}"
  location_name: "{{ inputs.device.location.name if inputs.device.location else 'No location' }}"
```

**Using Properties in Conditional Logic:**
```yaml
# In dependent field filters
- key: interfaces
  label: "Interfaces"  
  type: object
  model: "dcim.interface"
  depends_on: ["device"]
  filter:
    query:
      device: "{{ inputs.device.id }}"
      # Filter based on device role
      device__device_role__slug: "{{ inputs.device.device_role.slug }}"
      # Only enabled interfaces
      enabled: true

# In action conditions
- type: awx
  name: "configure_production"
  template_id: 200
  # Only run for production sites
  when: "{{ 'prod' in inputs.site.name.lower() or inputs.site.slug.startswith('prod-') }}"
  
- type: nautobot  
  name: "assign_management_ip"
  script: "scripts/assign_mgmt_ip.py"
  # Only run if device doesn't have primary IP
  when: "{{ not inputs.device.primary_ip4 }}"
```

**Complex Property Navigation:**
```yaml
extra_vars:
  # Navigate through related objects
  rack_name: "{{ inputs.device.rack.name if inputs.device.rack else 'No rack' }}"
  location_hierarchy: "{{ inputs.device.location.parent.name if inputs.device.location and inputs.device.location.parent else 'Top level' }}"
  
  # Access custom fields (if available)
  cost_center: "{{ inputs.device.custom_field_data.cost_center or 'Unknown' }}"
  maintenance_window: "{{ inputs.device.custom_field_data.maintenance_window or 'Standard' }}"
  
  # Network-specific properties
  mgmt_interface: "{{ inputs.device.interfaces.filter(mgmt_only=True).first().name if inputs.device.interfaces.filter(mgmt_only=True).exists() else 'No mgmt interface' }}"
```

This enhanced object property access enables sophisticated workflows that can make intelligent decisions and pass rich context to both Nautobot scripts and AWX playbooks.

#### Template Includes

Workflow Launcher supports template includes to promote reusability and reduce duplication across workflow definitions. You can extract common input definitions into separate template files and include them in multiple workflows using the `{% include %}` directive.

**Template File Structure:**

Template files should be placed in a `templates/` directory within your Git repository and contain properly indented YAML fragments:

```
repository/
├── workflows/
│   ├── server_deployment.yaml
│   ├── network_config.yaml
│   └── templates/
│       └── inputs/
│           ├── change_management.yaml
│           └── location_selection.yaml
```

**Creating Template Files:**

Template files contain YAML fragments that will be inserted into workflows. They must be properly indented to match the context where they'll be included:

```yaml
# templates/inputs/change_management.yaml
  - key: change_id
    label: "Service Now Ticket#"
    type: string
    required: true
    validators:
      - regex: "^CHG|RITM[0-9]{6,20}$"
        message: "Enter the Service Now Change ID or RITM ID (e.g., CHG1234567, RITM1234567)."

  - key: priority
    label: "Change Priority"
    type: choice
    required: true
    choices:
      - value: "low"
        label: "Low"
      - value: "medium"
        label: "Medium"
      - value: "high"
        label: "High"
      - value: "critical"
        label: "Critical"
```

**Using Includes in Workflows:**

Include template files in your workflow definitions using the `{% include %}` directive:

```yaml
# workflows/server_deployment.yaml
version: 1
key: server_deployment
name: "Server Deployment"
description: "Deploy a new server with change management tracking"
category: "Infrastructure"
manual_duration_minutes: 30

inputs:
  # Include common change management fields
  {% include "templates/inputs/change_management.yaml" %}

  # Workflow-specific inputs
  - key: server_name
    label: "Server Name"
    type: string
    required: true
    validators:
      - regex: "^[a-zA-Z0-9.-]{1,64}$"
        message: "Use letters, digits, dots, and hyphens (max 64 chars)."

  - key: environment
    label: "Environment"
    type: choice
    choices:
      - value: "dev"
        label: "Development"
      - value: "staging"
        label: "Staging"
      - value: "prod"
        label: "Production"

actions:
  - type: nautobot
    name: "Deploy Server"
    script: "scripts/deploy_server.py"
    with:
      change_id: "{{ inputs.change_id }}"
      priority: "{{ inputs.priority }}"
      server_name: "{{ inputs.server_name }}"
      environment: "{{ inputs.environment }}"
```

**Multiple Includes:**

You can use multiple includes in a single workflow:

```yaml
inputs:
  # Standard change management inputs
  {% include "templates/inputs/change_management.yaml" %}
  
  # Common location selection inputs
  {% include "templates/inputs/location_selection.yaml" %}
  
  # Workflow-specific inputs
  - key: device_type
    label: "Device Type"
    type: object
    model: dcim.DeviceType
    required: true
```

**Benefits of Template Includes:**

- **Consistency**: Ensure standard fields like change management IDs use the same validation across all workflows
- **Maintainability**: Update common field definitions in one place
- **Reusability**: Share input definitions across multiple workflows
- **Organization**: Keep workflow files focused on their specific logic
- **Standardization**: Enforce organizational standards for common inputs

**Important Notes:**

- Template paths are relative to the workflow file location
- Template files must use proper YAML indentation for the context where they'll be included
- Include processing happens during workflow sync from Git repositories
- Templates can include any valid YAML content that would be appropriate for the inclusion context

#### Workflow Organization

Workflows are automatically organized by category in the launcher interface:

```yaml
key: interface-provisioning
name: "Interface Provisioning"
description: "Automated interface configuration"
category: "Network Provisioning"  # Groups similar workflows together
manual_duration_minutes: 30

```

Categories provide several benefits:
- **Global Search**: Workflows are searchable from anywhere in Nautobot by name, description, or even category
- **Visual Organization**: Workflows are grouped and displayed by category
- **Filtering**: Users can filter to show only workflows from specific categories  
- **Navigation**: Category-specific views reduce clutter in large deployments
- **Reporting**: Statistics and reports can be broken down by category

#### Dependent Fields
Create cascading dropdowns where child fields depend on parent selections, with full access to parent object properties:

```yaml
inputs:
  - key: controller
    label: "Controller"
    type: object
    model: "dcim.controller"
    required: true

  - key: tenant
    label: "Tenant"
    type: object
    model: "tenancy.tenant"
    required: true
    depends_on: ["controller"]
    filter:
      query:
        # Use controller name to filter tenants by tenant group
        tenant_group__name: "{{ inputs.controller.name }}"

  - key: site
    label: "Site"
    type: object
    model: "dcim.site" 
    required: true
    depends_on: ["tenant"]
    filter:
      query:
        # Use tenant ID for exact matching
        tenant: "{{ inputs.tenant.id }}"

  - key: device
    label: "Device"
    type: object
    model: "dcim.device"
    required: true
    depends_on: ["site"]
    filter:
      query:
        site: "{{ inputs.site.id }}"

  - key: vrf
    label: "VRF"
    type: object
    model: "ipam.vrf"
    required: true
    depends_on: ["tenant"]
    filter:
      query:
        # Filter VRFs by tenant using direct ID reference
        tenant_id: "{{ inputs.tenant.id }}"
```

**Enhanced Object Property Access**: Dependent field filters can now access any property of parent objects:

```yaml
# Complex filtering using object properties
- key: interface
  label: "Interface"
  type: object
  model: "dcim.interface"
  required: true
  depends_on: ["device"]
  filter:
    query:
      device: "{{ inputs.device.id }}"
      # Filter by device role using object property access
      device__device_role__slug: "{{ inputs.device.device_role.slug }}"
      # Only show enabled interfaces
      enabled: true
```

#### Conditional Actions
Use Jinja2 expressions to conditionally execute actions based on inputs or previous action results:

```yaml
actions:
  - type: nautobot
    name: check_device_status
    script: "scripts/check_device.py"
    with:
      device_id: "{{ inputs.device }}"
    
  - type: nautobot
    name: create_interface
    script: "scripts/create_interface.py"
    when: "{{ inputs.action_type == 'create' }}"
    with:
      device_id: "{{ inputs.device }}"
      interface_name: "{{ inputs.interface_name }}"
    
  - type: nautobot
    name: delete_interface
    script: "scripts/delete_interface.py"
    when: "{{ inputs.action_type == 'delete' }}"
    with:
      device_id: "{{ inputs.device }}"
      interface_name: "{{ inputs.interface_name }}"

  - type: awx
    name: configure_production
    template_id: 100
    when: "{{ inputs.environment == 'production' and results.check_device_status.return_value.status == 'online' }}"
    extra_vars:
      device: "{{ inputs.device.name }}"
    
  - type: awx
    name: configure_development 
    template_id: 101
    when: "{{ inputs.environment == 'development' and results.create_interface.status == 'success' }}"
    extra_vars:
      device: "{{ inputs.device.name }}"
```

#### Action Return Values

Actions can return values that are accessible to subsequent actions in conditional expressions and parameter substitution:

```yaml
actions:
  - type: nautobot
    name: "validate_device"
    script: "scripts/validate_device.py"
    with:
      device_id: "{{ inputs.device }}"
  
  - type: nautobot
    name: "create_interface"
    script: "scripts/create_interface.py"
    when: "{{ results.validate_device.return_value.is_valid }}"
    with:
      device_id: "{{ inputs.device }}"
      interface_name: "{{ inputs.interface_name }}"
      interface_type: "{{ results.validate_device.return_value.recommended_type }}"
  
  - type: awx
    name: "configure_interface"
    template_id: 123
    when: "{{ results['create_interface'].status.success }}"
    extra_vars:
      interface_id: "{{ results.create_interface.return_value.interface_id }}"
      device_name: "{{ inputs.device.name }}"
```

Available context variables:
- `results.<action_name>` - Return value from the named action
- `last` - Return value from the previous action (convenience accessor)

#### IP Address Operations

The workflow system includes custom Jinja2 filters for IP address operations using Python's `ipaddress` library:

```yaml
actions:
  - type: nautobot
    name: "subnet_calculation"
    script: "scripts/create_subnets.py"
    with:
      base_network: "{{ inputs.network }}"
      # Get /24 subnets from a /22 network
      subnets: "{{ inputs.network | subnets(new_prefix=24) }}"
      # Get network address
      network_addr: "{{ inputs.network | network_address }}"
      # Check if IP is in network
      ip_in_net: "{{ inputs.ip_address | ip_in_network(inputs.network) }}"

  - type: awx
    name: "configure_routing"
    template_id: 100
    # Only run if IP is private and in the management network
    when: "{{ inputs.ip_address | is_private and inputs.ip_address | ip_in_network('10.0.0.0/8') }}"
    extra_vars:
      target_ip: "{{ inputs.ip_address }}"
      is_ipv4: "{{ (inputs.ip_address | ip_version) == 4 }}"
      # Use result from previous subnet_calculation action
      network_info: "{{ results.subnet_calculation }}"
      # Get first usable host in subnet
      gateway: "{{ (inputs.network | hosts)[0] }}"
```

**Available IP filters:**

**Basic Operations:**
- `ip_address` - Convert string to IP address object
- `ip_network` - Convert string to IP network object
- `ip_interface` - Convert string to IP interface object

**Network Properties:**
- `network_address` - Get network address (e.g., `192.168.1.0/24 | network_address` → `192.168.1.0`)
- `broadcast_address` - Get broadcast address (e.g., `192.168.1.0/24 | broadcast_address` → `192.168.1.255`)
- `netmask` - Get subnet mask (e.g., `192.168.1.0/24 | netmask` → `255.255.255.0`)
- `hostmask` - Get host mask (e.g., `192.168.1.0/24 | hostmask` → `0.0.0.255`)
- `prefixlen` - Get prefix length (e.g., `192.168.1.0/24 | prefixlen` → `24`)
- `num_addresses` - Get number of addresses in network

**Subnet Operations:**
- `subnets(prefixlen_diff=1)` - Get subnets (e.g., `10.0.0.0/22 | subnets(new_prefix=24)`)
- `supernet(prefixlen_diff=1)` - Get supernet (e.g., `192.168.1.0/24 | supernet(new_prefix=22)`)
- `hosts` - Get list of host addresses (excludes network and broadcast)

**Network Relationships:**
- `subnet_of(other)` - Check if network is subnet of another
- `supernet_of(other)` - Check if network is supernet of another  
- `overlaps(other)` - Check if networks overlap
- `address_exclude(other)` - Get networks in first but not second
- `ip_in_network(network)` - Check if IP address is in network

**Address Classification:**
- `is_private` - Check if address/network is private (RFC 1918)
- `is_global` - Check if address/network is global
- `is_multicast` - Check if address/network is multicast
- `is_reserved` - Check if address/network is reserved
- `ip_version` - Get IP version (4 or 6)

**Examples:**

```yaml
# Calculate subnets for VLAN assignment
vlans: "{{ '10.0.0.0/16' | subnets(new_prefix=24) | list | length }}"

# Get management IP (first host in subnet)  
mgmt_ip: "{{ (inputs.mgmt_network | hosts)[0] }}"

# Check network relationships
is_subnet: "{{ inputs.vlan_network | subnet_of('10.0.0.0/8') }}"

# Conditional based on IP properties
when: "{{ inputs.ip | is_private and inputs.ip | ip_version == 4 }}"

# Network calculations
gateway: "{{ inputs.network | network_address }}"
broadcast: "{{ inputs.network | broadcast_address }}"
```

#### Dynamic Choice Fields with Jinja Templates

Choice fields can use Jinja templates to dynamically generate options based on other input values. This is particularly useful for creating dependent dropdowns that filter available choices based on previously selected inputs.

**Example: Dynamic Rack Position Selection**

```yaml
inputs:
  - key: rack
    label: "Rack"
    type: object
    model: dcim.rack
    required: true

  - key: device_type
    label: "Device Type"
    type: object
    model: dcim.devicetype
    required: true

  - key: position
    label: "Rack Position"
    type: choice
    required: true
    depends_on:
      - rack
      - device_type
    choices: |
      [
        {% for unit in inputs.rack | available_rack_units(inputs.device_type) %}
        {"value": "{{ unit }}", "label": "RU-{{ unit }}"}{% if not loop.last %},{% endif %}
        {% endfor %}
      ]
```

**Available DCIM Filters:**

- `available_rack_units(device_type=None)` - Get available rack units, optionally filtered by device height
    - When `device_type` is provided, filters positions to only include those where the device will fit
    - Considers the device type's `u_height` property for multi-unit devices
    - Returns positions sorted from top to bottom (highest RU first)

**Usage Patterns:**

```yaml
# All available positions in rack
simple_positions: "{{ inputs.rack | available_rack_units }}"

# Positions that can fit a specific device type
filtered_positions: "{{ inputs.rack | available_rack_units(inputs.device_type) }}"

# Use in choice field for dynamic dropdown
choices: |
  [
    {% for unit in inputs.rack | available_rack_units(inputs.device_type) %}
    {"value": "{{ unit }}", "label": "Position {{ unit }}"}{% if not loop.last %},{% endif %}
    {% endfor %}
  ]

# Custom labeling with additional context
choices: |
  [
    {% set available = inputs.rack | available_rack_units(inputs.device_type) %}
    {% for unit in available %}
    {"value": "{{ unit }}", "label": "RU-{{ unit }} ({{ available|length - loop.index0 }} remaining)"}{% if not loop.last %},{% endif %}
    {% endfor %}
  ]
```

**Key Features:**

- **Dependency-aware**: Choice options update automatically when dependent fields change
- **Height-aware filtering**: Considers device type height requirements (multi-RU devices)
- **Real-time validation**: Only shows positions where the selected device will actually fit
- **Flexible templating**: Full Jinja2 support for custom choice formatting and logic

mask: "{{ inputs.network | netmask }}"
total_ips: "{{ inputs.network | num_addresses }}"
```

## Script Development

### Nautobot Script Structure

Scripts referenced in workflows must define a `run(params)` function and can optionally return values:

```python
def run(params):
    """
    Main entry point for workflow script.
    
    Args:
        params (dict): Parameters passed from workflow with values
        
    Returns:
        dict: Optional return value accessible to subsequent actions
    """
    from nautobot.dcim.models import Device, Interface
    from nautobot.ipam.models import IPAddress
    
    # Access parameters
    device_id = params["device"]
    interface_name = params["interface_name"]
    ip_address = params.get("ip_address")
    
    # Perform operations
    device = Device.objects.get(pk=device_id)
    interface, created = Interface.objects.get_or_create(
        device=device,
        name=interface_name
    )
    
    if ip_address:
        ip, created = IPAddress.objects.get_or_create(address=ip_address)
        ip.assigned_object = interface
        ip.save()
    
    print(f"Created interface {interface_name} on {device.name}")
    
    # Return values accessible to subsequent actions
    return {
        "interface_id": interface.id,
        "interface_created": created,
        "ip_assigned": bool(ip_address),
        "device_name": device.name
    }
```

### Modular Script Development

The workflow launcher supports modular Python development by storing all Python files from your Git repository. This allows you to create reusable helper modules and organize your code effectively.

#### Repository Structure

```
your-workflow-repo/
├── workflows/                  # Workflow YAML definitions
│   ├── device-provisioning.yaml
│   └── interface-setup.yaml
├── filters/                    # Repo-backed Jinja filter modules
│   └── string_filters.py
├── scripts/                   # Action scripts  
│   ├── create_device.py
│   ├── create_interface.py
│   ├── utils.py               # Helper module
│   ├── lib/                   # Shared libraries for scripts and filters
│   │   ├── network_utils.py
│   │   └── validation.py
│   └── common/                # Common utilities
│       └── logging.py
```

#### Using Helper Modules

Scripts can import and use helper modules using standard Python imports:

```python
# scripts/create_interface.py
from scripts.utils import validate_interface_name, get_next_available_ip
from scripts.lib.network_utils import calculate_subnet
from common.logging import log_action

def run(params):
    """Create interface with validation and IP assignment."""
    
    # Use helper functions
    if not validate_interface_name(params["interface_name"]):
        raise ValueError("Invalid interface name format")
    
    device_id = params["device"]
    interface_name = params["interface_name"]
    
    # Use shared network utilities
    if params.get("auto_assign_ip"):
        subnet = calculate_subnet(params["network"])
        ip_address = get_next_available_ip(subnet)
    else:
        ip_address = params.get("ip_address")
    
    # Perform the actual work
    from nautobot.dcim.models import Device, Interface
    device = Device.objects.get(pk=device_id)
    interface, created = Interface.objects.get_or_create(
        device=device,
        name=interface_name
    )
    
    # Log the action
    log_action(f"Created interface {interface_name} on {device.name}")
    
    return {
        "interface_id": interface.id,
        "created": created,
        "ip_address": ip_address
    }
```

```python
# scripts/utils.py
import re
from nautobot.ipam.models import IPAddress, Prefix

def validate_interface_name(name):
    """Validate interface name format."""
    patterns = [
        r'^GigabitEthernet\d+/\d+/\d+$',
        r'^FastEthernet\d+/\d+$',
        r'^eth\d+$'
    ]
    return any(re.match(pattern, name) for pattern in patterns)

def get_next_available_ip(prefix):
    """Get next available IP address in prefix."""
    prefix_obj = Prefix.objects.get(prefix=prefix)
    used_ips = set(IPAddress.objects.filter(
        address__net_contained=prefix_obj.prefix
    ).values_list('address__ip', flat=True))
    
    for ip in prefix_obj.prefix.hosts():
        if str(ip) not in used_ips:
            return str(ip)
    return None
```

Custom filter modules follow the same import pattern. If shared code lives under `scripts/lib`, import it directly from the materialized package path:

```python
# filters/string_filters.py
from scripts.lib.text_helpers import normalize_hostname


def hostname_is_valid(value):
    """Return True when the provided hostname passes shared normalization rules."""
    return bool(normalize_hostname(value))


FILTERS = {
    "hostname_is_valid": hostname_is_valid,
}
```

#### Benefits of Modular Development

1. **Code Reuse**: Share common functionality across multiple workflow scripts
2. **Organization**: Keep related functions grouped in logical modules
3. **Testing**: Easier to unit test individual components
4. **Maintenance**: Changes to shared logic only need to be made in one place
5. **Readability**: Workflow scripts focus on business logic rather than implementation details

### Security Considerations

- Only trusted repositories should be used for workflow definitions
- Scripts can import any Python module available to Nautobot

## AWX Integration

### Setup External Integration

1. Create an **External Integration** in Nautobot:

    - **Name**: `AWX` (or configure custom name in plugin settings)
    - **URL**: Your AWX instance URL
    - **Verify SSL**: Configure as needed

2. Create a **Secrets Group** with:

    - **Secret**: Generic -> Username (AWX User)
    - **Secret**: Generic -> Password (AWX Password)

3. Associate the Secrets Group with your External Integration

### AWX Job Monitoring

- Jobs are automatically monitored when `monitor: true` (default)
- Job status updates are reflected in workflow run logs
- Direct links to AWX jobs are provided in the run details

## Examples

### Complete Workflow Example

```yaml
key: interface-provisioning
name: "Advanced Interface Provisioning Workflow"
description: "Provision network interfaces with validation, IP assignment, and configuration"
category: "Network Provisioning"
manual_duration_minutes: 30

inputs:
  - key: site
    label: "Site"
    type: object
    model: "dcim.site"
    required: true

  - key: device
    label: "Device"
    type: object
    model: "dcim.device"
    required: true
    depends_on: ["site"]
    filter:
      query:
        site: "{{ inputs.site.id }}"

  - key: interface_name
    label: "Interface Name"
    type: string
    required: true

  - key: ip_assignment_mode
    label: "IP Assignment Mode"
    type: choice
    required: true
    choices:
      - value: "manual"
        label: "Manual Assignment"
      - value: "auto"
        label: "Automatic Assignment"
      - value: "none"
        label: "No IP Assignment"

  - key: ip_address
    label: "IP Address"
    type: string
    required: false
    show_if:
      ip_assignment_mode: "manual"

  - key: network_prefix
    label: "Network Prefix"
    type: object
    model: "ipam.prefix"
    required: false
    depends_on: ["site"]
    show_if:
      ip_assignment_mode: "auto"
    filter:
      query:
        site: "{{ inputs.site.id }}"

  - key: configure_routing
    label: "Configure Routing"
    type: boolean
    required: false

  - key: routing_protocol
    label: "Routing Protocol"
    type: choice
    required: false
    show_if:
      configure_routing: true
    choices:
      - value: "ospf"
        label: "OSPF"
      - value: "bgp"
        label: "BGP"

actions:
  - type: nautobot
    name: "validate_prerequisites"
    script: "scripts/validate_device.py"
    with:
      device_id: "{{ inputs.device }}"
      interface_name: "{{ inputs.interface_name }}"

  - type: nautobot
    name: "create_interface"
    script: "scripts/create_interface.py"
    when: "{{ results['validate_prerequisites'].result.is_valid }}"
    with:
      device_id: "{{ inputs.device }}"
      interface_name: "{{ inputs.interface_name }}"
      ip_assignment_mode: "{{ inputs.ip_assignment_mode }}"
      ip_address: "{{ inputs.ip_address }}"
      network_prefix: "{{ inputs.network_prefix }}"

  - type: awx
    name: "configure_interface_basic"
    template_id: 50
    when: "{{ results['create_interface'].status =='success' }}"
    extra_vars:
      # Use object properties in AWX templates
      target_device: "{{ inputs.device.name }}"
      device_mgmt_ip: "{{ inputs.device.primary_ip4.address if inputs.device.primary_ip4 else '' }}"
      site_code: "{{ inputs.site.slug.upper() }}"
      interface_name: "{{ inputs.interface_name }}"
      ip_address: "{{ results['create_interface'].return_value.ip_address }}"
      interface_id: "{{ results['create_interface'].return_value.interface_id }}"

  - type: awx
    name: "configure_ospf_routing"
    template_id: 51
    when: "{{ inputs.configure_routing and inputs.routing_protocol == 'ospf' and results['configure_interface_basic'].status =='success'  }}"
    extra_vars:
      target_device: "{{ inputs.device.name }}"
      interface_name: "{{ inputs.interface_name }}"
      network_area: "{{ results['create_interface'].return_value.result.ospf_area }}"

  - type: awx
    name: "configure_bgp_routing"
    template_id: 52
    when: "{{ inputs.configure_routing and inputs.routing_protocol == 'bgp' and results['configure_interface_basic'].status =='success'  }}"
    extra_vars:
      target_device: "{{ inputs.device.name }}"
      interface_name: "{{ inputs.interface_name }}"
      peer_ip: "{{ results['create_interface'].return_value.result.bgp_peer }}"

  - type: nautobot
    name: "finalize_configuration"
    script: "scripts/finalize_interface.py"
    when: "{{ results['create_interface'].success }}"
    with:
      interface_id: "{{ results['create_interface'].result.interface_id }}"
      routing_configured: "{{ inputs.configure_routing }}"
      validation_results: "{{ results['validate_prerequisites'].result }}"
```

### Example Script with Return Values

```python
# scripts/create_interface.py
from .utils import validate_interface_name, get_next_available_ip
from lib.network_utils import determine_ospf_area, find_bgp_peer
from common.logging import log_action

def run(params):
    """Create interface with comprehensive IP and routing setup."""
    
    # Import inside function to avoid import issues
    from nautobot.dcim.models import Device, Interface
    from nautobot.ipam.models import IPAddress, Prefix
    
    device_id = params["device_id"]
    interface_name = params["interface_name"]
    ip_assignment_mode = params["ip_assignment_mode"]
    
    device = Device.objects.get(pk=device_id)
    
    # Create the interface
    interface, created = Interface.objects.get_or_create(
        device=device,
        name=interface_name,
        defaults={'type': '1000base-t', 'enabled': True}
    )
    
    ip_address = None
    ospf_area = None
    bgp_peer = None
    
    # Handle IP assignment
    if ip_assignment_mode == "manual":
        ip_address = params["ip_address"]
        if ip_address:
            ip_obj, _ = IPAddress.objects.get_or_create(address=ip_address)
            ip_obj.assigned_object = interface
            ip_obj.save()
            
    elif ip_assignment_mode == "auto":
        prefix_id = params["network_prefix"]
        if prefix_id:
            prefix = Prefix.objects.get(pk=prefix_id)
            ip_address = get_next_available_ip(str(prefix.prefix))
            if ip_address:
                ip_obj, _ = IPAddress.objects.get_or_create(address=ip_address)
                ip_obj.assigned_object = interface
                ip_obj.save()
    
    # Determine routing parameters for potential use
    if ip_address:
        ospf_area = determine_ospf_area(device, ip_address)
        bgp_peer = find_bgp_peer(device, ip_address)
    
    log_action(f"Created interface {interface_name} on {device.name} with IP {ip_address}")
    
    # Return comprehensive results for subsequent actions
    return {
        "interface_id": interface.id,
        "interface_created": created,
        "interface_name": interface_name,
        "device_name": device.name,
        "ip_address": ip_address,
        "ip_assigned": bool(ip_address),
        "ospf_area": ospf_area,
        "bgp_peer": bgp_peer,
        "ready_for_configuration": bool(ip_address and interface.enabled)
    }
```

## Troubleshooting

### Git Sync Issues
- Ensure the Git repository is accessible and contains workflow YAML files
- Check that the `workflow_dir` setting matches your repository structure
- Verify YAML syntax is valid
- Helper Python files are automatically stored during sync - check sync logs for helper file storage messages

### Script Execution Errors
- Scripts must define a `run(params)` function
- Scripts can optionally return a dictionary that becomes accessible to subsequent actions
- Check script syntax and imports - helper modules are available via relative imports
- Review workflow run logs for detailed error messages
- Verify that imported helper modules exist in the repository and were synced successfully

### Module Import Issues
- Helper modules are automatically stored and made available during script execution
- Use relative imports like `from .utils import function_name` for scripts in the same directory
- Use absolute imports like `from scripts.lib.module import function_name` for modules in other directories
- Check that the Git repository sync completed successfully and stored helper files

### Conditional Logic Issues
- Action conditions have access to `inputs` and `actions` context variables
- Use `results.<action_name>.status` to check if previous actions succeeded
- Use `results.<action_name>.return_value` to access return values from Nautobot scripts
- AWX actions don't provide result data, only success/failure status
- Validate Jinja2 syntax in `when` conditions

### AWX Integration Issues
- Verify External Integration configuration
- Check AWX API token permissions
- Ensure AWX template IDs exist and are accessible
- AWX job monitoring requires `monitor: true` (default) and valid External Integration setup
