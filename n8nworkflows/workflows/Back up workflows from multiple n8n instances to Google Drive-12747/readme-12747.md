Back up workflows from multiple n8n instances to Google Drive

https://n8nworkflows.xyz/workflows/back-up-workflows-from-multiple-n8n-instances-to-google-drive-12747


# Back up workflows from multiple n8n instances to Google Drive

## 1. Workflow Overview

**Purpose:** Automatically back up workflows from **multiple n8n instances** into **Google Drive** as JSON files, using a “smart sync” approach:
- Upload workflows that don’t exist yet in Drive
- Update existing Drive files only when the workflow changed (leveraging Google Drive file version history)
- Skip unchanged workflows

**Primary use cases:**
- Disaster recovery and audit trail for n8n workflow definitions
- Multi-instance backup (prod/staging/self-hosted clusters) into separate Drive folders
- Lightweight versioning via Google Drive’s native revision history

### 1.1 Fetch & Dispatch (Scheduled multi-instance collection)
Runs daily, calls each n8n instance API, aggregates workflows per host, and dispatches each host’s workflow list (plus target Drive folder id) to the backup logic via a sub-workflow call.

### 1.2 Core Backup Logic (Per-host smart sync to Drive)
Receives `targetFloderId` + `workflows[]`, lists files in the Drive folder, splits workflows, filters archived ones, converts each workflow to a JSON file, matches it to any existing Drive file by workflowId embedded in filename, checks timestamps, then uploads or updates accordingly.

---

## 2. Block-by-Block Analysis

### Block A — Scheduling + Fetching workflows from multiple n8n instances (“Fetch & Dispatch”)
**Overview:** On a schedule, fetches workflow lists from two different n8n instances via the n8n Public API, aggregates each host’s workflows into a single array payload, assigns the Google Drive folder ID per host, then invokes the backup sub-workflow.

**Nodes involved:**
- Run daily
- Get n8n host 1
- Batch Host 1 data
- Config Host 1
- Get n8n host 2
- Batch Host 2 data
- Config Host 2
- Call 'n8n auto-backup to google drive'

#### Node: **Run daily**
- **Type / role:** Schedule Trigger — entry point.
- **Config choices:** Runs on an interval rule (configured as “daily” intent; the JSON shows `rule.interval:[{}]` which relies on UI defaults).
- **Outputs to:** `Get n8n host 1`, `Get n8n host 2`.
- **Edge cases:** If schedule rule is not fully defined in UI, it may not fire as expected. Timezone controlled by workflow setting (`Etc/UTC`).

#### Node: **Get n8n host 1**
- **Type / role:** n8n node (n8n Public API) — fetch workflows from instance #1.
- **Config choices:** No filters set (returns workflows per node defaults; typically “get many” workflows).
- **Credentials:** `n8nApi` = `n8n-workflow-k3s`.
- **Outputs to:** `Batch Host 1 data`.
- **Failure modes:** API auth failure (wrong API key / URL), networking, rate limits, insufficient permissions.

#### Node: **Batch Host 1 data**
- **Type / role:** Aggregate — converts multiple workflow items into a single item containing an array.
- **Config choices:** `aggregateAllItemData` into `workflows`.
- **Outputs to:** `Config Host 1`.
- **Edge cases:** If upstream returns zero items, behavior depends on n8n aggregate behavior/version; may output empty array or no item.

#### Node: **Config Host 1**
- **Type / role:** Set — attaches the target Google Drive folder and preserves workflows array.
- **Config choices:**
  - `targetFloderId`: currently **empty string** (must be filled with Google Drive folder ID for host 1).
  - `workflows`: set to `{{$json.workflows}}`.
- **Outputs to:** `Call 'n8n auto-backup to google drive'`.
- **Edge cases:** Empty/invalid folder ID will cause Drive query/upload failures later.
- **Note:** Parameter name is spelled `targetFloderId` (typo) and must match everywhere.

#### Node: **Get n8n host 2**
- **Type / role:** n8n node (n8n Public API) — fetch workflows from instance #2.
- **Config choices:** Filter `activeWorkflows: true` (only active workflows).
- **Credentials:** `n8nApi` = `n8n-workflows`.
- **Outputs to:** `Batch Host 2 data`.
- **Edge cases:** Inactive workflows will not be backed up from host 2 due to the filter.

#### Node: **Batch Host 2 data**
- **Type / role:** Aggregate — builds `workflows[]` array.
- **Config choices:** `aggregateAllItemData` into `workflows`.
- **Outputs to:** `Config Host 2`.

#### Node: **Config Host 2**
- **Type / role:** Set — attaches the target Drive folder for host 2.
- **Config choices:**
  - `targetFloderId`: currently **empty string** (must be filled).
  - `workflows`: `{{$json.workflows}}`.
- **Outputs to:** `Call 'n8n auto-backup to google drive'`.

#### Node: **Call 'n8n auto-backup to google drive'**
- **Type / role:** Execute Workflow — dispatches to a separate workflow (sub-workflow).
- **Config choices:**
  - Mode: `each`
  - `waitForSubWorkflow: false` (fire-and-forget; parent won’t wait for completion)
  - Inputs:
    - `workflows`: `{{$json.workflows}}`
    - `targetFloderId`: `{{$json.targetFloderId}}`
- **Sub-workflow reference:** Calls workflow named **“n8n auto-backup to google drive”** (ID `kNBofPjNI2bsqxNl7yRNu`).
- **Failure modes / edge cases:**
  - If sub-workflow errors, parent may still show success because it doesn’t wait.
  - Caller policy set to `workflowsFromSameOwner` may prevent calling across owners.
  - The called workflow must define matching inputs, or the trigger side must accept them.

---

### Block B — Sub-workflow entry + Drive folder inventory (“Core Backup Logic” start)
**Overview:** This workflow can be executed as a sub-workflow. It receives a Drive folder id and a workflow array, then lists existing files in that folder so later steps can decide whether to upload or update.

**Nodes involved:**
- When Executed by Another Workflow
- Get file list
- Extract file information
- Filter files

#### Node: **When Executed by Another Workflow**
- **Type / role:** Execute Workflow Trigger — entry point for sub-workflow execution.
- **Config choices:** Defines inputs:
  - `targetFloderId` (string)
  - `workflows` (array)
- **Outputs to:** `Get file list` and `Split Out` (parallel branches).
- **Edge cases:** If caller doesn’t pass `workflows` as array, downstream `Split Out` fails or produces no items.

#### Node: **Get file list**
- **Type / role:** Google Drive — lists files in the target folder.
- **Config choices:**
  - Resource: `fileFolder`
  - Search method: query
  - Query: `='{{ $json.targetFloderId }}' in parents and trashed = false`
  - Return all: true
  - Fields: `*` (requests full metadata; heavier but convenient)
  - `alwaysOutputData: true` (continues even if no files are found)
- **Credentials:** Google Drive OAuth2 = `Google Drive-William-N8NDRIVE`.
- **Outputs to:** `Extract file information`.
- **Failure modes:**
  - Invalid folder ID → 404 / permission error
  - OAuth token revoked/expired
  - Large folders → pagination/latency; `returnAll` can be slow

#### Node: **Extract file information**
- **Type / role:** Set — normalizes Drive file metadata for matching.
- **Config choices / key expressions:**
  - `driveFileId` = `{{$json.id}}`
  - `driveFileModiedTime` = `{{$json.modifiedTime}}` (note the misspelling “Modied”)
  - `workflowId` = `{{$json.name.match(/\(#(.+)\)/)[1] }}`
    - Extracts workflow id from filenames like: `My Workflow(#abc123).json`
  - `driveMimeType` = `{{$json.kind}}` (likely should be `mimeType`; `kind` is not the MIME type)
- **Outputs to:** `Filter files`.
- **Edge cases / failures:**
  - If a file name does not contain `(#...)`, `.match(...)` returns `null` and indexing `[1]` throws an expression error.
  - If Drive returns folders or special items, fields may differ.

#### Node: **Filter files**
- **Type / role:** Filter — removes folders from the Drive listing.
- **Config choices:** Condition: `{{$('Get file list').item.json.mimeType}}` **not ends with** `folder`.
  - This is an unusual check; Drive folders have `mimeType = application/vnd.google-apps.folder` (does not end with “folder”). This filter may not behave as intended.
- **Outputs to:** `Merge`.
- **Edge cases:** Because the condition references `Get file list` item context, ensure item pairing is correct; also the “endsWith folder” heuristic is fragile.

---

### Block C — Workflow iteration + file creation + metadata enrichment
**Overview:** Splits the incoming `workflows[]` into individual items, excludes archived workflows, creates a JSON file binary for each workflow, and sets standardized naming and matching fields for later comparison.

**Nodes involved:**
- Split Out
- Filter archived workflows
- Convert to File
- Set Workflow Name

#### Node: **Split Out**
- **Type / role:** Split Out — converts `workflows[]` array into one item per workflow.
- **Config choices:**
  - Field to split: `workflows`
  - Include: `allOtherFields` (keeps `targetFloderId` alongside each workflow item)
- **Outputs to:** `Filter archived workflows`.
- **Edge cases:** If `workflows` is missing/not array → no output or error depending on n8n version.

#### Node: **Filter archived workflows**
- **Type / role:** Filter — removes archived workflows from backup.
- **Config choices:** Condition: `{{$json.workflows.isArchived}}` is **false**.
- **Outputs to:** `Convert to File`.
- **Edge cases:** If `isArchived` is undefined, strict validation could fail; depends on returned API schema.

#### Node: **Convert to File**
- **Type / role:** Convert to File — turns each workflow JSON into a binary file for Drive upload/update.
- **Config choices:**
  - Operation: `toJson`
  - Mode: `each`
  - Binary property: `data`
  - Filename: `{{$json.workflows.name }}(#{{$json.workflows.id}}).json`
- **Outputs to:** `Set Workflow Name`.
- **Edge cases:** Workflow names with `/` or other special characters can produce invalid Drive filenames or unexpected behavior.

#### Node: **Set Workflow Name**
- **Type / role:** Set — prepares consistent fields for merge/matching and upload/update.
- **Config choices / expressions:**
  - `workflowName` = `{{$('Filter archived workflows').item.json.workflows.name}}(#{{$('Filter archived workflows').item.json.workflows.id}}).json`
  - `workflows` = object from `Filter archived workflows`
  - `targetFloderId` = from `Filter archived workflows`
  - `workflowId` = workflow id
  - `includeOtherFields: true` (keeps binary `data` from Convert to File)
- **Outputs to:** `Merge` (input 2).
- **Edge cases:** Heavy reliance on `$('Filter archived workflows').item` linkage; if item pairing breaks, fields can mismatch.

---

### Block D — Matching Drive files to workflows + change detection
**Overview:** Merges “workflow items” with “Drive file items” by `workflowId`, then checks whether Drive’s modified time is older/equal to the workflow `updatedAt` to decide whether to upload or update.

**Nodes involved:**
- Merge
- Filter files by update time
- Route: New vs Update

#### Node: **Merge**
- **Type / role:** Merge — combines Drive file metadata with workflow metadata.
- **Config choices:**
  - Mode: `combine`
  - Join mode: `keepEverything`
  - Field match: `workflowId`
- **Inputs:**
  - Input 1: from `Filter files` (Drive files with extracted workflowId)
  - Input 2: from `Set Workflow Name` (workflows)
- **Outputs to:** `Filter files by update time`.
- **Edge cases / failures:**
  - If Drive parsing fails (no workflowId), merging won’t match and may create partial records.
  - Multiple Drive files with same extracted workflowId → ambiguous merges / duplicated updates.

#### Node: **Filter files by update time**
- **Type / role:** Filter — checks if backup should run based on timestamps.
- **Config choices:**
  - Condition 1: `{{$json.driveFileModiedTime || 0}}` **beforeOrEquals** `{{$json.workflows.updatedAt}}`
  - Condition 2: a blank “equals” condition exists with empty left/right values (likely accidental). Depending on node behavior, it may always pass or fail; it’s risky.
- **Outputs to:** `Route: New vs Update`.
- **Edge cases:**
  - Timezone/format mismatches: Drive `modifiedTime` is RFC3339; `updatedAt` from n8n API might be ISO string—usually compatible, but strict parsing issues can occur.
  - If `driveFileModiedTime` missing, uses `0` (epoch) which will almost always be “before”, causing unnecessary updates.

#### Node: **Route: New vs Update**
- **Type / role:** IF — decides whether to upload new or update existing.
- **Config choices:** If `{{$('Merge')?.item?.json['driveFileId'] || ""}}` is empty → treat as **new**, else **update**.
- **Outputs:**
  - True branch → `Upload workflow`
  - False branch → `Update workflow`
- **Edge cases:** If merge didn’t match due to filename parsing, existing backups may be treated as “new” and duplicated.

---

### Block E — Write to Google Drive (upload/update)
**Overview:** Creates or updates the JSON file in Google Drive. Updates keep Drive revision history.

**Nodes involved:**
- Upload workflow
- Update workflow

#### Node: **Upload workflow**
- **Type / role:** Google Drive — upload/create file.
- **Config choices:**
  - Name: `{{$json.workflowName}}`
  - Drive: “My Drive”
  - Folder ID: `{{$json.targetFloderId}}`
  - (Implicit) file content: comes from binary property `data` created earlier (Google Drive node uses incoming binary when uploading; ensure binary is present).
- **Credentials:** `Google Drive-William-N8NDRIVE`.
- **Failure modes:** Missing binary data, insufficient folder permissions, name conflicts (Drive allows duplicates), quota issues.

#### Node: **Update workflow**
- **Type / role:** Google Drive — update file content and rename.
- **Config choices:**
  - Operation: update
  - `fileId`: `{{$json.driveFileId}}`
  - `changeFileContent: true` (updates the JSON)
  - `newUpdatedFileName`: `{{$json.workflowName}}`
- **Credentials:** `Google Drive-William-N8NDRIVE`.
- **Failure modes:** fileId not found, permission denied, missing binary data, update conflicts.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run daily | Schedule Trigger | Starts the backup run on a schedule | — | Get n8n host 2; Get n8n host 1 | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Get n8n host 1 | n8n | Fetch workflows from n8n instance #1 | Run daily | Batch Host 1 data | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Batch Host 1 data | Aggregate | Aggregate host #1 workflows into `workflows[]` | Get n8n host 1 | Config Host 1 | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Config Host 1 | Set | Attach Drive folder id for host #1 | Batch Host 1 data | Call 'n8n auto-backup to google drive' | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Get n8n host 2 | n8n | Fetch workflows from n8n instance #2 (active only) | Run daily | Batch Host 2 data | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Batch Host 2 data | Aggregate | Aggregate host #2 workflows into `workflows[]` | Get n8n host 2 | Config Host 2 | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Config Host 2 | Set | Attach Drive folder id for host #2 | Batch Host 2 data | Call 'n8n auto-backup to google drive' | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Call 'n8n auto-backup to google drive' | Execute Workflow | Invoke the backup sub-workflow per host | Config Host 1; Config Host 2 | — | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point for core backup logic | — | Get file list; Split Out | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Get file list | Google Drive | List existing files in target Drive folder | When Executed by Another Workflow | Extract file information | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Extract file information | Set | Normalize Drive metadata + parse workflowId from filename | Get file list | Filter files | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Filter files | Filter | Attempt to exclude folder entries | Extract file information | Merge | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Split Out | Split Out | Split `workflows[]` into one item per workflow | When Executed by Another Workflow | Filter archived workflows | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Filter archived workflows | Filter | Exclude archived workflows from backup | Split Out | Convert to File | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Convert to File | Convert to File | Convert workflow JSON to binary file | Filter archived workflows | Set Workflow Name | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Set Workflow Name | Set | Standardize file naming + fields for merge/upload/update | Convert to File | Merge | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Merge | Merge | Join Drive file info with workflow info by workflowId | Filter files; Set Workflow Name | Filter files by update time | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Filter files by update time | Filter | Pass only workflows that are new/updated | Merge | Route: New vs Update | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Route: New vs Update | IF | Decide between creating a new file vs updating existing | Filter files by update time | Upload workflow; Update workflow | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Upload workflow | Google Drive | Upload new workflow JSON file | Route: New vs Update (true) | — | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Update workflow | Google Drive | Update existing Drive file content (keeps revisions) | Route: New vs Update (false) | — | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |
| Sticky Note6 | Sticky Note | Documentation block (how it works / setup) | — | — | ## 📘 How it works\nThis workflow provides a robust, automated backup solution for one or multiple n8n instances, syncing your workflows directly to Google Drive with version history support.\n\nIt operates in two main stages:\n1. **Fetch & Dispatch**: The workflow runs on a schedule (e.g., daily). It connects to your configured n8n instances via the n8n Public API to retrieve all active workflows.\n2. **Smart Sync**: It recursively processes the workflows for each instance. It compares the local workflows against the files in your specified Google Drive folder.\n   - **New Workflows**: Automatically uploaded as JSON files.\n   - **Updated Workflows**: If a workflow has been modified since the last backup, the existing file on Drive is updated. This leverages Google Drive's native version history, allowing you to roll back changes if needed.\n   - **Unchanged Workflows**: Skipped to save bandwidth and API calls.\n\n## 📝 Setup steps\n1. **Credentials**: Configure your **n8n API** credentials for each instance you want to back up, and your **Google Drive OAuth2** credentials.\n2. **Folder Setup**: Create a folder in Google Drive for each n8n instance. Copy the `Folder ID` from the browser URL for each.\n3. **Configuration**:\n   - In the \"Config Host\" nodes (Set nodes), paste your Google Drive `Folder ID`.\n   - Ensure the \"Process backup recursively\" node is set to call *this* workflow's ID (or the specific sub-workflow ID if you separated them).\n4. **Schedule**: Enable the Schedule Trigger to start backing up automatically. |
| Sticky Note | Sticky Note | Documentation label for stage 1 | — | — | ## ⏱️ 1. Fetch & Dispatch  \nRuns on a schedule to fetch workflow data from multiple n8n instances. |
| Sticky Note1 | Sticky Note | Documentation label for stage 2 | — | — | ## 💾 2. Core Backup Logic  \nHandles the comparison, version control, and file upload to Google Drive. |

---

## 4. Reproducing the Workflow from Scratch

### A) Create the **Core Backup Logic** workflow (the one that receives inputs and writes to Drive)

1. **Create new workflow** named something like: `n8n auto-backup to google drive` (this will be the sub-workflow).
2. Add **Execute Workflow Trigger** node:
   - Name: `When Executed by Another Workflow`
   - Define inputs:
     - `targetFloderId` (string)
     - `workflows` (array)

3. Add **Google Drive** node:
   - Name: `Get file list`
   - Resource: *File/Folder*
   - Operation: *Search/List* (query search)
   - Query: `='{{ $json.targetFloderId }}' in parents and trashed = false`
   - Return All: enabled
   - Fields: `*`
   - Credentials: create/select **Google Drive OAuth2** credential with access to the target folders.

4. Add **Set** node:
   - Name: `Extract file information`
   - Set:
     - `driveFileId` = `{{$json.id}}`
     - `driveFileModiedTime` = `{{$json.modifiedTime}}`
     - `workflowId` = `{{$json.name.match(/\(#(.+)\)/)[1]}}`
     - (Optional but recommended) store `mimeType` properly (the current workflow uses `kind`, which is likely wrong)

5. Add **Filter** node:
   - Name: `Filter files`
   - Condition to exclude folders (recommended robust option):
     - `{{$json.mimeType}} != "application/vnd.google-apps.folder"`
   - Connect: `Extract file information` → `Filter files`.

6. From the trigger node, add **Split Out** node:
   - Name: `Split Out`
   - Field to split: `workflows`
   - Include all other fields: enabled
   - Connect: `When Executed...` → `Split Out`.

7. Add **Filter** node:
   - Name: `Filter archived workflows`
   - Condition: `{{$json.workflows.isArchived}}` is false
   - Connect: `Split Out` → `Filter archived workflows`.

8. Add **Convert to File** node:
   - Name: `Convert to File`
   - Operation: JSON → file
   - Binary property: `data`
   - File name: `{{$json.workflows.name}}(#{{$json.workflows.id}}).json`
   - Connect: `Filter archived workflows` → `Convert to File`.

9. Add **Set** node:
   - Name: `Set Workflow Name`
   - Include other fields: enabled
   - Set:
     - `workflowName` = `{{$json.workflows.name}}(#{{$json.workflows.id}}).json`
     - `workflowId` = `{{$json.workflows.id}}`
     - `targetFloderId` = `{{$json.targetFloderId}}`
     - `workflows` = `{{$json.workflows}}`
   - Connect: `Convert to File` → `Set Workflow Name`.

10. Add **Merge** node:
    - Name: `Merge`
    - Mode: *Combine*
    - Match field: `workflowId`
    - Join mode: keep everything
    - Connect:
      - `Filter files` → `Merge` (Input 1)
      - `Set Workflow Name` → `Merge` (Input 2)

11. Add **Filter** node:
    - Name: `Filter files by update time`
    - Condition: `{{$json.driveFileModiedTime || 0}}` is **before or equal** `{{$json.workflows.updatedAt}}`
    - (Remove any accidental empty condition rows in the UI.)
    - Connect: `Merge` → `Filter files by update time`

12. Add **IF** node:
    - Name: `Route: New vs Update`
    - Condition: `{{$json.driveFileId || ""}}` is empty
    - Connect: `Filter files by update time` → `Route: New vs Update`.

13. Add **Google Drive** node for uploads:
    - Name: `Upload workflow`
    - Operation: Upload/Create file
    - Name: `{{$json.workflowName}}`
    - Folder ID: `{{$json.targetFloderId}}`
    - Ensure it uses the incoming binary property `data` as file content.
    - Connect IF (true) → `Upload workflow`

14. Add **Google Drive** node for updates:
    - Name: `Update workflow`
    - Operation: Update file
    - File ID: `{{$json.driveFileId}}`
    - Change file content: enabled (binary `data`)
    - New file name: `{{$json.workflowName}}`
    - Connect IF (false) → `Update workflow`

15. Save and activate this sub-workflow if desired.

---

### B) Create the **Dispatcher** workflow (scheduled, multi-instance)
1. Create a second workflow named: `Back up n8n workflows to Google Drive automatically`.

2. Add **Schedule Trigger**:
   - Name: `Run daily`
   - Set schedule (daily at desired time; timezone as needed).

3. Add **n8n** node:
   - Name: `Get n8n host 1`
   - Configure for “List workflows” (default)
   - Credentials: create `n8n API` credential pointing to instance #1 (base URL + API key).

4. Add **Aggregate** node:
   - Name: `Batch Host 1 data`
   - Aggregate all items into `workflows`.

5. Add **Set** node:
   - Name: `Config Host 1`
   - Set:
     - `targetFloderId` = *(paste Google Drive folder ID for host 1 backups)*
     - `workflows` = `{{$json.workflows}}`

6. Repeat steps 3–5 for host 2:
   - `Get n8n host 2` (optionally filter active workflows only)
   - `Batch Host 2 data`
   - `Config Host 2` with its own `targetFloderId`

7. Add **Execute Workflow** node:
   - Name: `Call 'n8n auto-backup to google drive'`
   - Select the sub-workflow created in section A
   - Pass inputs:
     - `targetFloderId` = `{{$json.targetFloderId}}`
     - `workflows` = `{{$json.workflows}}`
   - Consider setting **Wait for sub-workflow = true** if you want the parent execution to reflect errors.

8. Connect:
   - `Run daily` → `Get n8n host 1` and `Get n8n host 2`
   - Each host chain → `Call 'n8n auto-backup to google drive'`

9. Configure/verify credentials:
   - **n8nApi** for each host (API URL + key)
   - **Google Drive OAuth2** with access to the backup folders

10. Activate the dispatcher workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” + setup steps (credentials, folder IDs, schedule) | Included in sticky note: *How it works* (content embedded in workflow) |
| Design intent: two-stage approach (“Fetch & Dispatch” then “Core Backup Logic”) | Sticky Note: “⏱️ 1. Fetch & Dispatch” and “💾 2. Core Backup Logic” |
| Important naming dependency: workflowId is extracted from Drive filename pattern `(#ID)` | If files in Drive don’t follow `Name(#<workflowId>).json`, merging will fail and duplicates may be created |
| Potential misconfigurations to fix | `Filter files` logic for folders is fragile; `Extract file information` uses `kind` instead of `mimeType`; `Filter files by update time` contains an empty condition row; both `Config Host` nodes have empty `targetFloderId` placeholders |
| Disclaimer (provided) | Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. |