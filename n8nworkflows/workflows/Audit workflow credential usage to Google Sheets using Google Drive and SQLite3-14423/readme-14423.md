Audit workflow credential usage to Google Sheets using Google Drive and SQLite3

https://n8nworkflows.xyz/workflows/audit-workflow-credential-usage-to-google-sheets-using-google-drive-and-sqlite3-14423


# Audit workflow credential usage to Google Sheets using Google Drive and SQLite3

## 1. Workflow Overview

This workflow audits credential usage across a self-hosted n8n instance and exports the results into two Google Sheets stored in Google Drive:

1. **Workflow Credentials Inventory**: for each workflow, list the nodes that use credentials and the credential names involved.
2. **Credential Impact Map**: for each credential, list the workflows that reference it.

It combines two data sources:

- **n8n public API** for workflow and credential metadata
- **internal SQLite database** for version labels and credential-to-workflow lookup not directly available from the API

It is designed for **self-hosted n8n Docker environments**, and explicitly depends on direct database access through Code nodes using the `sqlite3` module.

### 1.1 Entry and Global Preparation
The workflow starts on a weekly schedule, with an additional disabled manual trigger for testing. It creates a timestamped Google Drive folder that will contain both exported spreadsheets.

### 1.2 Branch 1: Workflow Credentials Inventory
This branch creates a spreadsheet, retrieves all workflows from the n8n API, enriches them with version label data from SQLite, filters only workflows and nodes that actually use credentials, aggregates credential usage per workflow, formats the result, appends it to Google Sheets, and moves the file into the newly created Drive folder.

### 1.3 Branch 2: Credential Impact Map
This branch creates a second spreadsheet, retrieves all credentials from the n8n API, sorts them, looks up all workflows referencing each credential via SQLite, formats the result, appends it to Google Sheets, and moves the file into the same Drive folder.

### 1.4 Time Tracking
Each branch ends with a Time Saved node to estimate manual effort avoided.

---

## 2. Block-by-Block Analysis

## 2.1 Entry and Scheduling

**Overview:**  
This block defines how the workflow starts. It supports both scheduled execution and ad hoc manual execution, though the manual path is disabled by default.

**Nodes Involved:**  
- Schedule Trigger
- When clicking ‘Execute workflow’

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger` — primary production entry point.
- **Configuration:** Runs every week on **Friday at 14:00**.
- **Expressions/variables:** None.
- **Input/Output:** No input; outputs to **Create folder**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failures:** Misconfigured timezone at instance level may shift actual run time.
- **Sub-workflow reference:** None.

#### When clicking ‘Execute workflow’
- **Type and role:** `n8n-nodes-base.manualTrigger` — optional testing entry point.
- **Configuration:** No parameters; node is **disabled**.
- **Expressions/variables:** None.
- **Input/Output:** No input; would output to **Create folder** if enabled.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** None beyond being disabled.
- **Sub-workflow reference:** None.

---

## 2.2 Global Output Folder Creation

**Overview:**  
This block creates a dated folder in Google Drive and acts as the shared setup step for both reporting branches.

**Nodes Involved:**  
- Create folder

### Node Details

#### Create folder
- **Type and role:** `n8n-nodes-base.googleDrive` — creates a target folder for output spreadsheets.
- **Configuration choices:**
  - Resource: **folder**
  - Folder name: current date and current time
  - Parent folder: fixed folder ID placeholder that must be replaced
  - Drive: **My Drive**
  - Folder color set to `#F18F17`
- **Key expressions/variables used:**
  - `={{ $now.toISODate() }}_{{ $now.toISOTime().split('.')[0] }}`
- **Input and output connections:**
  - Input from **Schedule Trigger** and **When clicking ‘Execute workflow’**
  - Outputs to **Create credMap** and **Create credInventory**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - Missing or invalid Google Drive OAuth2 credentials
  - Invalid parent folder ID
  - Permissions error if the target Drive/folder is not accessible
- **Sub-workflow reference:** None.

---

## 2.3 Branch 1 Setup: Spreadsheet Creation and Workflow Retrieval

**Overview:**  
This block creates the workflow inventory spreadsheet and retrieves the workflow list from the n8n API, splitting the API response into individual workflow items.

**Nodes Involved:**  
- Create credInventory
- flowsList
- splitFlows

### Node Details

#### Create credInventory
- **Type and role:** `n8n-nodes-base.googleSheets` — creates the spreadsheet for workflow-level audit output.
- **Configuration choices:**
  - Resource: **spreadsheet**
  - Spreadsheet title includes timestamp
  - Initial sheet: `Sheet1`
- **Key expressions/variables used:**
  - `=Workflow-Credentials-Inventory_{{ $now.toISODate() }}_{{ $now.toISOTime().split('.')[0] }}`
- **Input and output connections:**
  - Input from **Create folder**
  - Output to **flowsList**
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failures:**
  - Google Sheets OAuth2 authentication issues
  - Spreadsheet creation quota/permission errors
- **Sub-workflow reference:** None.

#### flowsList
- **Type and role:** `n8n-nodes-base.httpRequest` — fetches workflows from the n8n REST API.
- **Configuration choices:**
  - URL points to `/api/v1/workflows`
  - Header `X-N8N-API-KEY` manually supplied
  - Authentication mode effectively handled through custom header
- **Key expressions/variables used:** Static placeholders must be replaced:
  - `https://<WEBHOOK DOMAIN GOES HERE>/api/v1/workflows`
  - `API KEY GOES HERE`
- **Input and output connections:**
  - Input from **Create credInventory**
  - Output to **splitFlows**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases/failures:**
  - Invalid API key
  - Wrong base URL
  - SSL or reverse proxy issues
  - Response shape mismatch if API version differs
- **Sub-workflow reference:** None.

#### splitFlows
- **Type and role:** `n8n-nodes-base.splitOut` — expands the API response `data` array into one item per workflow.
- **Configuration choices:**
  - Field to split out: `data`
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **flowsList**
  - Output to **version_labels**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:**
  - If the API response has no `data` field, the node fails or emits nothing depending on runtime behavior
- **Sub-workflow reference:** None.

---

## 2.4 Branch 1 Enrichment: Workflow Version Labels from SQLite

**Overview:**  
This block supplements each workflow with the latest version label from the internal SQLite database, then normalizes workflow metadata into a reusable structure.

**Nodes Involved:**  
- version_labels
- flowEdits

### Node Details

#### version_labels
- **Type and role:** `n8n-nodes-base.code` — queries SQLite for the latest workflow history label.
- **Configuration choices:**
  - Runs once per item
  - Uses Node.js `sqlite3` module
  - Opens `/home/node/.n8n/database.sqlite` in read-only mode
  - Queries `workflow_history` by `workflowId`
  - Returns fallback values like `Conn Error: ...`, `Query Error`, or `No Label`
- **Key expressions/variables used:**
  - Workflow ID: `$json.id`
  - SQL:
    - `SELECT name FROM workflow_history WHERE workflowId = ? AND name IS NOT NULL ORDER BY createdAt DESC LIMIT 1`
- **Input and output connections:**
  - Input from **splitFlows**
  - Output to **flowEdits**
- **Version-specific requirements:**
  - Type version `2`
  - Requires instance env vars:
    - `NODE_FUNCTION_ALLOW_BUILTIN=*`
    - `NODE_FUNCTION_ALLOW_EXTERNAL=sqlite3`
  - Requires `sqlite3` to be available in runtime
- **Edge cases/failures:**
  - Code node external module access disabled
  - Database file path differs from `/home/node/.n8n/database.sqlite`
  - Database lock/contention in nonstandard execution setups
  - Missing `workflow_history` table or schema mismatch
- **Sub-workflow reference:** None.

#### flowEdits
- **Type and role:** `n8n-nodes-base.set` — reshapes workflow metadata into nested fields for later use.
- **Configuration choices:**
  - Creates fields:
    - `Workflow.name`
    - `Workflow.id`
    - `Workflow.published.status`
    - `Workflow.published.latestVersion`
    - `Workflow.published.date`
    - `Workflow.currentVersion.id`
  - Preserves other fields
- **Key expressions/variables used:**
  - `={{ $json.name }}`
  - `={{ $json.id }}`
  - `={{ $json.active ? 'Yes' : 'No' }}`
  - `={{ $json.version_label }}`
  - `={{ $json.activeVersion.createdAt }}`
  - `={{ $json.versionId }}`
- **Input and output connections:**
  - Input from **version_labels**
  - Output to **If**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - `activeVersion` may be absent on some workflows, causing undefined output
  - Expression paths may produce empty values if API response differs
- **Sub-workflow reference:** None.

---

## 2.5 Branch 1 Filtering: Workflows and Nodes with Credentials

**Overview:**  
This block first removes workflows that contain no credential-bearing nodes, then splits each workflow into node-level items and removes nodes that do not define credentials.

**Nodes Involved:**  
- If
- splitNodes
- If1

### Node Details

#### If
- **Type and role:** `n8n-nodes-base.if` — keeps only workflows where at least one node has a `credentials` property.
- **Configuration choices:**
  - Boolean true check on:
    - `$json.nodes.some(node => node.hasOwnProperty('credentials'))`
- **Key expressions/variables used:**
  - `={{ $json.nodes.some(node => node.hasOwnProperty('credentials')) }}`
- **Input and output connections:**
  - Input from **flowEdits**
  - True output to **splitNodes**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases/failures:**
  - If `nodes` is undefined or not an array, the expression fails
- **Sub-workflow reference:** None.

#### splitNodes
- **Type and role:** `n8n-nodes-base.splitOut` — emits one item per workflow node while retaining other fields.
- **Configuration choices:**
  - Field to split out: `nodes`
  - Include: `allOtherFields`
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **If**
  - Output to **If1**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:**
  - Missing `nodes` field
  - Unexpected node object shape
- **Sub-workflow reference:** None.

#### If1
- **Type and role:** `n8n-nodes-base.if` — keeps only node items whose `nodes` object contains a `credentials` field.
- **Configuration choices:**
  - Object existence check on `$json.nodes.credentials`
- **Key expressions/variables used:**
  - `={{ $json.nodes.credentials }}`
- **Input and output connections:**
  - Input from **splitNodes**
  - True output to **nodeEdits**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases/failures:**
  - If the split output shape changes, path may be wrong
- **Sub-workflow reference:** None.

---

## 2.6 Branch 1 Aggregation and Cleanup

**Overview:**  
This block extracts node and credential names, aggregates them by workflow, and converts the summarize node output into cleaner JSON suitable for spreadsheet export.

**Nodes Involved:**  
- nodeEdits
- Summarize
- CleanJson

### Node Details

#### nodeEdits
- **Type and role:** `n8n-nodes-base.set` — converts node-level credential data into a simple structure.
- **Configuration choices:**
  - Includes only selected fields plus `Workflow` and other existing fields
  - Creates:
    - `nodeName`
    - `credentials`
- **Key expressions/variables used:**
  - `={{ $json.nodes.name }}`
  - `={{ Object.values($json.nodes.credentials)[0]?.name || 'None' }}`
- **Input and output connections:**
  - Input from **If1**
  - Output to **Summarize**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - If a node has multiple credential types, only the first credential name is captured
  - If credentials object exists but lacks expected structure, output may be `None`
- **Sub-workflow reference:** None.

#### Summarize
- **Type and role:** `n8n-nodes-base.summarize` — groups rows by workflow and aggregates credential-bearing node names and credential names.
- **Configuration choices:**
  - Output format: separate items
  - Split/group by: `Workflow`
  - Fields summarized:
    - `nodeName` appended
    - `credentials` appended, including empty values
    - `nodeName` counted
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **nodeEdits**
  - Output to **CleanJson**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases/failures:**
  - Grouping by object-like field can produce serialization-dependent output
  - Aggregated append values may be strings requiring later parsing
- **Sub-workflow reference:** None.

#### CleanJson
- **Type and role:** `n8n-nodes-base.code` — parses summarized values, removes duplicates, and prepares export-friendly fields.
- **Configuration choices:**
  - Iterates over all input items
  - Uses helper `safeParse`
  - Deduplicates node names and credential names with `Set`
  - Outputs:
    - `workflow`
    - `nodeCount`
    - `nodes`
    - `credentials`
- **Key expressions/variables used:** Internally references:
  - `data.appended_nodeName`
  - `data.appended_credentials`
  - `data.Workflow`
  - `data.count_nodeName`
- **Input and output connections:**
  - Input from **Summarize**
  - Output to **finalFlowEdits**
- **Version-specific requirements:** Type version `2`.
- **Edge cases/failures:**
  - `JSON.parse` can fail if summarize output is not valid JSON text
  - The code contains unreachable lines after the first `return { json: cleanedCreds };`, so null filtering for arrays after that point never executes
  - Since `nodes` and `credentials` are converted to joined strings before the later array checks, those checks would not help anyway
- **Sub-workflow reference:** None.

---

## 2.7 Branch 1 Export to Google Sheets and Drive

**Overview:**  
This block formats the final workflow inventory rows, sorts them by workflow name, appends them to the spreadsheet, moves the file into the created Drive folder, and records estimated time saved.

**Nodes Involved:**  
- finalFlowEdits
- Sort
- credInventory
- moveFlows
- Time Saved1

### Node Details

#### finalFlowEdits
- **Type and role:** `n8n-nodes-base.set` — final export shaping for the workflow inventory sheet.
- **Configuration choices:**
  - Creates output columns:
    - Workflow
    - ID
    - Published
    - Live Version
    - Last Published
    - Current Version ID
    - Node w/ Creds
    - Nodes
    - Credentials
    - URL
- **Key expressions/variables used:**
  - `={{ $('CleanJson').item.json.workflow.name }}`
  - `='{{ $('CleanJson').item.json.workflow.id }}`
  - `={{ $('CleanJson').item.json.workflow.published.status }}`
  - `={{ $('CleanJson').item.json.workflow.published.latestVersion }}`
  - `='{{ $('CleanJson').item.json.workflow.published.date }}`
  - `='{{ $('CleanJson').item.json.workflow.currentVersion.id }}`
  - `={{ $('CleanJson').item.json.nodeCount }}`
  - `={{ $json.nodes }}`
  - `={{ $json.credentials }}`
  - `=https://{{YOUR WEBHOOK GOES HERE}}/workflow/{{ $('CleanJson').item.json.workflow.id }}`
- **Input and output connections:**
  - Input from **CleanJson**
  - Output to **Sort**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - URL placeholder must be replaced
  - Some values intentionally begin with `'` to preserve formatting in Sheets; current expressions appear malformed and may produce leading literal characters if not corrected
- **Sub-workflow reference:** None.

#### Sort
- **Type and role:** `n8n-nodes-base.sort` — sorts workflow export rows by workflow name.
- **Configuration choices:**
  - Sort field: `Workflow`
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **finalFlowEdits**
  - Output to **credInventory**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** Sorting may be lexicographic rather than natural.
- **Sub-workflow reference:** None.

#### credInventory
- **Type and role:** `n8n-nodes-base.googleSheets` — appends the workflow inventory rows to the generated spreadsheet.
- **Configuration choices:**
  - Operation: append
  - Target spreadsheet ID from **Create credInventory**
  - Sheet name: `Sheet1`
  - Mapping mode: define below, but schema is empty in JSON
- **Key expressions/variables used:**
  - `={{ $('Create credInventory').item.json.spreadsheetId }}`
- **Input and output connections:**
  - Input from **Sort**
  - Output to **moveFlows**
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failures:**
  - If column schema is not auto-detected in UI, append behavior may depend on input structure
  - OAuth2 and permission issues possible
- **Sub-workflow reference:** None.

#### moveFlows
- **Type and role:** `n8n-nodes-base.googleDrive` — moves the workflow inventory spreadsheet into the created folder.
- **Configuration choices:**
  - Operation: move
  - File ID from **Create credInventory**
  - Destination folder ID from **Create folder**
- **Key expressions/variables used:**
  - `={{ $('Create credInventory').item.json.spreadsheetId }}`
  - `={{ $('Create folder').item.json.id }}`
- **Input and output connections:**
  - Input from **credInventory**
  - Output to **Time Saved1**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - File move permissions
  - Cross-drive move restrictions
- **Sub-workflow reference:** None.

#### Time Saved1
- **Type and role:** `n8n-nodes-base.timeSaved` — logs estimated saved time for branch 1.
- **Configuration choices:** `15` minutes saved.
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **moveFlows**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** Informational only.
- **Sub-workflow reference:** None.

---

## 2.8 Branch 2 Setup: Spreadsheet Creation and Credential Retrieval

**Overview:**  
This block creates the credential impact spreadsheet, fetches credentials from the n8n API, and splits the response into one item per credential.

**Nodes Involved:**  
- Create credMap
- credsList
- splitCreds

### Node Details

#### Create credMap
- **Type and role:** `n8n-nodes-base.googleSheets` — creates the spreadsheet for credential-to-workflow mapping.
- **Configuration choices:**
  - Resource: spreadsheet
  - Spreadsheet title includes timestamp
  - Initial sheet: `Sheet1`
- **Key expressions/variables used:**
  - `=Credential-Impact-Map_{{ $now.toISODate() }}_{{ $now.toISOTime().split('.')[0] }}`
- **Input and output connections:**
  - Input from **Create folder**
  - Output to **credsList**
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failures:** Same Google Sheets creation/auth concerns as branch 1.
- **Sub-workflow reference:** None.

#### credsList
- **Type and role:** `n8n-nodes-base.httpRequest` — fetches credential metadata from the n8n API.
- **Configuration choices:**
  - URL points to `/api/v1/credentials`
  - Uses header `X-N8N-API-KEY`
- **Key expressions/variables used:** Static placeholders must be replaced:
  - `https://<WEBHOOK DOMAIN GOES HERE>/api/v1/credentials`
  - `API KEY GOES HERE`
- **Input and output connections:**
  - Input from **Create credMap**
  - Output to **splitCreds**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases/failures:**
  - Invalid API key
  - Wrong instance URL
  - API compatibility differences
- **Sub-workflow reference:** None.

#### splitCreds
- **Type and role:** `n8n-nodes-base.splitOut` — expands the credentials API `data` array.
- **Configuration choices:**
  - Field to split out: `data`
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **credsList**
  - Output to **sortCredNames**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** Missing `data` field in API response.
- **Sub-workflow reference:** None.

---

## 2.9 Branch 2 Enrichment and Export

**Overview:**  
This block sorts credentials, queries SQLite to identify workflows referencing each credential, formats the export rows, writes them to Google Sheets, moves the file to Drive, and records time saved.

**Nodes Involved:**  
- sortCredNames
- flow_names
- credEdits
- credMap
- moveCreds
- Time Saved

### Node Details

#### sortCredNames
- **Type and role:** `n8n-nodes-base.sort` — sorts credentials alphabetically by name.
- **Configuration choices:**
  - Sort field: `name`
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **splitCreds**
  - Output to **flow_names**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** Lexicographic sorting only.
- **Sub-workflow reference:** None.

#### flow_names
- **Type and role:** `n8n-nodes-base.code` — queries SQLite to find workflows whose stored `nodes` JSON contains the credential ID.
- **Configuration choices:**
  - Runs once per item
  - Uses `sqlite3`
  - Reads `/home/node/.n8n/database.sqlite`
  - SQL searches `workflow_entity.nodes` with `LIKE`
  - Returns comma-separated workflow names or `Not in use`
- **Key expressions/variables used:**
  - Credential ID: `$json.id`
  - Search string: `%${$json.id}%`
  - SQL:
    - `SELECT name FROM workflow_entity WHERE nodes LIKE ?`
- **Input and output connections:**
  - Input from **sortCredNames**
  - Output to **credEdits**
- **Version-specific requirements:**
  - Type version `2`
  - Same env vars and module access requirements as **version_labels**
- **Edge cases/failures:**
  - False positives possible because `LIKE` matches raw JSON text rather than exact parsed credential references
  - Database path/schema mismatch
  - Module or permission issues
- **Sub-workflow reference:** None.

#### credEdits
- **Type and role:** `n8n-nodes-base.set` — shapes credential export rows.
- **Configuration choices:**
  - Creates:
    - Credential
    - ID
    - Type
    - Created
    - Updated
    - Workflows
- **Key expressions/variables used:**
  - `={{ $json.name }}`
  - `='{{ $json.id }}`
  - `={{ $json.type }}`
  - `='{{ $json.createdAt }}`
  - `='{{ $json.updatedAt }}`
  - `={{ $json.impacted_workflows }}`
- **Input and output connections:**
  - Input from **flow_names**
  - Output to **credMap**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - As in branch 1, some string expressions appear malformed with embedded quote syntax and may need correction
- **Sub-workflow reference:** None.

#### credMap
- **Type and role:** `n8n-nodes-base.googleSheets` — appends credential impact rows to the second spreadsheet.
- **Configuration choices:**
  - Operation: append
  - Mapping mode: auto-map input data
  - Sheet: `Sheet1`
  - Spreadsheet ID from **Create credMap**
- **Key expressions/variables used:**
  - `={{ $('Create credMap').item.json.spreadsheetId }}`
- **Input and output connections:**
  - Input from **credEdits**
  - Output to **moveCreds**
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failures:**
  - OAuth2/auth issues
  - Header inference in sheet may vary if row structure changes
- **Sub-workflow reference:** None.

#### moveCreds
- **Type and role:** `n8n-nodes-base.googleDrive` — moves the credential impact spreadsheet to the created folder.
- **Configuration choices:**
  - Operation: move
  - File ID from **Create credMap**
  - Folder ID from **Create folder**
- **Key expressions/variables used:**
  - `={{ $('Create credMap').item.json.spreadsheetId }}`
  - `={{ $('Create folder').item.json.id }}`
- **Input and output connections:**
  - Input from **credMap**
  - Output to **Time Saved**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - Same Drive permissions and move restrictions as branch 1
- **Sub-workflow reference:** None.

#### Time Saved
- **Type and role:** `n8n-nodes-base.timeSaved` — logs estimated saved time for branch 2.
- **Configuration choices:** `10` minutes saved.
- **Key expressions/variables used:** None.
- **Input and output connections:**
  - Input from **moveCreds**
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** Informational only.
- **Sub-workflow reference:** None.

---

## 2.10 Documentation and In-Canvas Guidance

**Overview:**  
These nodes are non-executable notes used to document prerequisites, limitations, setup steps, and branch descriptions directly in the workflow canvas.

**Nodes Involved:**  
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note7
- Sticky Note9

### Node Details

#### Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote` — full workflow documentation block.
- **Configuration choices:** Large introductory note covering purpose, limitations, environment variables, API key setup, and branch descriptions.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** Sticky note for branch 2 description.
- **Configuration choices:** Labels the credentials mapping branch.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note5
- **Type and role:** Sticky note for global actions.
- **Configuration choices:** Labels the shared setup area.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note6
- **Type and role:** Sticky note for branch 1 description.
- **Configuration choices:** Labels the workflow credential inventory branch.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note
- **Type and role:** Sticky note indicating manual edits are required.
- **Configuration choices:** “Edit URL and API Key”.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note1
- **Type and role:** Sticky note indicating manual edits are required.
- **Configuration choices:** “Edit URL and API Key”.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note2
- **Type and role:** Sticky note indicating folder setup is required.
- **Configuration choices:** “Edit folder destination and parent folder ID”.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note7
- **Type and role:** Sticky note indicating URL customization is required.
- **Configuration choices:** “Edit URL”.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

#### Sticky Note9
- **Type and role:** Sticky note highlighting runner incompatibility.
- **Configuration choices:** “Disable Runners - If you have runners enabled on your n8n container this workflow will not work.”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Optional manual test trigger |  | Create folder | ## Global Actions |
| If | if | Filter workflows that contain at least one credentialed node | flowEdits | splitNodes | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| If1 | if | Filter nodes without credentials | splitNodes | nodeEdits | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| Summarize | summarize | Aggregate credential-bearing nodes by workflow | nodeEdits | CleanJson | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| Time Saved | timeSaved | Record estimated time saved for credential impact map branch | moveCreds |  | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| Time Saved1 | timeSaved | Record estimated time saved for workflow inventory branch | moveFlows |  | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| credsList | httpRequest | Fetch credential list from n8n API | Create credMap | splitCreds | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one<br>## Edit URL and API Key |
| flowsList | httpRequest | Fetch workflow list from n8n API | Create credInventory | splitFlows | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet<br>## Edit URL and API Key |
| version_labels | code | Query SQLite for latest workflow version label | splitFlows | flowEdits | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| flow_names | code | Query SQLite for workflows referencing a credential | sortCredNames | credEdits | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| CleanJson | code | Parse summarize output and deduplicate node/credential names | Summarize | finalFlowEdits | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| credInventory | googleSheets | Append workflow inventory rows to spreadsheet | Sort | moveFlows | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| credEdits | set | Format credential impact output rows | flow_names | credMap | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| credMap | googleSheets | Append credential impact rows to spreadsheet | credEdits | moveCreds | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| Create credInventory | googleSheets | Create workflow inventory spreadsheet | Create folder | flowsList | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| Create credMap | googleSheets | Create credential impact spreadsheet | Create folder | credsList | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| splitFlows | splitOut | Split workflow API array into individual items | flowsList | version_labels | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| splitCreds | splitOut | Split credential API array into individual items | credsList | sortCredNames | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| splitNodes | splitOut | Split workflow nodes into node-level items | If | If1 | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| sortCredNames | sort | Sort credentials alphabetically | splitCreds | flow_names | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| nodeEdits | set | Extract node and credential names for aggregation | If1 | Summarize | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| finalFlowEdits | set | Final format for workflow inventory export | CleanJson | Sort | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet<br>## Edit URL |
| flowEdits | set | Normalize workflow metadata fields | version_labels | If | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| Create folder | googleDrive | Create destination folder in Google Drive | Schedule Trigger, When clicking ‘Execute workflow’ | Create credMap, Create credInventory | ## Global Actions<br>## Edit folder destination and parent folder ID |
| moveCreds | googleDrive | Move credential impact spreadsheet to created folder | credMap | Time Saved | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| moveFlows | googleDrive | Move workflow inventory spreadsheet to created folder | credInventory | Time Saved1 | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| Sticky Note3 | stickyNote | Embedded workflow documentation |  |  | # Workflow Introduction  |
| Sticky Note4 | stickyNote | Branch 2 label/documentation |  |  | ## Branch 2 - Credentials Mapping<br>This part of the workflow grabs your credentials and the associated workflows and generates a google sheet<br>- Lists all credentials and all the workflows using each one |
| Sticky Note5 | stickyNote | Global section label |  |  | ## Global Actions |
| Sticky Note6 | stickyNote | Branch 1 label/documentation |  |  | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |
| Sticky Note | stickyNote | Manual configuration reminder for API nodes |  |  | ## Edit URL and API Key |
| Sticky Note1 | stickyNote | Manual configuration reminder for API nodes |  |  | ## Edit URL and API Key |
| Sticky Note2 | stickyNote | Manual configuration reminder for Drive folder setup |  |  | ## Edit folder destination and parent folder ID |
| Sticky Note7 | stickyNote | Manual configuration reminder for workflow URL output |  |  | ## Edit URL |
| Schedule Trigger | scheduleTrigger | Weekly production trigger |  | Create folder | ## Global Actions |
| Sticky Note9 | stickyNote | Environment/runtime warning about runners |  |  | ## Disable Runners - If you have runners enabled on your n8n container this workflow will not work. |
| Sort | sort | Sort workflow inventory rows by workflow name | finalFlowEdits | credInventory | ## Branch 1 - Workflow Credentials Inventory<br>This part of the workflow grabs all your workflows that have credentials attached and all the credentials involved in each one, and generates a google sheet |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**.
   - Type: `Schedule Trigger`
   - Configure weekly execution:
     - Interval unit: `weeks`
     - Day: Friday
     - Hour: 14
   - This is the main production trigger.

3. **Add a Manual Trigger node** for testing.
   - Type: `Manual Trigger`
   - Leave default settings.
   - Disable it if you want parity with the provided workflow.

4. **Add a Google Drive node named `Create folder`**.
   - Resource: `Folder`
   - Operation: create
   - Name:
     - `={{ $now.toISODate() }}_{{ $now.toISOTime().split('.')[0] }}`
   - Drive: `My Drive`
   - Parent folder ID: set your destination folder ID
   - Optional folder color: `#F18F17`
   - Credentials: Google Drive OAuth2
   - Connect both triggers to this node.

5. **Create Google credentials in n8n**.
   - One **Google Drive OAuth2** credential
   - One **Google Sheets OAuth2** credential
   - Ensure the authenticated Google account can create and move files in the target Drive location.

6. **Create Branch 1 spreadsheet node: `Create credInventory`**.
   - Type: `Google Sheets`
   - Resource: `Spreadsheet`
   - Operation: create spreadsheet
   - Title:
     - `=Workflow-Credentials-Inventory_{{ $now.toISODate() }}_{{ $now.toISOTime().split('.')[0] }}`
   - Initial sheet name: `Sheet1`
   - Credentials: Google Sheets OAuth2
   - Connect `Create folder` → `Create credInventory`.

7. **Create Branch 2 spreadsheet node: `Create credMap`**.
   - Type: `Google Sheets`
   - Resource: `Spreadsheet`
   - Operation: create spreadsheet
   - Title:
     - `=Credential-Impact-Map_{{ $now.toISODate() }}_{{ $now.toISOTime().split('.')[0] }}`
   - Initial sheet name: `Sheet1`
   - Credentials: Google Sheets OAuth2
   - Connect `Create folder` → `Create credMap`.

8. **Prepare n8n API access**.
   - In your n8n instance, create an API key under Settings → API.
   - Either:
     - put the header directly in HTTP Request nodes, or
     - create a Header Auth credential
   - Required header:
     - `X-N8N-API-KEY: <your-key>`

9. **Add HTTP Request node `flowsList`**.
   - Method: GET
   - URL: `https://<your-domain>/api/v1/workflows` or `http://localhost:5678/api/v1/workflows`
   - Send Headers: enabled
   - Header:
     - Name: `X-N8N-API-KEY`
     - Value: your API key
   - Connect `Create credInventory` → `flowsList`.

10. **Add Split Out node `splitFlows`**.
    - Field to split out: `data`
    - Connect `flowsList` → `splitFlows`.

11. **Enable Code node external access in your n8n runtime**.
    - Required environment variables:
      - `NODE_FUNCTION_ALLOW_BUILTIN=*`
      - `NODE_FUNCTION_ALLOW_EXTERNAL=sqlite3`
    - Ensure the `sqlite3` package is available in the runtime.
    - Ensure the n8n database file is reachable at `/home/node/.n8n/database.sqlite`, or adapt the code if your path differs.
    - Do not use runners for this workflow if that prevents local DB access.

12. **Add Code node `version_labels`**.
    - Mode: `Run once for each item`
    - Paste logic that:
      - opens SQLite in read-only mode
      - queries `workflow_history`
      - fetches the latest non-null `name` for the current workflow ID
      - returns it as `version_label`
    - Use workflow ID from `$json.id`.
    - Connect `splitFlows` → `version_labels`.

13. **Add Set node `flowEdits`**.
    - Keep other fields.
    - Add fields:
      - `Workflow.name = {{$json.name}}`
      - `Workflow.id = {{$json.id}}`
      - `Workflow.published.status = {{$json.active ? 'Yes' : 'No'}}`
      - `Workflow.published.latestVersion = {{$json.version_label}}`
      - `Workflow.published.date = {{$json.activeVersion.createdAt}}`
      - `Workflow.currentVersion.id = {{$json.versionId}}`
    - Connect `version_labels` → `flowEdits`.

14. **Add If node `If`**.
    - Condition type: boolean
    - Expression:
      - `{{$json.nodes.some(node => node.hasOwnProperty('credentials'))}}`
    - Route only `true` output forward.
    - Connect `flowEdits` → `If`.

15. **Add Split Out node `splitNodes`**.
    - Field to split out: `nodes`
    - Include: all other fields
    - Connect `If` true → `splitNodes`.

16. **Add If node `If1`**.
    - Condition: object exists
    - Left value:
      - `{{$json.nodes.credentials}}`
    - Route only `true` output forward.
    - Connect `splitNodes` → `If1`.

17. **Add Set node `nodeEdits`**.
    - Include selected fields, while preserving `Workflow`
    - Add:
      - `nodeName = {{$json.nodes.name}}`
      - `credentials = {{Object.values($json.nodes.credentials)[0]?.name || 'None'}}`
    - Connect `If1` true → `nodeEdits`.

18. **Add Summarize node `Summarize`**.
    - Group/split by: `Workflow`
    - Output format: separate items
    - Summaries:
      - append `nodeName`
      - append `credentials`
      - count `nodeName`
    - Connect `nodeEdits` → `Summarize`.

19. **Add Code node `CleanJson`**.
    - Use code to:
      - parse appended arrays if needed
      - deduplicate node names and credential names
      - produce:
        - `workflow`
        - `nodeCount`
        - `nodes`
        - `credentials`
    - Connect `Summarize` → `CleanJson`.
   - Recommended improvement: fix the unreachable code and perform null filtering before joining arrays.

20. **Add Set node `finalFlowEdits`**.
    - Create export columns:
      - Workflow
      - ID
      - Published
      - Live Version
      - Last Published
      - Current Version ID
      - Node w/ Creds
      - Nodes
      - Credentials
      - URL
    - Use `CleanJson` outputs and build a workflow URL like:
      - `https://<your-n8n-domain>/workflow/{{$('CleanJson').item.json.workflow.id}}`
    - Connect `CleanJson` → `finalFlowEdits`.

21. **Add Sort node `Sort`**.
    - Sort by field: `Workflow`
    - Connect `finalFlowEdits` → `Sort`.

22. **Add Google Sheets node `credInventory`**.
    - Operation: `Append`
    - Spreadsheet ID:
      - `{{$('Create credInventory').item.json.spreadsheetId}}`
    - Sheet name: `Sheet1`
    - Use either auto-map or explicit mapping for the columns from `finalFlowEdits`
    - Credentials: Google Sheets OAuth2
    - Connect `Sort` → `credInventory`.

23. **Add Google Drive node `moveFlows`**.
    - Operation: `Move`
    - File ID:
      - `{{$('Create credInventory').item.json.spreadsheetId}}`
    - Destination folder ID:
      - `{{$('Create folder').item.json.id}}`
    - Drive: `My Drive`
    - Connect `credInventory` → `moveFlows`.

24. **Add Time Saved node `Time Saved1`**.
    - Minutes saved: `15`
    - Connect `moveFlows` → `Time Saved1`.

25. **Add HTTP Request node `credsList`**.
    - Method: GET
    - URL: `https://<your-domain>/api/v1/credentials` or `http://localhost:5678/api/v1/credentials`
    - Send Headers: enabled
    - Header:
      - `X-N8N-API-KEY: <your-key>`
    - Connect `Create credMap` → `credsList`.

26. **Add Split Out node `splitCreds`**.
    - Field to split out: `data`
    - Connect `credsList` → `splitCreds`.

27. **Add Sort node `sortCredNames`**.
    - Sort by field: `name`
    - Connect `splitCreds` → `sortCredNames`.

28. **Add Code node `flow_names`**.
    - Mode: `Run once for each item`
    - Use SQLite code that:
      - opens the database read-only
      - searches `workflow_entity.nodes` for the current credential ID using SQL `LIKE`
      - joins matching workflow names into one string
      - outputs as `impacted_workflows`
    - Connect `sortCredNames` → `flow_names`.
   - Recommended improvement: replace `LIKE` with structured JSON analysis if available in your environment to reduce false positives.

29. **Add Set node `credEdits`**.
    - Add export fields:
      - `Credential = {{$json.name}}`
      - `ID = {{$json.id}}`
      - `Type = {{$json.type}}`
      - `Created = {{$json.createdAt}}`
      - `Updated = {{$json.updatedAt}}`
      - `Workflows = {{$json.impacted_workflows}}`
    - Connect `flow_names` → `credEdits`.

30. **Add Google Sheets node `credMap`**.
    - Operation: `Append`
    - Spreadsheet ID:
      - `{{$('Create credMap').item.json.spreadsheetId}}`
    - Sheet name: `Sheet1`
    - Mapping mode: auto-map input data
    - Credentials: Google Sheets OAuth2
    - Connect `credEdits` → `credMap`.

31. **Add Google Drive node `moveCreds`**.
    - Operation: `Move`
    - File ID:
      - `{{$('Create credMap').item.json.spreadsheetId}}`
    - Destination folder ID:
      - `{{$('Create folder').item.json.id}}`
    - Drive: `My Drive`
    - Connect `credMap` → `moveCreds`.

32. **Add Time Saved node `Time Saved`**.
    - Minutes saved: `10`
    - Connect `moveCreds` → `Time Saved`.

33. **Add canvas notes if desired** to mirror the original operational guidance:
    - workflow introduction
    - branch labels
    - “Edit URL and API Key”
    - “Edit folder destination and parent folder ID”
    - “Disable Runners”

34. **Replace all placeholders** before activation:
    - workflow API base URL
    - credentials API base URL
    - workflow URL used in sheet output
    - Google Drive parent folder ID
    - API key if directly inserted into nodes

35. **Test the workflow manually**.
    - First enable the Manual Trigger if needed.
    - Confirm:
      - folder is created
      - both spreadsheets are created
      - rows are appended
      - files are moved
      - Code nodes can access SQLite successfully

36. **Activate the workflow** once validated.

### Required credentials and configuration summary
- **Google Drive OAuth2**: used by `Create folder`, `moveFlows`, `moveCreds`
- **Google Sheets OAuth2**: used by `Create credInventory`, `credInventory`, `Create credMap`, `credMap`
- **n8n API key**: used by `flowsList` and `credsList`
- **Runtime access to sqlite3**: mandatory for `version_labels` and `flow_names`

### Input/output expectations for the two code nodes
- **version_labels**
  - Input: one workflow item from `/api/v1/workflows`
  - Output: same item plus `version_label`
- **flow_names**
  - Input: one credential item from `/api/v1/credentials`
  - Output: same item plus `impacted_workflows`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is specifically for Self-Hosted N8N Docker users. | Workflow scope |
| It audits credential usage without exposing encrypted credential secrets. | Functional design note |
| It is optimized for Standard n8n Execution Mode and is not compatible with Task Runners (`N8N_RUNNERS_ENABLED=true`). | Runtime limitation |
| Required environment variables for Code nodes: `NODE_FUNCTION_ALLOW_BUILTIN=*` and `NODE_FUNCTION_ALLOW_EXTERNAL=sqlite3` | Runtime requirement |
| If you need multiple external modules, the note suggests a comma-separated value such as `NODE_FUNCTION_ALLOW_EXTERNAL=sqlite3, better-sqlite3` | Runtime requirement |
| Required credentials: n8n self-hosted API key, Google Sheets account, Google Drive account | Setup requirement |
| The nodes requiring manual parameter review are: Create folder, flowsList, credsList, and finalFlowEdits | Setup reminder |
| API key location: Settings → API → Create API key | n8n instance administration |
| Suggested workflow endpoint: `http://localhost:5678/api/v1/workflows` or `https://<your webhook domain>/api/v1/workflows` | flowsList |
| Suggested credentials endpoint: `http://localhost:5678/api/v1/credentials` or `https://<your webhook domain>/api/v1/credentials` | credsList |
| If API key is directly inserted in HTTP Request nodes, that key may not appear as connected in the generated Credential Impact Map. | Known limitation |
| Disable runners if enabled on the n8n container; otherwise the workflow may not work. | Operational warning |