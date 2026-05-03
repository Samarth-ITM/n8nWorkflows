Back up self-hosted workflows to Google Drive daily with change detection

https://n8nworkflows.xyz/workflows/back-up-self-hosted-workflows-to-google-drive-daily-with-change-detection-12242


# Back up self-hosted workflows to Google Drive daily with change detection

## 1. Workflow Overview

**Purpose:**  
This workflow backs up workflows from a **self-hosted n8n instance** to **Google Drive** on a daily schedule. It uses **content hashing** to detect meaningful changes and only uploads a new backup when a workflow is new or has changed.

**Target use cases:**
- Daily disaster-recovery backup of n8n workflows
- Auditable version snapshots (one file per workflow, always up-to-date)
- Reducing Drive clutter by avoiding duplicate uploads when nothing changed

**Logical blocks (based on dependencies):**
1.1 **Trigger, Configuration & Validation**  
1.2 **Workflow Discovery & Iteration**  
1.3 **Backup Scope Filtering (all / active / tagged)**  
1.4 **Normalization, Hashing & Change Detection (Data Table index)**  
1.5 **Google Drive Backup + Data Table Upsert (index maintenance)**  
1.6 **Documentation / Notes (Sticky Notes)**

---

## 2. Block-by-Block Analysis

### 2.1 Trigger, Configuration & Validation

**Overview:**  
Runs once per day, loads configuration values, and validates required settings before making API calls.

**Nodes involved:**
- Daily Backup Trigger
- Set Configuration
- Config Validation

#### Node: Daily Backup Trigger
- **Type / role:** `Schedule Trigger` — starts the workflow on a schedule.
- **Configuration (interpreted):** Runs daily at **01:00** (server/workflow timezone).
- **Connections:**  
  - Output → **Set Configuration**
- **Edge cases / failures:**
  - Timezone differences between server and expectation.
  - If n8n instance is down at trigger time, execution is missed (unless you add retry logic externally).

#### Node: Set Configuration
- **Type / role:** `Set` — central configuration holder (raw JSON).
- **Configuration choices:**
  - Outputs a JSON object with:
    - `n8nHost` (base URL for your n8n, expected to include protocol)
    - `apiKey` (X-N8N-API-KEY for API auth)
    - `backupFolder` (intended Drive folder path; **not actually used by the Drive nodes in this workflow**)
    - `hashAlgorithm` (e.g. `sha256`)
    - `dataTableTitle` (validated but **not used directly**; actual Data Table is selected by ID in nodes)
    - `backupScope` (`all` / `active` / `tagged`)
    - `requiredTag` (only when `backupScope=tagged`)
- **Key variables:** `$node['Set Configuration'].json.<field>`
- **Connections:**  
  - Output → **Config Validation**
- **Edge cases / failures:**
  - Leaving placeholders empty will cause validation to fail.
  - `n8nHost` must be formatted correctly for URL concatenation (see “Get All n8n Workflows” note about trailing slash).

#### Node: Config Validation
- **Type / role:** `Code` — validates configuration and hard-fails early if invalid.
- **Configuration choices (logic):**
  - Throws an Error if any of these are empty:
    - `n8nHost`, `apiKey`, `backupFolder`, `dataTableTitle`, `backupScope`
  - Additionally: if `backupScope === 'tagged'` then `requiredTag` must be non-empty.
- **Connections:**  
  - Input ← **Set Configuration**  
  - Output → **Get All n8n Workflows**
- **Edge cases / failures:**
  - This validation enforces `backupFolder` and `dataTableTitle`, but those values are not actually used later (Drive folder is selected by `folderId`; Data Table is selected by `dataTableId`). This can create confusion and “false required” configuration.

---

### 2.2 Workflow Discovery & Iteration

**Overview:**  
Fetches workflows from the n8n REST API, splits the returned list into individual items, and iterates them in batches of 1 for reliability.

**Nodes involved:**
- Get All n8n Workflows
- Split Workflows
- Process Each Workflow

#### Node: Get All n8n Workflows
- **Type / role:** `HTTP Request` — calls n8n’s REST API to list workflows.
- **Configuration choices:**
  - **URL:** `{{ $node['Set Configuration'].json.n8nHost }}api/v1/workflows`
  - **Header:** `X-N8N-API-KEY: {{ $node['Set Configuration'].json.apiKey }}`
  - Sends headers enabled; body sending is enabled but body is effectively empty.
- **Connections:**  
  - Input ← **Config Validation**  
  - Output → **Split Workflows**
- **Version-specific requirements:**
  - Uses HTTP Request node v4.x behavior (expressions in URL/headers).
- **Edge cases / failures:**
  - **Trailing slash issue:** if `n8nHost` does not end with `/`, the URL becomes `https://example.comapi/v1/workflows`. You should typically set `n8nHost` to something like `https://your-domain/` or change the expression to insert `/`.
  - 401/403 if API key invalid or API disabled.
  - Network timeouts, TLS issues, or reverse proxy restrictions.
  - Response shape is assumed to have a `data` array (used by Split Workflows).

#### Node: Split Workflows
- **Type / role:** `Split Out` — converts an array field into multiple items.
- **Configuration choices:**
  - Splits the field `data` from the API response into separate items (one per workflow).
- **Connections:**  
  - Input ← **Get All n8n Workflows**  
  - Output → **Process Each Workflow**
- **Edge cases / failures:**
  - If API response does not contain `data` or it’s not an array, the node will produce no items or error depending on n8n behavior.

#### Node: Process Each Workflow
- **Type / role:** `Split In Batches` — iterates workflows one-by-one (batch loop).
- **Configuration choices:**
  - Default batch settings (implicit batch size; typical default is 1).
  - Uses the node’s second output (“Next batch”) to control looping.
- **Connections:**
  - Input ← **Split Workflows**
  - Output (main, index 1 / next batch) → **Backup scope all?**
  - Multiple nodes loop back into **Process Each Workflow** to continue iteration:
    - From **Hash Changed?** (false branch)
    - From **If workflow active?** (false branch)
    - From **Backup scope tagged?** (false branch)
    - From **Upsert Workflow Records**
- **Edge cases / failures:**
  - If an upstream node errors for one workflow, the run may stop (unless error handling is configured per-node or via an error workflow).
  - Large instances: batching reduces memory pressure but increases total runtime.

---

### 2.3 Backup Scope Filtering (all / active / tagged)

**Overview:**  
Filters which workflows should be backed up, based on `backupScope` in configuration: all workflows, only active workflows, or workflows matching a tag.

**Nodes involved:**
- Backup scope all?
- Backup scope active?
- If workflow active?
- Backup scope tagged?

#### Node: Backup scope all?
- **Type / role:** `IF` — routes workflows depending on whether scope is “all”.
- **Configuration choices:**
  - Condition: `backupScope == 'all'` (uses expression `$('Set Configuration').item.json.backupScope`)
  - **True branch:** continue to preparation/hashing  
  - **False branch:** evaluate “active?” branch
- **Connections:**
  - Input ← **Process Each Workflow**
  - True → **Prepare Workflow**
  - False → **Backup scope active?**
- **Edge cases / failures:**
  - If `backupScope` is set to something else (e.g. typo), it will fall through to “active?” then “tagged?” logic and likely skip everything.

#### Node: Backup scope active?
- **Type / role:** `IF` — checks if scope is “active”.
- **Configuration choices:**
  - Condition: `backupScope == 'active'`
  - True branch: check if the workflow item is active
  - False branch: check “tagged?”
- **Connections:**
  - Input ← **Backup scope all?** (false)
  - True → **If workflow active?**
  - False → **Backup scope tagged?**
- **Edge cases / failures:**
  - Loose type validation enabled; usually safe here.
  - If workflow items don’t contain `active`, the next node may misbehave.

#### Node: If workflow active?
- **Type / role:** `IF` — filters only enabled workflows.
- **Configuration choices:**
  - Condition: `{{ $json.active }} === true`
- **Connections:**
  - Input ← **Backup scope active?** (true)
  - True → **Prepare Workflow**
  - False → **Process Each Workflow** (continue loop)
- **Edge cases / failures:**
  - If `active` is undefined or not boolean, it may always evaluate false (skipping backups).

#### Node: Backup scope tagged?
- **Type / role:** `IF` — filters workflows by a required tag.
- **Configuration choices (combined AND):**
  - `backupScope == 'tagged'`
  - `$json.tags` is not empty
  - `requiredTag` is not empty
  - `$json.tags[0].name == requiredTag` (**important limitation**: checks only the first tag)
- **Connections:**
  - Input ← **Backup scope active?** (false)
  - True → **Prepare Workflow**
  - False → **Process Each Workflow** (continue loop)
- **Edge cases / failures:**
  - **Tag matching limitation:** only checks `tags[0]`. If the required tag is not the first tag, it will incorrectly skip workflows.
  - If tags structure differs (e.g. no `.name`), it will fail or skip.

---

### 2.4 Normalization, Hashing & Change Detection (Data Table index)

**Overview:**  
Normalizes workflow JSON to remove volatile fields, hashes the normalized content, then compares with the stored hash in an n8n Data Table to decide whether to back up.

**Nodes involved:**
- Prepare Workflow
- Generate Workflow Hash
- Lookup Workflow in Data Table
- Workflow Exists?
- Hash Changed?

#### Node: Prepare Workflow
- **Type / role:** `Code` — normalizes workflow JSON and prepares deterministic content for hashing.
- **Configuration choices (logic):**
  - Copies the workflow object: `wf = {...$input.first().json}`
  - Removes volatile fields: `createdAt`, `updatedAt`
  - Creates `content = JSON.stringify(wf)` (intended as stable string; note it is not a true canonical JSON sort, but often sufficient if input field order is consistent)
  - Outputs:
    - `workflowId`, `workflowName`, `workflowJson`, `content`
- **Connections:**
  - Input ← one of:
    - **Backup scope all?** (true)
    - **If workflow active?** (true)
    - **Backup scope tagged?** (true)
  - Output → **Generate Workflow Hash**
- **Edge cases / failures:**
  - If the incoming item is not shaped like a workflow, required fields may be missing.
  - If n8n changes workflow object structure, the “volatile fields” list may be incomplete.

#### Node: Generate Workflow Hash
- **Type / role:** `Crypto` — generates a hash fingerprint of the workflow content.
- **Configuration choices:**
  - Hash type: `{{ $node['Set Configuration'].json.hashAlgorithm }}` (e.g. `sha256`)
  - Value to hash: `{{ $node['Prepare Workflow'].json.content }}`
  - Output field name: `hashCode`
- **Connections:**
  - Input ← **Prepare Workflow**
  - Output → **Lookup Workflow in Data Table**
- **Edge cases / failures:**
  - If `hashAlgorithm` is empty/invalid, node errors. (Validation currently does **not** enforce `hashAlgorithm` non-empty.)
  - If content is huge, hashing still usually fine but could slow down.

#### Node: Lookup Workflow in Data Table
- **Type / role:** `Data Table` — looks up an existing index record by `workflowId`.
- **Configuration choices:**
  - Operation: **get**
  - Filter: `workflowId == {{ $node['Prepare Workflow'].json.workflowId }}`
  - Return all: enabled
  - `alwaysOutputData: true` (ensures downstream nodes still run even if no record exists)
  - Data Table selected by **dataTableId** (cached name: `N8N_Workflow_Backup`)
- **Connections:**
  - Input ← **Generate Workflow Hash**
  - Output → **Workflow Exists?**
- **Edge cases / failures:**
  - Data Table not found / permissions.
  - If multiple records match, behavior depends on node’s return format; downstream assumes a single object-like record.
  - If no record exists, output may be empty object/empty array depending on n8n version—this matters for “Workflow Exists?” logic.

#### Node: Workflow Exists?
- **Type / role:** `IF` — checks whether a Data Table record exists for the workflow.
- **Configuration choices:**
  - Condition: “Lookup Workflow in Data Table” JSON is **not empty**.
- **Connections:**
  - Input ← **Lookup Workflow in Data Table**
  - True → **Hash Changed?** (existing workflow: compare hashes)
  - False → **Upload New Workflow Backup** (new workflow: upload directly)
- **Edge cases / failures:**
  - Existence check depends on the exact shape returned by Data Table “get”.
  - If the Data Table node returns an array, “notEmpty” on an object may not behave as intended.

#### Node: Hash Changed?
- **Type / role:** `IF` — determines whether a backup update is needed.
- **Configuration choices:**
  - Condition: stored `hashCode` != newly computed `hashCode`
  - Left: `{{ $node['Lookup Workflow in Data Table'].json.hashCode }}`
  - Right: `{{ $node['Generate Workflow Hash'].json.hashCode }}`
- **Connections:**
  - Input ← **Workflow Exists?** (true)
  - True → **Delete Existing Workflow Backup**
  - False → **Process Each Workflow** (continue loop; skip unchanged)
- **Edge cases / failures:**
  - If stored record has no `hashCode`, comparison may treat it as changed or error depending on coercion (loose validation is enabled).
  - If Data Table output is an array, `.json.hashCode` may not exist.

---

### 2.5 Google Drive Backup + Data Table Upsert

**Overview:**  
For new or changed workflows, ensures only one authoritative backup file exists in Drive by deleting the old file (when applicable), uploading the new JSON, then upserting the Data Table index record.

**Nodes involved:**
- Delete Existing Workflow Backup
- Upload New Workflow Backup
- Upsert Workflow Records

#### Node: Delete Existing Workflow Backup
- **Type / role:** `Google Drive` — deletes the previous backup file.
- **Configuration choices:**
  - Operation: `deleteFile`
  - File ID: `{{ $node["Lookup Workflow in Data Table"].json.DriveFileId }}`
  - `deletePermanently: false` (moves to trash rather than permanent delete)
- **Connections:**
  - Input ← **Hash Changed?** (true)
  - Output → **Upload New Workflow Backup**
- **Edge cases / failures:**
  - If `DriveFileId` is missing/invalid, deletion fails.
  - If file already deleted manually, Drive returns 404.
  - Permission issues on the file/folder.

#### Node: Upload New Workflow Backup
- **Type / role:** `Google Drive` — uploads a JSON file per workflow.
- **Configuration choices:**
  - Operation: `createFromText`
  - File name: `{{ workflowId }}_{{ workflowName with spaces replaced by underscores }}.json`
  - File content: `{{ workflowJson.toJsonString() }}`
  - Folder: chosen via `folderId` (cached name: `n8n_workflow_backups`)  
    Link: https://drive.google.com/drive/folders/1CCs1nAzFe5wDDytXJKGNyEEXqmOmb9kZ
  - Drive: `My Drive`
- **Connections:**
  - Input ← either:
    - **Workflow Exists?** (false branch = new workflow)
    - **Delete Existing Workflow Backup** (changed workflow)
  - Output → **Upsert Workflow Records**
- **Edge cases / failures:**
  - Google OAuth token expiration / revoked consent.
  - Folder ID hardcoded/selected: if folder is deleted or permissions change, upload fails.
  - Workflow names with special characters may still be problematic for filenames (less common on Drive, but possible).

#### Node: Upsert Workflow Records
- **Type / role:** `Data Table` — inserts or updates the index record for the workflow.
- **Configuration choices:**
  - Operation: `upsert`
  - Filter/match: `workflowId == {{ workflowId }}`
  - Columns written:
    - `workflowId`, `workflowName`, `hashCode`, `DriveFileId` (from uploaded file `id`)
  - Data Table: selected by `dataTableId` (cached name: `N8N_Workflow_Backup`)
- **Connections:**
  - Input ← **Upload New Workflow Backup**
  - Output → **Process Each Workflow** (continue loop)
- **Edge cases / failures:**
  - Schema mismatches in Data Table (column IDs must match exactly).
  - If Data Table is in a different project or permissions change, upsert fails.

---

### 2.6 Documentation / Notes (Sticky Notes)

**Overview:**  
Contains human guidance and intended design notes. These do not execute but are important for understanding.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

#### Node: Sticky Note
- **Type / role:** `Sticky Note` — overview, setup steps, scope explanation.
- **Notable content:** Describes hashing, delete-then-upload behavior, Data Table indexing, and scope filtering (all/active/tagged).

#### Node: Sticky Note1
- **Type / role:** `Sticky Note` — describes trigger/config/validation block.

#### Node: Sticky Note2
- **Type / role:** `Sticky Note` — describes workflow discovery and stable per-item processing.

#### Node: Sticky Note3
- **Type / role:** `Sticky Note` — describes normalization + hashing + change detection.

#### Node: Sticky Note4
- **Type / role:** `Sticky Note` — describes Drive backup and Data Table updates behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Backup Trigger | Schedule Trigger | Daily scheduler (01:00) | — | Set Configuration | ## Trigger , Configuration & Validation<br>This section defines when the workflow runs and where all configuration lives.<br>The Cron Trigger controls the daily execution time.<br>The Set Backup Configuration node centralizes all reusable values. |
| Set Configuration | Set | Central config (host, key, scope, etc.) | Daily Backup Trigger | Config Validation | ## Trigger , Configuration & Validation<br>This section defines when the workflow runs and where all configuration lives.<br>The Cron Trigger controls the daily execution time.<br>The Set Backup Configuration node centralizes all reusable values. |
| Config Validation | Code | Validate required config; fail fast | Set Configuration | Get All n8n Workflows | ## Trigger , Configuration & Validation<br>This section defines when the workflow runs and where all configuration lives.<br>The Cron Trigger controls the daily execution time.<br>The Set Backup Configuration node centralizes all reusable values. |
| Get All n8n Workflows | HTTP Request | List workflows via n8n REST API | Config Validation | Split Workflows | ## Workflow Discovery & Looping<br>This section connects to the n8n API and retrieves all existing workflows.<br>Workflows are split into individual items and processed one at a time, ensuring stability even for large or complex n8n instances.<br>At this stage, no backups are created or deleted. |
| Split Workflows | Split Out | Split `data[]` into items | Get All n8n Workflows | Process Each Workflow | ## Workflow Discovery & Looping<br>This section connects to the n8n API and retrieves all existing workflows.<br>Workflows are split into individual items and processed one at a time, ensuring stability even for large or complex n8n instances.<br>At this stage, no backups are created or deleted. |
| Process Each Workflow | Split In Batches | Iterate workflows one-by-one | Split Workflows; Hash Changed? (false); If workflow active? (false); Backup scope tagged? (false); Upsert Workflow Records | Backup scope all? | ## Workflow Discovery & Looping<br>This section connects to the n8n API and retrieves all existing workflows.<br>Workflows are split into individual items and processed one at a time, ensuring stability even for large or complex n8n instances.<br>At this stage, no backups are created or deleted. |
| Backup scope all? | IF | Scope routing: all vs other | Process Each Workflow | Prepare Workflow; Backup scope active? | ## Workflow Overview - Daily n8n Workflow Backup to Google Drive (with Change Detection)<br>Runs daily, fetches workflows, hashes, compares, deletes/updates Drive, indexes in Data Table, supports all/active/tagged scope. |
| Backup scope active? | IF | Scope routing: active vs tagged | Backup scope all? (false) | If workflow active?; Backup scope tagged? | ## Workflow Overview - Daily n8n Workflow Backup to Google Drive (with Change Detection)<br>Runs daily, fetches workflows, hashes, compares, deletes/updates Drive, indexes in Data Table, supports all/active/tagged scope. |
| If workflow active? | IF | Filter enabled workflows | Backup scope active? (true) | Prepare Workflow; Process Each Workflow | ## Workflow Overview - Daily n8n Workflow Backup to Google Drive (with Change Detection)<br>Runs daily, fetches workflows, hashes, compares, deletes/updates Drive, indexes in Data Table, supports all/active/tagged scope. |
| Backup scope tagged? | IF | Filter workflows by tag | Backup scope active? (false) | Prepare Workflow; Process Each Workflow | ## Workflow Overview - Daily n8n Workflow Backup to Google Drive (with Change Detection)<br>Runs daily, fetches workflows, hashes, compares, deletes/updates Drive, indexes in Data Table, supports all/active/tagged scope. |
| Prepare Workflow | Code | Normalize workflow JSON; create hash input | Backup scope all? (true); If workflow active? (true); Backup scope tagged? (true) | Generate Workflow Hash | ## Hashing & Change Detection<br>Each workflow’s JSON is normalized and converted into a SHA-256 hash.<br>This hash acts as a fingerprint for the workflow.<br>If any node, connection, or parameter changes, the hash changes.<br>The workflow then checks the Data Table to determine whether the workflow already exists and whether its hash has changed. |
| Generate Workflow Hash | Crypto | Hash normalized JSON content | Prepare Workflow | Lookup Workflow in Data Table | ## Hashing & Change Detection<br>Each workflow’s JSON is normalized and converted into a SHA-256 hash.<br>This hash acts as a fingerprint for the workflow.<br>If any node, connection, or parameter changes, the hash changes.<br>The workflow then checks the Data Table to determine whether the workflow already exists and whether its hash has changed. |
| Lookup Workflow in Data Table | Data Table | Get prior record by workflowId | Generate Workflow Hash | Workflow Exists? | ## Hashing & Change Detection<br>Each workflow’s JSON is normalized and converted into a SHA-256 hash.<br>This hash acts as a fingerprint for the workflow.<br>If any node, connection, or parameter changes, the hash changes.<br>The workflow then checks the Data Table to determine whether the workflow already exists and whether its hash has changed. |
| Workflow Exists? | IF | Branch: new vs existing | Lookup Workflow in Data Table | Hash Changed?; Upload New Workflow Backup | ## Hashing & Change Detection<br>Each workflow’s JSON is normalized and converted into a SHA-256 hash.<br>This hash acts as a fingerprint for the workflow.<br>If any node, connection, or parameter changes, the hash changes.<br>The workflow then checks the Data Table to determine whether the workflow already exists and whether its hash has changed. |
| Hash Changed? | IF | Branch: changed vs unchanged | Workflow Exists? (true) | Delete Existing Workflow Backup; Process Each Workflow | ## Hashing & Change Detection<br>Each workflow’s JSON is normalized and converted into a SHA-256 hash.<br>This hash acts as a fingerprint for the workflow.<br>If any node, connection, or parameter changes, the hash changes.<br>The workflow then checks the Data Table to determine whether the workflow already exists and whether its hash has changed. |
| Delete Existing Workflow Backup | Google Drive | Delete previous Drive file (to Trash) | Hash Changed? (true) | Upload New Workflow Backup | ## Google Drive Backup & Data Table Updates<br>- **New workflow** → upload backup and insert Data Table record.<br>- **Changed workflow** → delete existing Google Drive backup, upload the new version, and update the Data Table.<br>- **Unchanged workflow** → skip entirely<br>This ensures a clean Google Drive folder with accurate backups. |
| Upload New Workflow Backup | Google Drive | Upload workflow JSON as Drive file | Workflow Exists? (false); Delete Existing Workflow Backup | Upsert Workflow Records | ## Google Drive Backup & Data Table Updates<br>- **New workflow** → upload backup and insert Data Table record.<br>- **Changed workflow** → delete existing Google Drive backup, upload the new version, and update the Data Table.<br>- **Unchanged workflow** → skip entirely<br>This ensures a clean Google Drive folder with accurate backups. |
| Upsert Workflow Records | Data Table | Upsert index record (id, hash, fileId) | Upload New Workflow Backup | Process Each Workflow | ## Google Drive Backup & Data Table Updates<br>- **New workflow** → upload backup and insert Data Table record.<br>- **Changed workflow** → delete existing Google Drive backup, upload the new version, and update the Data Table.<br>- **Unchanged workflow** → skip entirely<br>This ensures a clean Google Drive folder with accurate backups. |
| Sticky Note | Sticky Note | High-level description + setup guidance | — | — | ## Workflow Overview - Daily n8n Workflow Backup to Google Drive (with Change Detection)<br>(Contains setup steps and scope explanation) |
| Sticky Note1 | Sticky Note | Notes for trigger/config section | — | — | ## Trigger , Configuration & Validation<br>This section defines when the workflow runs and where all configuration lives.<br>The Cron Trigger controls the daily execution time.<br>The Set Backup Configuration node centralizes all reusable values. |
| Sticky Note2 | Sticky Note | Notes for discovery/looping section | — | — | ## Workflow Discovery & Looping<br>This section connects to the n8n API and retrieves all existing workflows.<br>Workflows are split into individual items and processed one at a time, ensuring stability even for large or complex n8n instances.<br>At this stage, no backups are created or deleted. |
| Sticky Note3 | Sticky Note | Notes for hashing/change detection | — | — | ## Hashing & Change Detection<br>Each workflow’s JSON is normalized and converted into a SHA-256 hash.<br>This hash acts as a fingerprint for the workflow.<br>If any node, connection, or parameter changes, the hash changes.<br>The workflow then checks the Data Table to determine whether the workflow already exists and whether its hash has changed. |
| Sticky Note4 | Sticky Note | Notes for Drive + Data Table updates | — | — | ## Google Drive Backup & Data Table Updates<br>- **New workflow** → upload backup and insert Data Table record.<br>- **Changed workflow** → delete existing Google Drive backup, upload the new version, and update the Data Table.<br>- **Unchanged workflow** → skip entirely<br>This ensures a clean Google Drive folder with accurate backups. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *Daily n8n workflow backup to Google Drive* (or your preferred name)

2) **Add Trigger**
- Add node: **Schedule Trigger**
- Set schedule: daily at **01:00** (adjust as needed)
- Connect → next step

3) **Add configuration node**
- Add node: **Set**
- Mode: **Raw JSON**
- Paste/edit a config object like:
  - `n8nHost`: `https://your-n8n-domain/` (recommend trailing `/`)
  - `apiKey`: your n8n API key
  - `backupFolder`: optional label (note: not used by Drive node unless you redesign)
  - `hashAlgorithm`: `sha256`
  - `dataTableTitle`: a name you want to use (note: not used later unless redesigned)
  - `backupScope`: `all` (or `active` / `tagged`)
  - `requiredTag`: e.g. `production` if `tagged`
- Connect: **Schedule Trigger → Set**

4) **Add configuration validation**
- Add node: **Code**
- Implement validation:
  - Fail if `n8nHost`, `apiKey`, `backupFolder`, `dataTableTitle`, `backupScope` are empty
  - If `backupScope == 'tagged'`, require `requiredTag`
- Connect: **Set → Code**

5) **Add HTTP request to list workflows**
- Add node: **HTTP Request**
- Method: GET
- URL: `{{$node['Set Configuration'].json.n8nHost}}api/v1/workflows`
- Header: `X-N8N-API-KEY` = `{{$node['Set Configuration'].json.apiKey}}`
- Connect: **Config Validation → HTTP Request**

6) **Split the returned workflows**
- Add node: **Split Out**
- Field to split out: `data`
- Connect: **HTTP Request → Split Out**

7) **Loop through workflows**
- Add node: **Split In Batches**
- Batch size: 1 (default is fine; explicitly set to 1 for clarity)
- Connect: **Split Out → Split In Batches**
- You will later connect “continue” paths back into this node to keep iterating.

8) **Add scope routing: all**
- Add node: **IF**
- Condition: `backupScope == 'all'`
  - Left: `{{$node['Set Configuration'].json.backupScope}}`
  - Right: `all`
- Connect: **Split In Batches (next batch output) → IF**
- True branch → later goes to “Prepare Workflow”
- False branch → to “Backup scope active?”

9) **Add scope routing: active**
- Add node: **IF**
- Condition: `backupScope == 'active'`
- Connect: **Backup scope all? (false) → Backup scope active?**
- True → “If workflow active?”
- False → “Backup scope tagged?”

10) **Add “If workflow active?” filter**
- Add node: **IF**
- Condition: `{{$json.active}} equals true`
- Connect: **Backup scope active? (true) → If workflow active?**
- True → “Prepare Workflow”
- False → loop back to **Split In Batches** to continue iteration

11) **Add scope routing: tagged**
- Add node: **IF**
- Conditions (AND):
  - `backupScope == 'tagged'`
  - `$json.tags` is not empty
  - `requiredTag` is not empty
  - Tag match (recommended improvement: check *any* tag; but to reproduce exactly, check `$json.tags[0].name`)
- Connect: **Backup scope active? (false) → Backup scope tagged?**
- True → “Prepare Workflow”
- False → loop back to **Split In Batches**

12) **Prepare (normalize) workflow JSON**
- Add node: **Code**
- Logic:
  - Copy workflow JSON
  - Delete `createdAt`, `updatedAt`
  - `content = JSON.stringify(wf)`
  - Output `workflowId`, `workflowName`, `workflowJson`, `content`
- Connect: True outputs of:
  - **Backup scope all?**
  - **If workflow active?**
  - **Backup scope tagged?**
  → into **Prepare Workflow**

13) **Hash the normalized workflow**
- Add node: **Crypto**
- Operation: hash
- Type: `{{$node['Set Configuration'].json.hashAlgorithm}}` (e.g. sha256)
- Value: `{{$node['Prepare Workflow'].json.content}}`
- Output property: `hashCode`
- Connect: **Prepare Workflow → Crypto**

14) **Create / select a Data Table index**
- In n8n, create a **Data Table** with columns:
  - `workflowId` (string)
  - `workflowName` (string)
  - `hashCode` (string)
  - `DriveFileId` (string)
- Add node: **Data Table** (operation **Get**)
- Filter: `workflowId` equals `{{$node['Prepare Workflow'].json.workflowId}}`
- Enable “Return All” (to match the original)
- Consider enabling “Always output data”
- Connect: **Crypto → Data Table (get)**

15) **Check whether the workflow exists in index**
- Add node: **IF**
- Condition: Data Table lookup result not empty
- Connect: **Lookup → Workflow Exists?**
- True → “Hash Changed?”
- False → “Upload New Workflow Backup”

16) **Check whether hash changed**
- Add node: **IF**
- Condition: `storedHash != newHash`
  - Stored: `{{$node['Lookup Workflow in Data Table'].json.hashCode}}`
  - New: `{{$node['Generate Workflow Hash'].json.hashCode}}`
- Connect: **Workflow Exists? (true) → Hash Changed?**
- True → delete old backup
- False → loop back to **Split In Batches** (skip unchanged)

17) **Configure Google Drive credentials**
- Create/attach **Google Drive OAuth2** credentials in n8n.
- Ensure the target folder exists in Drive.

18) **Delete old Drive file (changed workflows)**
- Add node: **Google Drive**
- Operation: **Delete File**
- File ID: `{{$node["Lookup Workflow in Data Table"].json.DriveFileId}}`
- deletePermanently: false (Trash)
- Connect: **Hash Changed? (true) → Delete Existing Workflow Backup**

19) **Upload workflow JSON to Drive**
- Add node: **Google Drive**
- Operation: **Create from text**
- File name expression:  
  `{{$node['Generate Workflow Hash'].json.workflowId}}_{{$node['Generate Workflow Hash'].json.workflowName.replaceAll(" ","_")}}.json`
- Content expression:  
  `{{$node['Generate Workflow Hash'].json.workflowJson.toJsonString()}}`
- Choose Drive: “My Drive”
- Choose Folder: your backup folder (by folder selector)
- Connect:
  - **Workflow Exists? (false) → Upload**
  - **Delete Existing Workflow Backup → Upload**

20) **Upsert Data Table record**
- Add node: **Data Table** operation **Upsert**
- Filter/match: `workflowId == {{$node['Generate Workflow Hash'].json.workflowId}}`
- Set columns:
  - `workflowId`: from hash node
  - `workflowName`: from hash node
  - `hashCode`: from hash node
  - `DriveFileId`: `{{$node['Upload New Workflow Backup'].json.id}}`
- Connect: **Upload → Upsert**
- Connect: **Upsert → Split In Batches** (continue loop)

21) **Activate workflow**
- Turn workflow to **Active**
- Confirm a run creates files for eligible workflows and populates the Data Table.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Drive backup folder used by the workflow is selected by folder ID (not by `backupFolder` string). If you change the folder, update the Google Drive node’s folder selection. | Drive folder link: https://drive.google.com/drive/folders/1CCs1nAzFe5wDDytXJKGNyEEXqmOmb9kZ |
| `dataTableTitle` is validated in code, but the Data Table nodes reference a specific Data Table by ID. To truly use `dataTableTitle`, you would need logic to resolve/create the table dynamically. | Configuration/design note |
| The “tagged” filter checks only `tags[0].name`. If workflows can have multiple tags, consider checking all tags for a match. | Scope filtering robustness |
| Ensure `n8nHost` formatting is correct (recommended trailing slash) to avoid malformed URL concatenation. | API call reliability |
| The workflow deletes old backups before uploading new ones to keep one authoritative file per workflow. Deletion is non-permanent (Trash). | Backup retention behavior |
| The workflow references an `errorWorkflow` setting (by ID). If you want centralized error handling, configure an Error Workflow in your instance and attach it. | Operational reliability |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.