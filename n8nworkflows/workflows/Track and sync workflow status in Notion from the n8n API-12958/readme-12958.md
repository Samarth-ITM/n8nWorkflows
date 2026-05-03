Track and sync workflow status in Notion from the n8n API

https://n8nworkflows.xyz/workflows/track-and-sync-workflow-status-in-notion-from-the-n8n-api-12958


# Track and sync workflow status in Notion from the n8n API

## 1. Workflow Overview

**Purpose:** Maintain a centralized inventory of all workflows in an n8n instance inside a Notion database. On a daily schedule, it pulls workflow metadata via the **n8n API**, normalizes status values, and then **creates or updates** corresponding pages in **Notion**, using the **n8n Workflow ID** as the unique key.

**Primary use cases**
- Track all n8n workflows (name, active status, created/edited timestamps) in Notion.
- Prevent duplicates by syncing on workflow ID.
- Provide non-technical stakeholders a Notion-based “registry” of automations.

### 1.1 Scheduled Input & Fetch from n8n
Runs daily, queries the n8n API for all workflows.

### 1.2 Batch Iteration
Processes workflows one-by-one using Split in Batches.

### 1.3 Data Mapping & Normalization
Maps workflow fields to a consistent schema and converts status to **“Active”** / **“Deactivated”**.

### 1.4 Notion Lookup (by workflow ID)
Searches Notion database for an existing page matching the workflow ID.

### 1.5 Upsert (Update or Create) in Notion
If found → update page; else → create a new page; then continue batching.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Input & Fetch from n8n
**Overview:** Triggers daily and retrieves the full list of workflows from the n8n instance via the n8n API node.

**Nodes involved**
- **Daily Trigger**
- **Fetch All n8n Workflows**

#### Node: Daily Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs on a daily interval (rule uses interval-based schedule).
- **Outputs:** Sends an empty/schedule payload to **Fetch All n8n Workflows**.
- **Edge cases / failures:**
  - n8n instance time zone may affect when “daily” occurs.
  - If n8n is down at trigger time, execution won’t happen (unless external scheduling is used).

#### Node: Fetch All n8n Workflows
- **Type / role:** `n8n` node (n8n API) — fetch workflows from the same/connected instance.
- **Configuration (interpreted):**
  - Operation: fetch all workflows (no filters set).
  - Uses **n8n API credentials** (`n8n account`).
- **Input:** From **Daily Trigger**.
- **Output:** A list/collection of workflow objects (each with fields like `id`, `name`, `active`, `createdAt`, `updatedAt`) to **Iterate Workflows**.
- **Version-specific notes:** Node `typeVersion: 1`.
- **Edge cases / failures:**
  - Auth failures (expired API key / invalid base URL).
  - Instance permissions: credential must be able to list workflows.
  - Large instances: response size/timeouts depending on API limits and n8n settings.

---

### Block 2 — Batch Iteration
**Overview:** Iterates through returned workflows sequentially, enabling controlled processing and easy looping back after update/create.

**Nodes involved**
- **Iterate Workflows**

#### Node: Iterate Workflows
- **Type / role:** `SplitInBatches` — iterator/loop controller.
- **Configuration (interpreted):**
  - Uses default options; batch size is not explicitly set in the JSON (n8n default applies).
  - **Connection pattern indicates loopback:** after each Notion create/update, it routes back into this node to request the next item.
- **Input:** From **Fetch All n8n Workflows**; also loopbacks from **Create Workflow Page** and **Update Workflow Page**.
- **Outputs:**
  - **Output 1 (index 0):** Not used (left empty in connections).
  - **Output 2 (index 1):** Sends the current batch item to **Map Workflow Metadata**.
- **Version-specific notes:** Node `typeVersion: 3`.
- **Edge cases / failures:**
  - If batch size is too large, downstream Notion calls can hit rate limits.
  - If the loopback is miswired, it can stop after the first batch or create infinite loops (here it is correctly looped via create/update).

---

### Block 3 — Data Mapping & Normalization
**Overview:** Transforms the workflow object into fields expected by Notion and standardizes the active flag into Notion-friendly status text.

**Nodes involved**
- **Map Workflow Metadata**
- **Normalize Workflow Status**

#### Node: Map Workflow Metadata
- **Type / role:** `Set` — maps and renames fields.
- **Configuration (interpreted):** Creates these fields per workflow:
  - `Workflow Name` ← `{{$json.name}}`
  - `Status` (boolean) ← `{{$json.active}}`
  - `ID` ← `{{$json.id}}`
  - `Created At` ← `{{$json.createdAt}}`
  - `Updated At` ← `{{$json.updatedAt}}`
- **Input:** From **Iterate Workflows** (output index 1).
- **Output:** To **Normalize Workflow Status**.
- **Version-specific notes:** `typeVersion: 3.4`.
- **Edge cases / failures:**
  - If n8n API shape changes (field names differ), mappings break.
  - Date strings must be compatible with Notion date property parsing (usually ISO-8601 works).

#### Node: Normalize Workflow Status
- **Type / role:** `Code` — normalizes boolean-ish values to fixed strings.
- **Configuration (interpreted):**
  - Runs once per item.
  - JS logic treats `true`, `"true"`, `1`, `"1"` as active.
  - Outputs: same object, but `Status` becomes `"Active"` or `"Deactivated"`.
- **Key code behavior:**
  - `return { ...$json, Status: isActive ? "Active" : "Deactivated" };`
- **Input:** From **Map Workflow Metadata**.
- **Output:** To **Find Workflow in Notion (by ID)**.
- **Version-specific notes:** `typeVersion: 2`.
- **Edge cases / failures:**
  - If `Status` is missing/null, it becomes `"Deactivated"`.
  - If Notion “Status” options don’t exactly match `"Active"` and `"Deactivated"`, create/update will fail with validation errors.

---

### Block 4 — Notion Lookup (by workflow ID)
**Overview:** Searches the Notion database for a page whose `ID` property matches the n8n workflow ID. Outputs results for decision-making.

**Nodes involved**
- **Find Workflow in Notion (by ID)**
- **Workflow Exists ?**

#### Node: Find Workflow in Notion (by ID)
- **Type / role:** `Notion` — query database pages.
- **Configuration (interpreted):**
  - Resource: Database Page
  - Operation: Get All (query)
  - Database: **Workflows** (`2848fa9d-4c4d-80cb-b593-cd9595a93464`)
  - Manual filter: `ID` (rich_text) **equals** `{{$json.ID}}`
  - Return all matches: enabled
  - **Always Output Data:** true (important for downstream consistency)
- **Input:** From **Normalize Workflow Status**.
- **Output:** To **Workflow Exists ?**
- **Version-specific notes:** `typeVersion: 2.2`.
- **Edge cases / failures:**
  - Notion auth/scopes missing (database read access).
  - Property type mismatch: Notion property must be `rich_text` named exactly `ID`.
  - Multiple matches (duplicate IDs) will return multiple pages; downstream logic assumes a single `id` exists but does not explicitly select the first.

#### Node: Workflow Exists ?
- **Type / role:** `IF` — branches based on whether a Notion page was found.
- **Configuration (interpreted):**
  - Condition: string **exists** on `{{$json.id}}`
  - If **true** → Update existing page
  - If **false** → Create new page
- **Input:** From **Find Workflow in Notion (by ID)**.
- **Outputs:**
  - True branch → **Update Workflow Page**
  - False branch → **Create Workflow Page**
- **Version-specific notes:** `typeVersion: 2.3`.
- **Edge cases / failures:**
  - If Notion returns an array and the item structure isn’t what’s expected, `$json.id` may not exist (leading to always-false and unintended page creation).
  - If multiple pages match, `$json.id` could refer to one item depending on n8n itemization; duplicates should be prevented at the database level.

---

### Block 5 — Upsert (Update or Create) in Notion + Loopback
**Overview:** Updates the found Notion page or creates a new one, then loops back to process the next workflow.

**Nodes involved**
- **Update Workflow Page**
- **Create Workflow Page**

#### Node: Update Workflow Page
- **Type / role:** `Notion` — update database page properties.
- **Configuration (interpreted):**
  - Operation: Update database page
  - Page ID: `{{$json.id}}` (the Notion page id from the lookup result)
  - Properties updated:
    - `Status` (status) ← `{{ $('Normalize Workflow Status').item.json.Status }}`
    - `Name` (title) ← `{{ $('Normalize Workflow Status').item.json['Workflow Name'] }}`
    - `Created` (date) ← `{{ $('Normalize Workflow Status').item.json['Created At'] }}`
    - `Edited` (date) ← `{{ $('Normalize Workflow Status').item.json['Updated At'] }}`
- **Input:** True branch from **Workflow Exists ?**.
- **Output:** Loops back to **Iterate Workflows** (to fetch next workflow).
- **Version-specific notes:** `typeVersion: 2.2`.
- **Edge cases / failures:**
  - Notion validation errors if properties don’t exist or types differ (e.g., `Status` not a status property).
  - Date parsing errors if timestamps aren’t in an accepted format.
  - If the lookup returned multiple pages, this updates whichever item is being processed, possibly leaving duplicates.

#### Node: Create Workflow Page
- **Type / role:** `Notion` — create a new database page.
- **Configuration (interpreted):**
  - Operation: Create database page in **Workflows** database
  - Title: workflow name
  - Icon: ⚙️
  - Properties set:
    - `Status` (status) ← normalized status
    - `ID` (rich_text) ← workflow ID
    - `Created` (date) ← createdAt
    - `Edited` (date) ← updatedAt
- **Input:** False branch from **Workflow Exists ?**.
- **Output:** Loops back to **Iterate Workflows**.
- **Version-specific notes:** `typeVersion: 2.2`.
- **Edge cases / failures:**
  - If “ID” property is missing or not `rich_text`, creation fails.
  - Rate limiting on Notion API if many workflows are created/updated in a single run.
  - If normalization outputs a status value not configured in Notion, creation fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Trigger | Schedule Trigger | Daily workflow start | — | Fetch All n8n Workflows | ## Workflow Overview<br><br>This workflow creates a centralized inventory of all n8n workflows in Notion.<br><br>It runs on a daily schedule and syncs workflow metadata from n8n<br>into a Notion database, allowing you to track workflow status,<br>creation date, and last modification date in one place.<br><br>### How it works<br>• Fetches all workflows from the n8n instance<br>• Processes workflows one by one<br>• Normalizes workflow status (Active / Deactivated)<br>• Checks if the workflow already exists in Notion<br>• Creates or updates Notion pages accordingly<br><br>### Setup steps<br>1. Connect your n8n API credentials<br>2. Connect your Notion account<br>3. Prepare your Notion database<br>Your database must include:<br>• Name (title)<br>• ID (text)<br>• Status (status) with these options : ( Active , Deactivated )<br>• Created (date)<br>• Edited (date)<br>4. Activate the workflow<br>5. Let it run on the daily schedule<br><br>This workflow is read-only and does not manage or modify n8n workflows. |
| Fetch All n8n Workflows | n8n (API) | List workflows from n8n | Daily Trigger | Iterate Workflows | ## Fetch workflows<br><br>Fetches all workflows from the n8n instance<br>and processes them one by one. |
| Iterate Workflows | SplitInBatches | Iterate through workflows with loopback | Fetch All n8n Workflows; Create Workflow Page; Update Workflow Page | Map Workflow Metadata (output 1 index=1) | ## Fetch workflows<br><br>Fetches all workflows from the n8n instance<br>and processes them one by one. |
| Map Workflow Metadata | Set | Map API fields to Notion-ready names | Iterate Workflows | Normalize Workflow Status | ## Normalize workflow data<br><br>Maps workflow fields and normalizes<br>the workflow status into readable values. |
| Normalize Workflow Status | Code | Convert status to “Active/Deactivated” | Map Workflow Metadata | Find Workflow in Notion (by ID) | ## Normalize workflow data<br><br>Maps workflow fields and normalizes<br>the workflow status into readable values. |
| Find Workflow in Notion (by ID) | Notion | Query Notion DB for matching workflow ID | Normalize Workflow Status | Workflow Exists ? | ## Check existing workflows<br><br>Uses the workflow ID to determine<br>whether the workflow already exists in Notion. |
| Workflow Exists ? | IF | Branch: update if found else create | Find Workflow in Notion (by ID) | Update Workflow Page; Create Workflow Page | ## 🔄 Sync Strategy<br>The workflow uses the n8n Workflow ID<br>as the unique identifier.<br><br>Logic:<br>• If the workflow ID exists in Notion → Update page<br>• If it does not exist → Create new page<br><br>This prevents duplicate entries<br>and keeps the database in sync. |
| Update Workflow Page | Notion | Update existing Notion page properties | Workflow Exists ? (true) | Iterate Workflows | ## 🔄 Sync Strategy<br>The workflow uses the n8n Workflow ID<br>as the unique identifier.<br><br>Logic:<br>• If the workflow ID exists in Notion → Update page<br>• If it does not exist → Create new page<br><br>This prevents duplicate entries<br>and keeps the database in sync. |
| Create Workflow Page | Notion | Create new Notion page for workflow | Workflow Exists ? (false) | Iterate Workflows | ## 🔄 Sync Strategy<br>The workflow uses the n8n Workflow ID<br>as the unique identifier.<br><br>Logic:<br>• If the workflow ID exists in Notion → Update page<br>• If it does not exist → Create new page<br><br>This prevents duplicate entries<br>and keeps the database in sync. |
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## Workflow Overview<br><br>This workflow creates a centralized inventory of all n8n workflows in Notion.<br><br>It runs on a daily schedule and syncs workflow metadata from n8n<br>into a Notion database, allowing you to track workflow status,<br>creation date, and last modification date in one place.<br><br>### How it works<br>• Fetches all workflows from the n8n instance<br>• Processes workflows one by one<br>• Normalizes workflow status (Active / Deactivated)<br>• Checks if the workflow already exists in Notion<br>• Creates or updates Notion pages accordingly<br><br>### Setup steps<br>1. Connect your n8n API credentials<br>2. Connect your Notion account<br>3. Prepare your Notion database<br>Your database must include:<br>• Name (title)<br>• ID (text)<br>• Status (status) with these options : ( Active , Deactivated )<br>• Created (date)<br>• Edited (date)<br>4. Activate the workflow<br>5. Let it run on the daily schedule<br><br>This workflow is read-only and does not manage or modify n8n workflows. |
| Sticky Note1 | Sticky Note | Documentation / block label | — | — | ## Fetch workflows<br><br>Fetches all workflows from the n8n instance<br>and processes them one by one. |
| Sticky Note2 | Sticky Note | Documentation / sync strategy | — | — | ## 🔄 Sync Strategy<br>The workflow uses the n8n Workflow ID<br>as the unique identifier.<br><br>Logic:<br>• If the workflow ID exists in Notion → Update page<br>• If it does not exist → Create new page<br><br>This prevents duplicate entries<br>and keeps the database in sync. |
| Sticky Note3 | Sticky Note | Documentation / normalization | — | — | ## Normalize workflow data<br><br>Maps workflow fields and normalizes<br>the workflow status into readable values. |
| Sticky Note4 | Sticky Note | Documentation / author link | — | — | ## 👋 Need help with n8n automations?<br><br>## 🌐 https://abdallahhussein.com<br>## Automation & Workflow Consulting |
| Sticky Note5 | Sticky Note | Documentation / lookup | — | — | ## Check existing workflows<br><br>Uses the workflow ID to determine<br>whether the workflow already exists in Notion. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Notion database**
   1. Create a database named (example) **Workflows**.
   2. Add properties (names must match exactly, including capitalization/spaces):
      - **Name** — *Title*
      - **ID** — *Rich text*
      - **Status** — *Status* with options **Active** and **Deactivated**
      - **Created** — *Date*
      - **Edited** — *Date*

2. **Create credentials**
   1. **Notion API credential**
      - Create an integration in Notion, copy token into n8n Notion credential.
      - Share the Workflows database with the integration.
   2. **n8n API credential**
      - Configure an n8n API credential that can list workflows on the target instance (base URL + API key/token as required by your setup).

3. **Build the workflow nodes (in order)**
   1. Add **Schedule Trigger** node named **Daily Trigger**
      - Configure it for **daily** execution (default daily interval is acceptable).
   2. Add **n8n** node named **Fetch All n8n Workflows**
      - Operation: list/get workflows (no filters).
      - Select your **n8n API** credential.
      - Connect: **Daily Trigger → Fetch All n8n Workflows**.
   3. Add **Split In Batches** node named **Iterate Workflows**
      - Leave options default (or set a batch size if you expect many workflows).
      - Connect: **Fetch All n8n Workflows → Iterate Workflows**.
   4. Add **Set** node named **Map Workflow Metadata**
      - Add fields:
        - `Workflow Name` = `{{$json.name}}`
        - `Status` = `{{$json.active}}`
        - `ID` = `{{$json.id}}`
        - `Created At` = `{{$json.createdAt}}`
        - `Updated At` = `{{$json.updatedAt}}`
      - Connect: **Iterate Workflows (output 2 / “next batch item”) → Map Workflow Metadata**.
   5. Add **Code** node named **Normalize Workflow Status**
      - Mode: *Run Once for Each Item*
      - Code:
        ```js
        const status = $json.Status;

        const isActive =
          status === true ||
          status === "true" ||
          status === 1 ||
          status === "1";

        return {
          ...$json,
          Status: isActive ? "Active" : "Deactivated"
        };
        ```
      - Connect: **Map Workflow Metadata → Normalize Workflow Status**.
   6. Add **Notion** node named **Find Workflow in Notion (by ID)**
      - Resource: *Database Page*
      - Operation: *Get All*
      - Database: select your **Workflows** database
      - Filter: `ID` (rich_text) **equals** `{{$json.ID}}`
      - Enable **Return All**
      - Enable **Always Output Data**
      - Connect: **Normalize Workflow Status → Find Workflow in Notion (by ID)**.
   7. Add **IF** node named **Workflow Exists ?**
      - Condition: String → **exists** on `{{$json.id}}`
      - Connect: **Find Workflow in Notion (by ID) → Workflow Exists ?**
   8. Add **Notion** node named **Update Workflow Page**
      - Resource: *Database Page*
      - Operation: *Update*
      - Page ID: `{{$json.id}}`
      - Properties:
        - `Status` = `{{ $('Normalize Workflow Status').item.json.Status }}`
        - `Name` = `{{ $('Normalize Workflow Status').item.json['Workflow Name'] }}`
        - `Created` = `{{ $('Normalize Workflow Status').item.json['Created At'] }}`
        - `Edited` = `{{ $('Normalize Workflow Status').item.json['Updated At'] }}`
      - Connect: **Workflow Exists ? (true) → Update Workflow Page**
   9. Add **Notion** node named **Create Workflow Page**
      - Resource: *Database Page*
      - Operation: *Create*
      - Database: **Workflows**
      - Title: `{{ $('Normalize Workflow Status').item.json['Workflow Name'] }}`
      - Optional icon: `⚙️`
      - Properties:
        - `Status` = normalized status
        - `ID` = workflow ID
        - `Created` = created date
        - `Edited` = updated date
      - Connect: **Workflow Exists ? (false) → Create Workflow Page**
   10. **Close the loop**
       - Connect **Update Workflow Page → Iterate Workflows** (to request the next item).
       - Connect **Create Workflow Page → Iterate Workflows**.

4. **Activate and validate**
   - Run once manually to ensure:
     - Notion database properties match types and names.
     - Status options exactly match **Active** / **Deactivated**.
     - Items are not duplicating (if they do, investigate Notion query output structure and duplicates in DB).

5. **(Optional but recommended) Error handling**
   - The workflow references an **error workflow** in settings (`errorWorkflow: vPTHs8q7dSToQ0VR`). Ensure your instance has an error workflow configured, or set one in workflow settings to capture Notion/n8n API errors.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Need help with n8n automations? | https://abdallahhussein.com |
| Automation & Workflow Consulting | (same link above) |
| Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided by the requester (workflow documentation context). |