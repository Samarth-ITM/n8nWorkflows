Selectively import workflows between instances using the n8n API and forms

https://n8nworkflows.xyz/workflows/selectively-import-workflows-between-instances-using-the-n8n-api-and-forms-13425


# Selectively import workflows between instances using the n8n API and forms

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Selectively import workflows between instances using the n8n API and forms

**Purpose:**  
This workflow selectively migrates workflows from one n8n instance (“source”) to another (“target”) without bulk export/import. It discovers workflows, lets a user choose specific ones via an n8n Form, fetches each selected workflow JSON, strips fields that are not accepted by the workflow creation API, then creates the workflows on the target instance. It supports two modes:

- **Default (static) mode:** source/target credentials are fixed via an n8n API credential.
- **Dynamic mode:** source/target instance URL + API key are loaded from a Notion database; the user selects source and target instances from a form.

### 1.1 Mode Selection & Entry
- Starts with a Form Trigger and routes to **Default** or **Dynamic** branches.

### 1.2 Dynamic Mode: Instance Selection & Source Discovery
- Loads instance records from Notion, lets user pick source/target, builds a combined “Source and Target” payload, then queries the source instance’s `/api/v1/workflows` with pagination.

### 1.3 Workflow Selection (Both Modes)
- Filters archived workflows.
- Builds a dynamic checklist form from discovered workflow names.
- Merges form selections back to workflow records to keep only selected workflows.

### 1.4 Workflow Fetch → Clean → Create (Both Modes)
- Retrieves full workflow JSON for each selected workflow.
- Strips incompatible fields using an allowlist strategy.
- Creates the workflows on the target instance (via n8n node in static mode; via raw HTTP in dynamic mode).

### 1.5 Results Formatting
- Aggregates per-workflow results and shows a final completion message in a Form node.

---

## 2. Block-by-Block Analysis

### Block A — Entry & Mode Routing
**Overview:** Receives the initial user input (mode choice) and routes execution to either static/default or dynamic instance-driven migration.

**Nodes involved:**
- On form submission
- Route Mode
- Sticky Note3
- Sticky Note

#### Node: On form submission
- **Type / Role:** `Form Trigger` — workflow entrypoint via hosted form.
- **Key configuration:**
  - Path: `migrate-dynamic-n8n-workflows`
  - Form title: “N8n Workflow Loader”
  - Field: Dropdown “Mode” with options `Default` and `Dynamic` (required).
  - Custom CSS: extensive dark theme styling.
- **Outputs:** to **Route Mode**.
- **Edge cases:**
  - If the trigger is in test mode, form URLs differ; ensure production URL is used for real usage.
  - If user submits unexpected value (edited client-side), switch rules may not match.

#### Node: Route Mode
- **Type / Role:** `Switch` — branches to Default vs Dynamic.
- **Key expressions:**
  - Checks `{{$json["Choose A Mode"]}}` equals `"Default"` or `"Dynamic"`.
- **Outputs:**
  - Output `default` → **Get All Source Instance Workflows** (static mode path)
  - Output `dynamic` → **Get Instance Information** (dynamic mode path)
- **Edge cases:**
  - If the form field label changes, the expression key `"Choose A Mode"` breaks.

#### Sticky Notes (documentation-only nodes)
- **Sticky Note3:** Explains high-level flow + setup steps.
- **Sticky Note:** Explains static vs dynamic mode differences.

---

### Block B — Default (Static) Mode: Discover & Select Workflows
**Overview:** Uses the n8n API credential (fixed instance) to list workflows, filter out archived ones, and present a checklist form for choosing which workflows to import.

**Nodes involved:**
- Get All Source Instance Workflows
- Filter Our Archived Items1
- Set Workflow Display Name (Static)
- Aggregate Workflow Options (Static)
- Create JSON Workflow Options1
- Select Workflows1
- Split Out Workflows1
- Select Matching Workflows1
- Sticky Note1

#### Node: Get All Source Instance Workflows
- **Type / Role:** `n8n` node (n8n API connector) — list workflows from the configured instance.
- **Configuration choices:**
  - Operation: implicit “get many” (node uses `filters` and `requestOptions`, and outputs workflow list).
  - Credentials: **Stephen n8n account** (`n8nApi`).
- **Outputs:** to **Filter Our Archived Items1**.
- **Failure modes:**
  - Invalid/expired n8n API credential.
  - Insufficient permissions (API key lacks workflow read).
  - Instance URL mismatch in credential.

#### Node: Filter Our Archived Items1
- **Type / Role:** `Filter` — remove archived workflows.
- **Condition:** `{{$json.isArchived}}` equals `false`.
- **Outputs:** to **Set Workflow Display Name (Static)** and **Select Matching Workflows1** (two parallel connections).
- **Edge cases:**
  - If API version changes field name (e.g., `isArchived` missing), strict validation may drop items or error.

#### Node: Set Workflow Display Name (Static)
- **Type / Role:** `Set` — normalizes the display name field for later option-building.
- **Assignments:**
  - `workflow_name = {{$json.name}}`
- **Outputs:** to **Aggregate Workflow Options (Static)**.

#### Node: Aggregate Workflow Options (Static)
- **Type / Role:** `Aggregate` — gathers all workflow items into a single item containing a `data` array.
- **Operation:** “aggregateAllItemData”.
- **Outputs:** to **Create JSON Workflow Options1**.
- **Edge cases:** Large instances may cause large payload; consider memory usage.

#### Node: Create JSON Workflow Options1
- **Type / Role:** `Code` — converts aggregated `data[]` into a string snippet of JSON objects for form options.
- **Key logic:**
  - Reads: `const data = $input.first().json.data;`
  - Builds: `options` as a comma-separated string of `{"option": "<workflow_name>"}` entries.
- **Outputs:** to **Select Workflows1**.
- **Edge cases / risks:**
  - Workflow names containing quotes, newlines, or backslashes can break the generated JSON string (because it interpolates directly into a JSON fragment).
  - If `data` is undefined (no workflows), code throws.

#### Node: Select Workflows1
- **Type / Role:** `Form` — dynamic checklist of workflow names.
- **Configuration:**
  - `defineForm: json`
  - `jsonOutput`: builds a checkbox field `workflows` with options injected via `{{ $json.options }}`.
- **Outputs:** to **Split Out Workflows1**.
- **Edge cases:**
  - If `options` string is malformed JSON, form rendering fails.
  - If too many options, UI may slow; CSS enables scrolling.

#### Node: Split Out Workflows1
- **Type / Role:** `Split Out` — creates one item per selected workflow name from the `workflows` array.
- **Field:** `workflows`.
- **Outputs:** to **Select Matching Workflows1** (input 2 of merge).

#### Node: Select Matching Workflows1
- **Type / Role:** `Merge (combine)` — matches selected names back to the full workflow objects.
- **Merge settings:**
  - Mode: `combine`
  - Merge by fields: `field1: name` (workflow object) equals `field2: workflows` (selected option string)
  - Output data from: `input1` (the workflow object stream)
- **Inputs:**
  - Input 1: from **Filter Our Archived Items1** (workflow objects)
  - Input 2: from **Split Out Workflows1** (selected names)
- **Outputs:** to **Get Workflow JSON(s)** (static fetch details stage).
- **Edge cases:**
  - Duplicate workflow names: merge may match multiple items unpredictably.
  - If user selects none, merge outputs nothing downstream.

---

### Block C — Default (Static) Mode: Fetch → Clean → Create → Results
**Overview:** For each selected workflow, fetches full JSON via n8n API connector, strips incompatible fields, creates workflows in target instance (also via connector), aggregates results, and displays a completion form.

**Nodes involved:**
- Get Workflow JSON(s)
- Strip Incompatible API Fields1
- Create Workflow(s)1
- Aggregate Workflows (Static)
- Results (Static)
- Sticky Note2
- Sticky Note6

#### Node: Get Workflow JSON(s)
- **Type / Role:** `n8n` node — fetch a single workflow by ID.
- **Configuration:**
  - Operation: `get`
  - Workflow ID: `{{ $('Select Matching Workflows1').item.json.id }}`
  - Credentials: **Stephen n8n account**
- **Outputs:** to **Strip Incompatible API Fields1**.
- **Failure modes:**
  - Missing ID due to merge mismatch.
  - Auth failure / insufficient permissions.

#### Node: Strip Incompatible API Fields1
- **Type / Role:** `Code` — builds an API-safe workflow payload (allowlist).
- **Key behavior:**
  - Keeps only: `name`, cleaned `nodes` (id/name/type/typeVersion/position/parameters/credentials/disabled/notes/notesInFlow), `connections`, plus allowlisted `settings` and optional `staticData`.
- **Settings allowlist:** `timezone`, `executionTimeout`, `saveExecutionProgress`, `saveManualExecutions`, `errorWorkflowId`.
- **Outputs:** to **Create Workflow(s)1**.
- **Edge cases:**
  - Nodes with missing required fields (rare) could cause creation failures.
  - Credentials references may not exist in target instance; creation succeeds but workflow may fail when executed, or n8n may reject invalid credential structure depending on version.

#### Node: Create Workflow(s)1
- **Type / Role:** `n8n` node — creates workflow in the (static) target instance configured by the credential.
- **Configuration:**
  - Operation: `create`
  - Workflow object: `{{ $json.toJsonString() }}`
  - Credentials: **Stephen n8n account** (note: in this workflow, source and target appear to use the same credential; in a real setup you typically use a distinct target credential).
- **Outputs:** to **Aggregate Workflows (Static)**.
- **Failure modes:**
  - Attempting to create workflows with same name may be allowed, but could cause confusion; n8n typically allows duplicates.
  - If the API expects different schema fields (n8n version mismatch), request can fail.

#### Node: Aggregate Workflows (Static)
- **Type / Role:** `Aggregate` — gathers all creation results.
- **Outputs:** to **Results (Static)**.

#### Node: Results (Static)
- **Type / Role:** `Form` (completion) — displays success/failure per workflow.
- **Completion message expression:**
  - Iterates `$json.data` and marks failed if `item.is_empty === true`, else success.
- **Edge cases:**
  - The `is_empty` flag is not guaranteed to exist in n8n API responses; if absent, everything will be treated as “Success”.
  - Better would be to rely on HTTP status / error handling, but this is not implemented.

---

### Block D — Dynamic Mode: Select Source/Target Instances (Notion)
**Overview:** Loads instance connection info from Notion, lets user select source and target instances, and creates a combined data structure used by downstream HTTP calls.

**Nodes involved:**
- Get Instance Information
- Set Source Name and URL
- Aggregate Workflow Options (Dynamic)
- Create JSON Workflow Options2
- Select Workflows2
- Merge Source Instance
- Merge Target Instance
- Set Source
- Set Target
- Merge Source and Target
- Source and Target
- Sticky Note5

#### Node: Get Instance Information
- **Type / Role:** `Notion` — reads records containing instance URL and API key.
- **Configuration:**
  - Resource: `databasePage`
  - Operation: `getAll`
  - Database: “Granite n8n Keys” (ID provided)
  - Return all: `true`
  - Credentials: **Stephen Notion account**
- **Outputs (three branches):**
  1) to **Set Source Name and URL** (to build dropdown options)
  2) to **Merge Target Instance** (input 1)
  3) to **Merge Source Instance** (input 1)
- **Failure modes:**
  - Notion credential revoked.
  - Database schema changed; expected properties (URL/API key) missing.
  - Large database: pagination handled by node, but may be slow.

#### Node: Set Source Name and URL
- **Type / Role:** `Set` — maps instance display name into `workflow_name` for option building.
- **Assignment:** `workflow_name = {{$json.name}}`
- **Outputs:** to **Aggregate Workflow Options (Dynamic)**.
- **Note:** Naming is slightly misleading (it sets instance name, not workflow name).

#### Node: Aggregate Workflow Options (Dynamic)
- **Type / Role:** `Aggregate` — gathers instance items into a single `data[]`.
- **Outputs:** to **Create JSON Workflow Options2**.

#### Node: Create JSON Workflow Options2
- **Type / Role:** `Code` — builds JSON option objects string from instance `workflow_name`.
- **Same risks as other option builders:**
  - Names with quotes can break JSON snippet.

#### Node: Select Workflows2
- **Type / Role:** `Form` — dynamic dropdowns to choose `source` and `target` instances.
- **Configuration:**
  - JSON-defined form with:
    - Dropdown `source` options injected from `{{ $json.options }}`
    - Dropdown `target` options injected from `{{ $json.options }}`
- **Outputs:** to **Merge Source Instance** (input 2) and **Merge Target Instance** (input 2).
- **Edge cases:**
  - It uses the same webhookId as another form node (`Select Workflows`), which can be problematic depending on n8n version/export; webhook IDs should generally be unique.

#### Node: Merge Source Instance
- **Type / Role:** `Merge (combine)` — matches the Notion instance record to the selected `source` name.
- **Merge by fields:** `name` == `source`
- **Output data from:** input1 (Notion record)
- **Outputs:** to **Set Source**.

#### Node: Merge Target Instance
- **Type / Role:** `Merge (combine)` — matches Notion instance record to selected `target`.
- **Merge by fields:** `name` == `target`
- **Outputs:** to **Set Target**.

#### Node: Set Source
- **Type / Role:** `Set` — standardizes the source object.
- **Assignments:**
  - `type = "source"`
  - `name = {{$json.name}}`
  - `url = {{$json.property_url}}`
  - `id = {{$json.id}}`
  - `api_key = {{$json.property_api_key}}`
- **Outputs:** to **Merge Source and Target** (input 1).
- **Failure modes:**
  - If `property_url` or `property_api_key` are not present (Notion property name mismatch), downstream HTTP requests fail.

#### Node: Set Target
- **Type / Role:** `Set` — standardizes the target object.
- **Assignments:** same fields, but `type = "target"`.
- **Outputs:** to **Merge Source and Target** (input 2).
- **Config note:** “includeOtherFields: true” is enabled here, so additional Notion fields will remain (unlike Set Source). Not harmful but inconsistent.

#### Node: Merge Source and Target
- **Type / Role:** `Merge` — appends both objects into one stream.
- **Outputs:** to **Source and Target**.

#### Node: Source and Target
- **Type / Role:** `Aggregate` — produces a single item with `data[]` containing both the source and target objects.
- **Outputs:** to **Get Source Workflows With Pagination**.
- **Downstream usage pattern:** Many HTTP nodes reference:
  - `$('Source and Target').first().json.data.filter(item => item.type === 'source'|'target')...`

---

### Block E — Dynamic Mode: Discover Workflows (Cursor Pagination)
**Overview:** Calls the source instance workflow list endpoint with cursor-based pagination, filters archived workflows, aggregates them, and prepares options for a workflow selection form.

**Nodes involved:**
- Get Source Workflows With Pagination
- Split Out
- Filter Our Archived Items
- Set Workflow Display Name (Dynamic)
- Aggregate Workflows
- Create JSON Workflow Options
- Select Workflows
- Split Out Workflows
- Select Matching Workflows
- Sticky Note4
- Sticky Note7

#### Node: Get Source Workflows With Pagination
- **Type / Role:** `HTTP Request` — GET `/api/v1/workflows` from the selected source.
- **URL expression:**
  - `{{ source.url }}/api/v1/workflows` where `source.url` is pulled from `Source and Target`.
- **Headers:** `X-N8N-API-KEY` from `source.api_key`.
- **Pagination:**
  - Cursor parameter: `cursor={{ $response.body.nextCursor }}`
  - Request interval: 1000ms
  - Pagination completes “when receive specific status codes”: `400`
    - This implies the instance returns 400 when cursor is invalid/finished.
- **Outputs:** to **Split Out**.
- **Failure modes / concerns:**
  - If the API does not return 400 to indicate completion, pagination may behave incorrectly.
  - If `nextCursor` is absent/null, cursor param may become invalid; behavior depends on n8n pagination handling.
  - 401/403 if API key invalid.

#### Node: Split Out
- **Type / Role:** `Split Out` — splits the returned `data[]` list into individual workflow items.
- **Field:** `data`
- **Outputs:** to **Filter Our Archived Items**.

#### Node: Filter Our Archived Items
- **Type / Role:** `Filter` — removes archived workflows (`isArchived == false`).
- **Outputs:**
  - to **Set Workflow Display Name (Dynamic)**
  - to **Select Matching Workflows** (input 1)
- **Edge cases:** same as static filter.

#### Node: Set Workflow Display Name (Dynamic)
- **Type / Role:** `Set` — maps `name` → `workflow_name`.
- **Outputs:** to **Aggregate Workflows**.

#### Node: Aggregate Workflows
- **Type / Role:** `Aggregate` — aggregates active workflows into one `data[]`.
- **Outputs:** to **Create JSON Workflow Options**.

#### Node: Create JSON Workflow Options
- **Type / Role:** `Code` — builds JSON snippet string of checkbox options from `workflow_name`.
- **Outputs:** to **Select Workflows**.
- **Edge cases:** unsafe string interpolation can break JSON if names contain quotes.

#### Node: Select Workflows
- **Type / Role:** `Form` — checkbox list of workflows to import.
- **Outputs:** to **Split Out Workflows**.

#### Node: Split Out Workflows
- **Type / Role:** `Split Out` — one item per selected workflow name.
- **Outputs:** to **Select Matching Workflows** (input 2).

#### Node: Select Matching Workflows
- **Type / Role:** `Merge (combine)` — matches selected workflow names back to full workflow items.
- **Merge by:** `name` == `workflows`
- **Output:** input1 data (workflow objects)
- **Outputs:** to **Get Workflow JSON(s)1**.
- **Edge cases:** duplicate names, empty selection.

---

### Block F — Dynamic Mode: Fetch → Clean → Create → Results
**Overview:** Fetches each selected workflow’s full JSON from the source via HTTP, strips incompatible fields, posts cleaned workflow JSON to the target instance, aggregates outcomes, and shows a completion message.

**Nodes involved:**
- Get Workflow JSON(s)1
- Strip Incompatible API Fields
- Create Workflow(s)
- Aggregate Workflows (Dynamic)
- Results (Dynamic)
- Sticky Note8
- Sticky Note9

#### Node: Get Workflow JSON(s)1
- **Type / Role:** `HTTP Request` — GET a single workflow JSON from source.
- **URL expression:**
  - `{{ source.url }}/api/v1/workflows/{{ $('Select Matching Workflows').item.json.id }}`
- **Headers:** `X-N8N-API-KEY` from source.
- **Outputs:** to **Strip Incompatible API Fields**.
- **Failure modes:**
  - If the merge produced no `id` (or wrong item scoping), this request fails.
  - 404 if workflow removed between discovery and selection.

#### Node: Strip Incompatible API Fields
- **Type / Role:** `Code` — same concept as static cleaning but includes a **default `settings` object** hardcoded (then overwritten by allowlisted settings if present).
- **Behavior highlights:**
  - Always sets `workflow.settings` initially to a fixed bundle (execution logging/timezone/etc).
  - Then, if `source.settings` exists, it overwrites with allowlisted settings only (timezone, executionTimeout, saveExecutionProgress, saveManualExecutions, errorWorkflowId).
  - Keeps optional `staticData`.
- **Outputs:** to **Create Workflow(s)**.
- **Edge cases / version concerns:**
  - Some hardcoded settings keys (e.g., `callerPolicy`, `callerIds`, `availableInMCP`, `timeSavedPerExecution`) may be unknown in some n8n versions, but note: those are only present in the initial object and may be overwritten/removed if `source.settings` exists. If `source.settings` does **not** exist, these extra keys will be sent and could cause API rejection on stricter servers.

#### Node: Create Workflow(s)
- **Type / Role:** `HTTP Request` — POST to target `/api/v1/workflows`.
- **URL expression:** `{{ target.url }}/api/v1/workflows`
- **Headers:** `X-N8N-API-KEY` from target.
- **Body:** JSON from `{{ $json.toJsonString() }}`.
- **Outputs:** to **Aggregate Workflows (Dynamic)**.
- **Failure modes:**
  - 401/403 invalid API key.
  - 400 validation errors due to schema mismatch or invalid node structures.
  - Credential references in nodes may not exist on target; behavior depends on API validation.

#### Node: Aggregate Workflows (Dynamic)
- **Type / Role:** `Aggregate` — collects all create results.
- **Outputs:** to **Results (Dynamic)**.

#### Node: Results (Dynamic)
- **Type / Role:** `Form` completion — displays per-workflow success/failure.
- **Completion message logic:** same as static (`is_empty` flag check).
- **Edge cases:** same caveat: `is_empty` may not reflect real API success/failure.

---

### Block G — Unused / Orphaned Nodes (Present but not executed)
These nodes exist in the workflow but are not connected from any entry path (based on the provided connections). They must still be documented because they are part of the workflow JSON.

#### Node: Create JSON Workflow Options (Static) — `Create JSON Workflow Options` (at position 2096,1200) and related chain
- In practice, the dynamic branch uses this node; the naming collision is confusing. The “Create JSON Workflow Options” node at (2096,1200) is used in dynamic workflow selection block (Block E).
- The similarly named nodes (`Create JSON Workflow Options1`, `Create JSON Workflow Options2`) are used for static workflow selection and dynamic instance selection respectively.

#### Node: Select Workflows2 webhookId duplication
- `Select Workflows2` shares `webhookId` with `Select Workflows` (`d6923eb5...`). This can cause deployment conflicts or unexpected behavior in some setups.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Entry point; collects mode selection | — | Route Mode | ## How it works 🧠 (Workflow explanation)\n\nThis workflow lets you selectively import workflows between n8n instances using the n8n API and forms.\n\nInstead of bulk imports or manual exports, it dynamically retrieves workflows from a source instance and lets you choose exactly what to load into a target instance.\n\n### Here’s what happens:\n1. The workflow retrieves workflows from a source n8n instance via the API.\n2. A dynamic form is generated listing available workflows.\n3. You select which workflows to import.\n4. Selected workflows are formatted for API compatibility.\n5. Workflows are created in the target n8n instance.\n\n### This ensures:\n- No bulk imports  \n- No accidental overwrites  \n- Full control over what gets moved  \n- Safe, API-driven workflow creation  \n\n## Setup steps ⚙️\n\n**Estimated setup time:** 2–5 minutes\n\n### Simple mode (default)\n1. Add API credentials for the source n8n instance.\n2. Add API credentials for the target n8n instance.\n3. Execute the workflow and select workflows to import.\n\n### Dynamic mode (optional)\n1. Connect a database (e.g. Notion or Supabase) containing:\n   - n8n instance URLs  \n   - n8n API keys  \n2. Select the source and target instances via the form.\n3. Choose which workflows to import and run.\n\nOnce credentials are connected, the workflow is ready to use 🚀 |
| Route Mode | Switch | Routes to Default vs Dynamic branch | On form submission | Get All Source Instance Workflows; Get Instance Information | ## 🔁 Mode selection (static vs dynamic)\n\nThis section determines how the source and target n8n instances are configured.\n\n**Simple mode**\n- Uses n8n API credentials configured directly in the workflow\n- Source and target instances are fixed\n- Fastest way to get started\n\n**Dynamic mode**\n- Loads instance URLs and API keys from an external database (e.g. Notion)\n- Lets you select source and target instances via a form\n- Ideal for managing multiple n8n instances securely\n\nBoth modes use the same import logic downstream; only the instance configuration differs. |
| Get All Source Instance Workflows | n8n | List workflows (static mode) | Route Mode (default) | Filter Our Archived Items1 | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Filter Our Archived Items1 | Filter | Remove archived workflows (static) | Get All Source Instance Workflows | Set Workflow Display Name (Static); Select Matching Workflows1 | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Set Workflow Display Name (Static) | Set | Map `name` → `workflow_name` (static) | Filter Our Archived Items1 | Aggregate Workflow Options (Static) | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Aggregate Workflow Options (Static) | Aggregate | Aggregate workflows list for option building (static) | Set Workflow Display Name (Static) | Create JSON Workflow Options1 | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Create JSON Workflow Options1 | Code | Build checkbox option JSON snippet (static) | Aggregate Workflow Options (Static) | Select Workflows1 | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Select Workflows1 | Form | Checklist form for workflows (static) | Create JSON Workflow Options1 | Split Out Workflows1 | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Split Out Workflows1 | Split Out | One item per selected workflow (static) | Select Workflows1 | Select Matching Workflows1 | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Select Matching Workflows1 | Merge | Match selected names to workflow objects (static) | Filter Our Archived Items1; Split Out Workflows1 | Get Workflow JSON(s) | ## Discover & Select Workflows (static mode)\n\nThis section retrieves workflows from the source n8n instance and prepares them for user selection.\n\n**What happens**\n- Fetches all available workflows via the n8n API\n- Filters out archived or inactive workflows\n- Aggregates results into a single list\n- Builds a dynamic selection form (JSON mode)\n\nYou then select exactly which workflows to import.  \nOnly selected workflows continue downstream. |
| Get Workflow JSON(s) | n8n | Fetch full workflow JSON by ID (static) | Select Matching Workflows1 | Strip Incompatible API Fields1 | ## Import & Clean Workflows\n\nThis section prepares selected workflows for safe import into the target n8n instance.\n\n**What happens**\n- Retrieves each selected workflow via the n8n API\n- Removes fields that are incompatible with workflow creation\n- Normalises the workflow structure for API import\n- Creates the workflow in the target instance\n\nThis ensures workflows are imported cleanly without conflicts or invalid metadata. |
| Strip Incompatible API Fields1 | Code | Allowlist workflow payload (static) | Get Workflow JSON(s) | Create Workflow(s)1 | ## Import & Clean Workflows\n\nThis section prepares selected workflows for safe import into the target n8n instance.\n\n**What happens**\n- Retrieves each selected workflow via the n8n API\n- Removes fields that are incompatible with workflow creation\n- Normalises the workflow structure for API import\n- Creates the workflow in the target instance\n\nThis ensures workflows are imported cleanly without conflicts or invalid metadata. |
| Create Workflow(s)1 | n8n | Create workflow in target (static) | Strip Incompatible API Fields1 | Aggregate Workflows (Static) | ## Import & Clean Workflows\n\nThis section prepares selected workflows for safe import into the target n8n instance.\n\n**What happens**\n- Retrieves each selected workflow via the n8n API\n- Removes fields that are incompatible with workflow creation\n- Normalises the workflow structure for API import\n- Creates the workflow in the target instance\n\nThis ensures workflows are imported cleanly without conflicts or invalid metadata. |
| Aggregate Workflows (Static) | Aggregate | Aggregate creation results (static) | Create Workflow(s)1 | Results (Static) | ## Structure Success Message\n\nFormats the final result for each processed workflow by creating a clean summary showing whether each workflow was created or failed. |
| Results (Static) | Form (completion) | Display summary (static) | Aggregate Workflows (Static) | — | ## Structure Success Message\n\nFormats the final result for each processed workflow by creating a clean summary showing whether each workflow was created or failed. |
| Get Instance Information | Notion | Load instance URL/API keys (dynamic) | Route Mode (dynamic) | Set Source Name and URL; Merge Target Instance; Merge Source Instance | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Set Source Name and URL | Set | Map instance name for dropdown building | Get Instance Information | Aggregate Workflow Options (Dynamic) | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Aggregate Workflow Options (Dynamic) | Aggregate | Aggregate instances for dropdown options | Set Source Name and URL | Create JSON Workflow Options2 | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Create JSON Workflow Options2 | Code | Build dropdown option JSON snippet for instances | Aggregate Workflow Options (Dynamic) | Select Workflows2 | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Select Workflows2 | Form | Select source & target instances | Create JSON Workflow Options2 | Merge Source Instance; Merge Target Instance | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Merge Source Instance | Merge | Match selected `source` to instance record | Get Instance Information; Select Workflows2 | Set Source | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Merge Target Instance | Merge | Match selected `target` to instance record | Get Instance Information; Select Workflows2 | Set Target | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Set Source | Set | Normalize source object (url/api_key/type) | Merge Source Instance | Merge Source and Target | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Set Target | Set | Normalize target object (url/api_key/type) | Merge Target Instance | Merge Source and Target | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Merge Source and Target | Merge | Combine source and target objects | Set Source; Set Target | Source and Target | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Source and Target | Aggregate | Produce single `data[]` containing both instances | Merge Source and Target | Get Source Workflows With Pagination | ## Select source and target instances (dynamic mode)\n\nThis section builds the final **Source and Target** payload by combining:\n\n- **Instance records** pulled from your database (e.g. Notion)\n- The user’s form selections (which instance is “source” and which is “target”)\n\n### What happens\n1. **Get Instance Information** loads all saved instances (URL + API key + display name).\n2. Two branches prepare each side:\n   - **Merge Source Instance → Set Source**\n   - **Merge Target Instance → Set Target**\n3. **Merge Source and Target** appends both objects into one array.\n4. **Source and Target** outputs a single structure used downstream to dynamically set:\n   - The base URL (instance URL)\n   - The API key header\n   - The instance role (`source` vs `target`)\n\n✅ Outcome: downstream nodes can reference the selected instances without hardcoding credentials in the workflow.\n⚠️ Keep API keys in a secure database, not inside n8n nodes or sticky notes. |
| Get Source Workflows With Pagination | HTTP Request | List workflows from source with cursor pagination | Source and Target | Split Out | ## Discover workflows\n\nThis section retrieves all workflows from the selected n8n instance.\n\nIt:\n- Calls the n8n `/api/v1/workflows` endpoint\n- Filters out archived workflows\n- Aggregates active workflows into a single list\n\nPagination is handled using a cursor-based approach, ensuring all workflows are retrieved even on large instances. |
| Split Out | Split Out | Split returned `data[]` to items | Get Source Workflows With Pagination | Filter Our Archived Items | ## Discover workflows\n\nThis section retrieves all workflows from the selected n8n instance.\n\nIt:\n- Calls the n8n `/api/v1/workflows` endpoint\n- Filters out archived workflows\n- Aggregates active workflows into a single list\n\nPagination is handled using a cursor-based approach, ensuring all workflows are retrieved even on large instances. |
| Filter Our Archived Items | Filter | Remove archived workflows (dynamic) | Split Out | Set Workflow Display Name (Dynamic); Select Matching Workflows | ## Discover workflows\n\nThis section retrieves all workflows from the selected n8n instance.\n\nIt:\n- Calls the n8n `/api/v1/workflows` endpoint\n- Filters out archived workflows\n- Aggregates active workflows into a single list\n\nPagination is handled using a cursor-based approach, ensuring all workflows are retrieved even on large instances. |
| Set Workflow Display Name (Dynamic) | Set | Map `name` → `workflow_name` (dynamic) | Filter Our Archived Items | Aggregate Workflows | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Aggregate Workflows | Aggregate | Aggregate workflows for option-building (dynamic) | Set Workflow Display Name (Dynamic) | Create JSON Workflow Options | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Create JSON Workflow Options | Code | Build checkbox option JSON snippet (dynamic) | Aggregate Workflows | Select Workflows | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Select Workflows | Form | Checklist form for workflows (dynamic) | Create JSON Workflow Options | Split Out Workflows | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Split Out Workflows | Split Out | One item per selected workflow (dynamic) | Select Workflows | Select Matching Workflows | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Select Matching Workflows | Merge | Match selected names to workflow objects (dynamic) | Filter Our Archived Items; Split Out Workflows | Get Workflow JSON(s)1 | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Get Workflow JSON(s)1 | HTTP Request | Fetch full workflow JSON by ID (dynamic) | Select Matching Workflows | Strip Incompatible API Fields | ## Import & clean workflows\n\nThis section prepares selected workflows for safe import into the target n8n instance.\n\n**What happens**\n- Retrieves each selected workflow via the n8n API\n- Removes fields incompatible with workflow creation\n- Normalises the workflow structure for import\n- Creates the workflow in the target n8n instance\n\nThis ensures workflows are imported cleanly without conflicts or invalid metadata. |
| Strip Incompatible API Fields | Code | Allowlist workflow payload (dynamic) | Get Workflow JSON(s)1 | Create Workflow(s) | ## Import & clean workflows\n\nThis section prepares selected workflows for safe import into the target n8n instance.\n\n**What happens**\n- Retrieves each selected workflow via the n8n API\n- Removes fields incompatible with workflow creation\n- Normalises the workflow structure for import\n- Creates the workflow in the target n8n instance\n\nThis ensures workflows are imported cleanly without conflicts or invalid metadata. |
| Create Workflow(s) | HTTP Request | Create workflow in target (dynamic) | Strip Incompatible API Fields | Aggregate Workflows (Dynamic) | ## Import & clean workflows\n\nThis section prepares selected workflows for safe import into the target n8n instance.\n\n**What happens**\n- Retrieves each selected workflow via the n8n API\n- Removes fields incompatible with workflow creation\n- Normalises the workflow structure for import\n- Creates the workflow in the target n8n instance\n\nThis ensures workflows are imported cleanly without conflicts or invalid metadata. |
| Aggregate Workflows (Dynamic) | Aggregate | Aggregate creation results (dynamic) | Create Workflow(s) | Results (Dynamic) | ## Structure success message\n\nFormats the final result for each processed workflow.\n\nCreates a clear summary showing whether each workflow was successfully created or failed, ensuring a consistent and easy-to-read output. |
| Results (Dynamic) | Form (completion) | Display summary (dynamic) | Aggregate Workflows (Dynamic) | — | ## Structure success message\n\nFormats the final result for each processed workflow.\n\nCreates a clear summary showing whether each workflow was successfully created or failed, ensuring a consistent and easy-to-read output. |
| Create JSON Workflow Options | Code | (already listed above) | Aggregate Workflows | Select Workflows | ## Select workflows\n\nBuilds a dynamic form that allows you to choose which workflows to import.\n\nIt:\n- Converts the discovered workflows into form options\n- Presents them using an n8n Form node\n- Passes only selected workflows downstream\n\nOnly workflows selected by the user continue to the import stage. |
| Create JSON Workflow Options2 | Code | (already listed above) | Aggregate Workflow Options (Dynamic) | Select Workflows2 | ## Select source and target instances (dynamic mode)\n\n... |
| Sticky Note, Sticky Note1..9 | Sticky Note | Documentation | — | — | (content shown in respective rows above where applicable) |
| Select Workflows (duplicate-named nodes) | Form | Workflow selection UI | Create JSON Workflow Options | Split Out Workflows | ## Select workflows\n\nBuilds a dynamic form... |

> Note: Sticky notes are documentation nodes; they don’t connect but “cover” sections. The table duplicates their content across covered nodes above.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add Form Trigger** node:
   - Node: **Form Trigger**
   - Path: `migrate-dynamic-n8n-workflows`
   - Title: “N8n Workflow Loader”
   - Description: “Load select workflows from your chosen n8n instance.”
   - Field: Dropdown:
     - Label: `Mode` (field name in output is `"Choose A Mode"` in this workflow)
     - Options: `Default`, `Dynamic`
     - Required: true
   - (Optional) add the provided custom CSS for dark theme.

3) **Add Switch** node “Route Mode”:
   - Two rules:
     - Output key `default`: `{{$json["Choose A Mode"]}} == "Default"`
     - Output key `dynamic`: `{{$json["Choose A Mode"]}} == "Dynamic"`
   - Connect **Form Trigger → Route Mode**.

---

### Build Default (Static) branch

4) **Add n8n node** “Get All Source Instance Workflows”:
   - Node type: **n8n**
   - Operation: list/get many workflows (workflows collection)
   - Configure **n8n API credential** for the *source instance*.
     - In n8n, create an **n8n API credential** pointing to the source base URL and API key.
   - Connect: **Route Mode (default) → Get All Source Instance Workflows**.

5) **Add Filter** “Filter Our Archived Items1”:
   - Condition: boolean equals
   - Left: `{{$json.isArchived}}`
   - Right: `false`
   - Connect: **Get All Source Instance Workflows → Filter Our Archived Items1**.

6) **Add Set** “Set Workflow Display Name (Static)”:
   - Add field: `workflow_name = {{$json.name}}`
   - Connect: **Filter Our Archived Items1 → Set Workflow Display Name (Static)**.

7) **Add Aggregate** “Aggregate Workflow Options (Static)”:
   - Aggregate all item data into one item
   - Connect: **Set Workflow Display Name (Static) → Aggregate Workflow Options (Static)**.

8) **Add Code** “Create JSON Workflow Options1”:
   - Use code that transforms `data[]` into a list of `{ "option": "<name>" }` objects (as a string), matching the workflow.
   - Connect: **Aggregate Workflow Options (Static) → Create JSON Workflow Options1**.

9) **Add Form** “Select Workflows1”:
   - Define form: JSON
   - JSON output should create a checkbox field `workflows` with options injected from `{{$json.options}}`.
   - Connect: **Create JSON Workflow Options1 → Select Workflows1**.

10) **Add Split Out** “Split Out Workflows1”:
   - Field to split: `workflows`
   - Connect: **Select Workflows1 → Split Out Workflows1**.

11) **Add Merge (combine)** “Select Matching Workflows1”:
   - Mode: Combine
   - Merge by fields:
     - Input1 field: `name`
     - Input2 field: `workflows`
   - Output: input1
   - Connect:
     - **Filter Our Archived Items1 → Select Matching Workflows1 (input1)**
     - **Split Out Workflows1 → Select Matching Workflows1 (input2)**

12) **Add n8n node** “Get Workflow JSON(s)”:
   - Operation: Get workflow by ID
   - Workflow ID expression: `{{ $('Select Matching Workflows1').item.json.id }}`
   - Credential: same n8n API credential as source
   - Connect: **Select Matching Workflows1 → Get Workflow JSON(s)**

13) **Add Code** “Strip Incompatible API Fields1”:
   - Paste the allowlist-based cleaner (static version).
   - Connect: **Get Workflow JSON(s) → Strip Incompatible API Fields1**

14) **Add n8n node** “Create Workflow(s)1”:
   - Operation: Create workflow
   - Workflow object: `{{ $json.toJsonString() }}`
   - Credential: **n8n API credential for the target instance** (create a separate one).
   - Connect: **Strip Incompatible API Fields1 → Create Workflow(s)1**

15) **Add Aggregate** “Aggregate Workflows (Static)”:
   - Aggregate all item data
   - Connect: **Create Workflow(s)1 → Aggregate Workflows (Static)**

16) **Add Form (completion)** “Results (Static)”:
   - Operation: Completion
   - Completion message: map over `$json.data` and show success/fail.
   - Connect: **Aggregate Workflows (Static) → Results (Static)**

---

### Build Dynamic branch (Notion-driven instance selection)

17) **Add Notion node** “Get Instance Information”:
   - Resource: Database page
   - Operation: Get all
   - Database: select your database containing instance name, URL, API key.
   - Credential: Notion integration token.
   - Connect: **Route Mode (dynamic) → Get Instance Information**.

18) **Add Set** “Set Source Name and URL”:
   - `workflow_name = {{$json.name}}`
   - Connect: **Get Instance Information → Set Source Name and URL**

19) **Add Aggregate** “Aggregate Workflow Options (Dynamic)”:
   - Aggregate all
   - Connect: **Set Source Name and URL → Aggregate Workflow Options (Dynamic)**

20) **Add Code** “Create JSON Workflow Options2”:
   - Build `options` string from aggregated `data[]` items’ `workflow_name`.
   - Connect: **Aggregate Workflow Options (Dynamic) → Create JSON Workflow Options2**

21) **Add Form** “Select Workflows2” (instance selection):
   - JSON-defined form with two dropdowns:
     - `source` options from `{{$json.options}}`
     - `target` options from `{{$json.options}}`
   - Connect: **Create JSON Workflow Options2 → Select Workflows2**

22) **Add Merge (combine)** “Merge Source Instance”:
   - Merge by `name` (Notion record) == `source` (form selection)
   - Output input1
   - Connect:
     - **Get Instance Information → Merge Source Instance (input1)**
     - **Select Workflows2 → Merge Source Instance (input2)**

23) **Add Merge (combine)** “Merge Target Instance”:
   - Merge by `name` == `target`
   - Connect:
     - **Get Instance Information → Merge Target Instance (input1)**
     - **Select Workflows2 → Merge Target Instance (input2)**

24) **Add Set** “Set Source”:
   - Fields:
     - `type = "source"`
     - `name = {{$json.name}}`
     - `url = {{$json.property_url}}` (adjust to your Notion property mapping)
     - `api_key = {{$json.property_api_key}}` (adjust)
     - `id = {{$json.id}}`
   - Connect: **Merge Source Instance → Set Source**

25) **Add Set** “Set Target”:
   - Same fields but `type = "target"`.
   - Connect: **Merge Target Instance → Set Target**

26) **Add Merge** “Merge Source and Target”:
   - Append/combine the two streams (default merge settings).
   - Connect:
     - **Set Source → Merge Source and Target (input1)**
     - **Set Target → Merge Source and Target (input2)**

27) **Add Aggregate** “Source and Target”:
   - Aggregate all to produce one item with `data[]`.
   - Connect: **Merge Source and Target → Source and Target**

28) **Add HTTP Request** “Get Source Workflows With Pagination”:
   - Method: GET
   - URL: `{{ $('Source and Target').first().json.data.filter(i => i.type === 'source').map(i => i.url).toString() }}/api/v1/workflows`
   - Header: `X-N8N-API-KEY` = source api_key expression
   - Pagination:
     - cursor param: `{{ $response.body.nextCursor }}`
     - interval: 1000ms
     - stop when HTTP 400 (as in the workflow) or adjust to your instance behavior.
   - Connect: **Source and Target → Get Source Workflows With Pagination**

29) **Add Split Out** “Split Out”:
   - Field: `data`
   - Connect: **Get Source Workflows With Pagination → Split Out**

30) **Add Filter** “Filter Our Archived Items”:
   - `{{$json.isArchived}} == false`
   - Connect: **Split Out → Filter Our Archived Items**

31) **Add Set** “Set Workflow Display Name (Dynamic)”:
   - `workflow_name = {{$json.name}}`
   - Connect: **Filter Our Archived Items → Set Workflow Display Name (Dynamic)**

32) **Add Aggregate** “Aggregate Workflows”:
   - Aggregate all
   - Connect: **Set Workflow Display Name (Dynamic) → Aggregate Workflows**

33) **Add Code** “Create JSON Workflow Options”:
   - Build checkbox options snippet from aggregated workflow names.
   - Connect: **Aggregate Workflows → Create JSON Workflow Options**

34) **Add Form** “Select Workflows”:
   - Checkbox list of workflows.
   - Connect: **Create JSON Workflow Options → Select Workflows**

35) **Add Split Out** “Split Out Workflows”:
   - Field: `workflows`
   - Connect: **Select Workflows → Split Out Workflows**

36) **Add Merge (combine)** “Select Matching Workflows”:
   - Merge by `name` == `workflows`
   - Output input1
   - Connect:
     - **Filter Our Archived Items → Select Matching Workflows (input1)**
     - **Split Out Workflows → Select Matching Workflows (input2)**

37) **Add HTTP Request** “Get Workflow JSON(s)1”:
   - GET:
   - URL: `{{ source.url }}/api/v1/workflows/{{ $('Select Matching Workflows').item.json.id }}`
   - Headers: `X-N8N-API-KEY` from source
   - Connect: **Select Matching Workflows → Get Workflow JSON(s)1**

38) **Add Code** “Strip Incompatible API Fields”:
   - Paste the dynamic cleaner (allowlist + settings behavior).
   - Connect: **Get Workflow JSON(s)1 → Strip Incompatible API Fields**

39) **Add HTTP Request** “Create Workflow(s)”:
   - POST
   - URL: `{{ target.url }}/api/v1/workflows`
   - Headers: `X-N8N-API-KEY` from target
   - Body: JSON = `{{ $json.toJsonString() }}`
   - Connect: **Strip Incompatible API Fields → Create Workflow(s)**

40) **Add Aggregate** “Aggregate Workflows (Dynamic)”:
   - Aggregate all
   - Connect: **Create Workflow(s) → Aggregate Workflows (Dynamic)**

41) **Add Form (completion)** “Results (Dynamic)”:
   - Completion message expression similar to the workflow.
   - Connect: **Aggregate Workflows (Dynamic) → Results (Dynamic)**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Two-mode design: “Default” uses fixed n8n credentials; “Dynamic” pulls URL/API keys from a database and lets the user pick instances. | Sticky Note (mode selection) |
| Dynamic pagination completion relies on receiving HTTP 400 to stop. If your n8n instance returns a different signal, adjust pagination stop condition. | Get Source Workflows With Pagination configuration |
| Option-building code constructs JSON snippets via raw string interpolation; workflow/instance names containing quotes can break form JSON. Consider escaping or building arrays directly (safer). | Create JSON Workflow Options / 1 / 2 |
| Completion result logic checks `is_empty` which may not be a reliable indicator of creation failures; consider adding error-handling branches and explicit status capture. | Results (Static/Dynamic) |
| Potential conflict: `Select Workflows2` and `Select Workflows` share the same webhookId in the provided JSON; ensure unique webhook IDs when rebuilding. | Form nodes (dynamic) |

