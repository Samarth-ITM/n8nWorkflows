Back up n8n workflows to Google Drive while preserving folder structure

https://n8nworkflows.xyz/workflows/back-up-n8n-workflows-to-google-drive-while-preserving-folder-structure-13245


# Back up n8n workflows to Google Drive while preserving folder structure

## 1. Workflow Overview

**Purpose:**  
This workflow automatically backs up all **non-archived n8n workflows** from a given n8n instance into **Google Drive**, while **preserving the complete nested folder structure** (root folders + subfolders). Each run creates a new timestamped backup folder, uploads the mirrored folder tree, then uploads each workflow as a JSON file into the correct Google Drive folder.

**Target use cases:**
- Scheduled backups for n8n Cloud or self-hosted.
- Disaster recovery (workflows exported as JSON files).
- Keeping a historical archive of project structure and workflows.
- Environments where Google Sheets API quotas are too restrictive (uses **n8n Data Tables** instead).

### 1.1 Trigger & n8n API Authentication
- Scheduled execution.
- Logs into n8n via `/rest/login` and extracts the `set-cookie` token (stored as `n8n-auth`) for subsequent API calls.

### 1.2 Data Tables Preparation (folders + workflows)
- Creates (or attempts to create) two n8n data tables: `folders` and `workflows`, defines their columns, then retrieves their IDs.
- Purges existing rows to start from a clean state each run.

### 1.3 Fetch Folder/Workflow Metadata and Build Folder Full Paths
- Fetches folders/workflows metadata from the n8n API.
- Inserts folder rows into the `folders` table.
- Computes each folder’s `full_path` (recursive parent chain).

### 1.4 Create a Timestamped Google Drive Backup Root
- Creates a new folder under a static Google Drive “backup_n8n” parent folder, named `n8n_backup_folder_structure_DDMMYYYY_HHmmss`.
- Stores that Google folder ID back into `folders` table rows.

### 1.5 Mirror Folder Tree into Google Drive (in correct creation order)
- Sorts folders by path depth so parents are created before children.
- For each folder:
  - Determines its Google Drive parent folder ID (root = backup run folder; subfolder = created parent’s ID).
  - Searches if folder exists; creates if missing.
  - Upserts the created Google folder ID into the `folders` table.

### 1.6 Save Workflow-to-Folder Relationships into `workflows` Table
- Fetches each workflow details and matches it to folder metadata.
- Inserts/upserts workflow rows including `full_path` and later will store Google file/folder IDs.

### 1.7 Export Workflows to JSON Files and Upload to Google Drive
- Retrieves each workflow JSON.
- Converts to `.json` binary file.
- Uploads either:
  - to the backup run root folder (if workflow is at main root), or
  - to the mapped Google Drive folder (if workflow is inside a folder).
- Updates `workflows` table with the Google file ID and folder ID.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Instance Credentials Initialization
**Overview:** Starts on schedule and defines the n8n instance connection details (URL + login).  
**Nodes involved:** `Schedule Trigger`, `n8n instance/project access details`  
**Sticky notes:**  
- “## Backup n8n workflows to Google Drive with automatic cleanup …”  
- “# 1/ Initialize n8n instance access details to reach n8n API”  
- “## Add your n8n URL and credentials here”

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger; runs workflow automatically.
- **Config:** Every 4 hours.
- **Outputs:** To `n8n instance/project access details`.
- **Edge cases:** n8n instance timezone; overlapping executions if runtime > interval (consider concurrency settings).

#### Node: n8n instance/project access details
- **Type / role:** Set node; stores connection parameters used throughout workflow.
- **Config choices:** Hard-coded fields:
  - `n8n_instance_URL` (must not end with `/`)
  - `emailOrLdapLoginId`
  - `password`
- **Used by expressions:** Many HTTP Request nodes reference `$('n8n instance/project access details').item.json.n8n_instance_URL`.
- **Edge cases:** storing plaintext password in workflow is risky; consider credentials/variables.

---

### Block 2 — n8n API Login & Auth Cookie Extraction
**Overview:** Logs into n8n REST API and captures the returned session cookie used for authenticated API calls.  
**Nodes involved:** `n8n login API1`, `n8n-auth for n8n API`  
**Sticky note:**  
- “## Connect to the account using API to get the "set-cookie" token value … {{ $json.headers["set-cookie"][0] }}”

#### Node: n8n login API1
- **Type / role:** HTTP Request; POST login to n8n.
- **Config:**
  - `POST {{$json.n8n_instance_URL}}/rest/login`
  - JSON body includes email + password from previous Set node
  - Requests full response to access headers
- **Output:** Full HTTP response (including headers).
- **Edge cases / failures:**
  - Wrong credentials → 401/403.
  - Cloud instances may enforce MFA/SSO (login endpoint might differ/deny).
  - Rate limiting or Captcha-like protections (rare but possible).

#### Node: n8n-auth for n8n API
- **Type / role:** Set node; extracts `n8n-auth` cookie.
- **Key expression:** `{{$json.headers["set-cookie"][0]}}`
- **Output field:** `n8n-auth`
- **Edge cases:**
  - Header shape differs (multiple cookies, changed order) → extraction may be wrong.
  - If n8n changes cookie name/format, downstream calls fail with 401.

---

### Block 3 — Create / Identify n8n Data Tables and Purge Their Data
**Overview:** Ensures two n8n data tables exist (`folders`, `workflows`), adds expected columns, resolves their table IDs, then clears existing rows.  
**Nodes involved:**  
`get projectId`, `create n8n table "folders"`, `create n8n table "workflows"`, `"folders" table columns list`, `"workflows" table columns list`, `add columns to "folders" table`, `add columns to "workflows" table`, `get n8n table "id"`, `Split Out`, `Filter on "folders" table`, `Purge data in "folders"`, `dataTableId "folders"`, `get n8n table "id"1`, `Split Out2`, `Filter on "workflows" table`, `Purge data in "workflows"`, `dataTableId "workflows"`, `Merge dataTableIds for "folders" & "workflows"`, `Use dataTableIds for "folders" & "workflows"`  
**Sticky notes:**  
- “# 2/ Create the n8n data tables "folders" & "workflows"”  
- “## n8n tables used to store … Gsheet API quotas limit … https://developers.google.com/workspace/sheets/api/limits”  
- “## Capture both n8n dataTableId …”

#### Node: get projectId
- **Type / role:** HTTP Request; lists projects to obtain a project ID.
- **Config:** `GET /rest/projects` with `Cookie: {{$json["n8n-auth"]}}`
- **Outputs:** Used to build `/rest/projects/{projectId}/data-tables` URLs.
- **Edge cases:** assumes `data[0].id` is correct project; if multiple projects, may back up unintended one.

#### Node: create n8n table "folders" / create n8n table "workflows"
- **Type / role:** HTTP Request; creates data tables.
- **Config:** `POST /rest/projects/{{projectId}}/data-tables` with body `{name:"folders"|"workflows", columns:[], hasHeaders:true}`
- **Error handling:** `onError: continueErrorOutput` (important).
- **Edge cases:** if table already exists → API may return error; workflow continues but must still retrieve IDs via later “get n8n table id” calls.

#### Node: "folders" table columns list / "workflows" table columns list
- **Type / role:** Code node; emits one item per column definition.
- **Config choices:** All columns typed as `"string"`.
- **Edge cases:** schema mismatch if you later change Data Table schema manually.

#### Node: add columns to "folders" table / add columns to "workflows" table
- **Type / role:** HTTP Request; adds columns to created table.
- **Config:** `POST .../data-tables/{{tableId}}/columns` with body set to current column JSON item.
- **Edge cases:** re-adding existing columns may error; not marked `continueErrorOutput`, so could fail if API rejects duplicates.

#### Node: get n8n table "id" / get n8n table "id"1 + Split/Filter
- **Type / role:** HTTP Request + SplitOut + Filter to locate table by name.
- **Config:** `GET /rest/projects/{{projectId}}/data-tables`, split `data.data`, filter by `.name == folders` or `.name == workflows`.
- **Edge cases:** response structure changes; filter expressions use `rightValue:"=folders"` style which relies on n8n’s expression parsing—misconfiguration can cause no matches.

#### Node: Purge data in "folders" / Purge data in "workflows"
- **Type / role:** Data Table node; deletes all rows.
- **Config:** `deleteRows` with filter `keyValue: 0, condition: gte` (acts as “delete everything”).
- **Edge cases:** if table empty, should no-op; if filters behave unexpectedly, might not purge all.

#### Node: dataTableId "folders" / dataTableId "workflows"
- **Type / role:** Set node; stores resolved table IDs.
- **Key expression:** `{{$json["data.data"] ? $json["data.data"].id : $json.data.dataTableId}}`
- **Edge cases:** relies on either API response shape from “get table list” or from “create table” response.

#### Node: Merge dataTableIds for "folders" & "workflows" → Use dataTableIds...
- **Type / role:** Merge nodes; combine IDs into one item for reuse.
- **Config:** combineByPosition.
- **Downstream:** Many Data Table nodes reference:
  - `$('Use dataTableIds for "folders" & "workflows"').item.json.folders_dataTableId`
  - `...workflows_dataTableId`

---

### Block 4 — Fetch Folder Structure from n8n and Populate `folders` Table
**Overview:** Retrieves folders metadata, normalizes it, and upserts into the `folders` table.  
**Nodes involved:** `get subfolder(s) structure`, `Split Out1`, `If resource == "folder"`, `rename fields1`, `Insert data into table "folders"`, `Aggregate3`  
**Sticky note:** “# 3/ Get project folder(s) and add names in n8n tables "folders" & "workflows" : root and subfolders”

#### Node: get subfolder(s) structure
- **Type / role:** HTTP Request; fetches workflows listing including folders/scopes.
- **Config:** `GET /rest/workflows?includeScopes=true&includeFolders=true&skip=0&take=400`
- **Auth:** `Cookie: {{$json['n8n-auth']}}`
- **Edge cases:** `take=400` limits results; larger instances may truncate.

#### Node: Split Out1
- **Type / role:** SplitOut; splits `data` array into items.

#### Node: If resource == "folder"
- **Type / role:** If node; keeps only records where `data.resource == 'folder'`.
- **Config detail:** Right side is set as `=folder` (string).
- **Edge cases:** if API uses different `resource` values, no folders pass.

#### Node: rename fields1
- **Type / role:** Set node; maps folder fields into normalized schema for table:
  - `folderId`, `folderName`, `folderType` (root vs sub folder)
  - `parentFolderName`, `parentFolderId` (or `'null'`)
  - counts, plus placeholders: `google_folder_id`, `parent_google_folderid`, `main_root_folder_name`, `main_root_folder_google_folder_id`
- **Edge cases:** `parentFolderId` might be null (actual `null` vs string `'null'`); later logic compares strings.

#### Node: Insert data into table "folders"
- **Type / role:** Data Table upsert into `folders`.
- **Matching:** filter on `folderName`.
- **Edge cases:** folderName collisions (two folders with same name in different branches) will overwrite each other.  
  **This is a key functional limitation:** upserting by `folderName` alone is unsafe for nested structures with duplicate folder names.

#### Node: Aggregate3
- **Type / role:** Aggregate all item data to feed multiple branches at once.
- **Outputs connected to:**  
  `get all workflows`, `Get all folders`, `Create backup folder ...`, `get all folders1`, `get all workflows1`

---

### Block 5 — Compute `full_path` for Each Folder and Store It
**Overview:** Builds each folder’s full path recursively using parentFolderId relationships, then upserts the computed `full_path` back into `folders` table.  
**Nodes involved:** `Get all folders`, `Define & assign folder's fullPath value`, `Insert "full_path" value for each folder`  
**Sticky note:** “# 4/ Assign the folder's fullPath value”

#### Node: Get all folders
- **Type / role:** Data Table get; fetches all folder rows.

#### Node: Define & assign folder's fullPath value
- **Type / role:** Code; builds map by `folderId` and recursively constructs `full_path`.
- **Logic highlights:**
  - Root folder: no parent or `'null'` → full path = folderName
  - Missing parent → returns folderName (edge case)
- **Edge cases:**
  - Cycles in parent references would recurse infinitely (unlikely, but not guarded).
  - If parentFolderId is real `null` (not string `'null'`), condition `folder.parentFolderId === 'null'` won’t catch; still handled by `!folder.parentFolderId` though.

#### Node: Insert "full_path" value for each folder
- **Type / role:** Data Table upsert; writes `full_path` to matching `folderName`.
- **Edge cases:** same folderName collision problem again.

---

### Block 6 — Create Backup Run Root Folder in Google Drive and Persist Its ID
**Overview:** Creates the timestamped root backup folder under a static Google Drive folder (“backup_n8n”), then stores the new root folder ID into the `folders` table rows.  
**Nodes involved:** `Create backup folder "n8n_backup_folder_structure_ddMMyyyy_HHmmss"`, `get all folders2`, `rename main_root_folder fields`, `add backup_folder google_id to "folders"`, `Merge3`  
**Sticky notes:**  
- “# 6/ Upload the n8n folder structure in gdrive … example link …”  
- “## Create manually your google drive folder "backup_n8n" and select it in this node”

#### Node: Create backup folder "n8n_backup_folder_structure_ddMMyyyy_HHmmss"
- **Type / role:** Google Drive folder create.
- **Config:**
  - Name: `n8n_backup_folder_structure_{{$now.format('ddMMyyyy_HHmmss')}}`
  - Parent Folder: fixed ID (example points to a “backup_n8n” folder)
- **Credentials:** Google Drive OAuth2.
- **Edge cases:**
  - Missing permission on parent folder → 403.
  - Duplicate folder names possible if triggered within same second (rare).

#### Node: get all folders2
- **Type / role:** Data Table get; returns all folder rows.

#### Node: rename main_root_folder fields
- **Type / role:** Set; stores:
  - `main_root_folder_name` = created backup folder name
  - `main_root_folder_google_folder_id` = created backup folder ID

#### Node: add backup_folder google_id to "folders"
- **Type / role:** Data Table upsert; updates each folder row to include backup run root folder info.
- **Matching:** by `folderName`.

#### Node: Merge3
- **Type / role:** Merge; enriches data needed for ordering folder creation.

---

### Block 7 — Order Folders and Create Them in Google Drive (Preserve Structure)
**Overview:** Sorts folders by depth, loops through each folder, determines the correct Google Drive parent, searches/creates the folder, then stores Google folder IDs back to the `folders` table.  
**Nodes involved:**  
`define full_path length`, `Sort to get root folders first`, `One run per folder`, `Get the parentFolderName if it exists`, `get parentFolderId if it exists`, `Search if folder already exists`, `if folder not found => create it`, `Attach the parentFolder if any`, `if parentFolder empty`, `Create folder under the static "backup_n8n" folder`, `Create folder under the parentFolder`, `get  parent_google_folderid value`, `Merge6`, `rename fields2`, `rename fields3`, `Add data about folder : google_folder_id1`, `Add data about folder : google_folder_id`  
**Sticky notes:**  
- “## root folder(s) : full_path = 1”  
- “## subfolders”  
- Long explanation about sorting order (Utilities/Reports example).  
- “## We get the id of each created folder into google drive … assign to corresponding folder name in "folders" table”

#### Node: define full_path length
- **Type / role:** Set; computes `order = full_path.split("/").length`
- **Purpose:** used to sort from shallow to deep.
- **Edge cases:** full_path missing/empty → expression error.

#### Node: Sort to get root folders first
- **Type / role:** Sort by `order` ascending.

#### Node: One run per folder
- **Type / role:** SplitInBatches; processes folders sequentially (reduces Drive API bursts).
- **Connections:** Second output triggers `Get the parentFolderName if it exists` (loop pattern).

#### Node: Get the parentFolderName if it exists
- **Type / role:** Data Table get; fetch parent folder row by `parentFolderName`.
- **alwaysOutputData:** true (prevents breaking flow when not found).

#### Node: get parentFolderId if it exists
- **Type / role:** Set; determines the Google Drive parent folder ID:
  - If no parent row found (`Object.keys($json).length === 0`) → parent is the backup run root folder ID.
  - Else → parent is `$json.google_folder_id` from parent row.
- **Key field name caveat:** it creates a field named `=parent_google_folder_Id` (note the leading `=` in the name).  
  Downstream nodes reference `$json.parent_google_folder_Id` (without `=`). This mismatch is likely a bug unless n8n normalizes it in UI.
- **Edge cases:** parent row not yet created in Drive → will fallback to run root, flattening structure.

#### Node: Search if folder already exists
- **Type / role:** Google Drive search folder by name under a parent folder ID.
- **Query:** `mimeType='application/vnd.google-apps.folder' and name='{{folderName}}'`
- **alwaysOutputData:** true
- **Edge cases:** duplicate folders with same name under same parent → returns multiple; later logic assumes empty vs non-empty.

#### Node: if folder not found => create it
- **Type / role:** If; checks `{{$json.isEmpty()}}` (expects search result array behavior).
- **Edge cases:** If Google node returns a non-array object, `.isEmpty()` may not exist → expression failure.

#### Node: Attach the parentFolder if any
- **Type / role:** Set; prepares `subfolder` and `parentFolder` from the batch item.

#### Node: if parentFolder empty
- **Type / role:** If; compares `parentFolder == "null"` (string).
- **True branch (root folder):** `Create folder under the static "backup_n8n" folder`
- **False branch (subfolder):** `get  parent_google_folderid value` + `Merge6` then `Create folder under the parentFolder`

#### Node: Create folder under the static "backup_n8n" folder
- **Type / role:** Google Drive create folder under backup run root (`main_root_folder_google_folder_id`).
- **Output normalized by:** `rename fields2` then upsert.

#### Node: get  parent_google_folderid value
- **Type / role:** Data Table get; fetches parent row by `folderName == parentFolder`.
- **Edge cases:** folderName collisions again; may fetch wrong parent.

#### Node: Merge6
- **Type / role:** Merge combine-by-position; combines parent folder lookup with current folder info before creating subfolder.

#### Node: Create folder under the parentFolder
- **Type / role:** Google Drive create folder under `google_folder_id` (parent’s Google folder ID).
- **Output normalized by:** `rename fields3` then upsert.

#### Node: Add data about folder : google_folder_id / ...google_folder_id1
- **Type / role:** Data Table upsert; stores `google_folder_id` for the created folder (matched by folderName).

---

### Block 8 — Assign `parent_google_folderid` in `folders` Table (Used for Workflow Upload Mapping)
**Overview:** Determines for every folder which Google Drive folder ID should be used as its parent, and writes that into `parent_google_folderid`.  
**Nodes involved:**  
`get all folders1`, `get directParentFolderName`, `Split root vs subfolders`, `parentFolder`, `rename fields5`, `Merge4`, `Remove Duplicates1`, `rename fields4`, `Upsert row(s)5`, `rename fields6`, `Upsert row(s)4`, `Remove Duplicates1`  
**Sticky notes:**  
- “# 7/ Assign "parent_google_folderid" value in "folders" table”  
- “## Assign "parent_google_folderid" value to root folders”  
- “## Assign "parent_google_folderid" value to sub folders”

#### Node: get all folders1
- **Type / role:** Data Table get; reads all folders.

#### Node: get directParentFolderName
- **Type / role:** Set; extracts the immediate parent folder name from `full_path`.
- **Expression:** if depth > 1 uses the segment before last, else uses `main_root_folder_name`.
- **Edge cases:** full_path format differences; leading/trailing slashes.

#### Node: Split root vs subfolders
- **Type / role:** If; `parentFolderName == "null"` → root, else subfolder.

##### Root branch
- `rename fields6` sets `parent_google_folderid = main_root_folder_google_folder_id`
- `Upsert row(s)4` writes back to `folders`

##### Subfolder branch
- `parentFolder` fetches parent folder row by `directParentFolderName`
- `rename fields5` creates temporary keys for merge (`parentFolderName_branch`, `parent_google_folderid_branch`)
- `Merge4` enrichInput1 joining on parent name vs direct parent
- `Remove Duplicates1` by `folderName`
- `rename fields4` maps to final schema
- `Upsert row(s)5` updates `folders`

**Core risk:** All joins are by folder **name**, not ID or full_path → duplicate names in different branches can corrupt mapping.

---

### Block 9 — Build and Store Workflow Metadata in `workflows` Table
**Overview:** Fetches workflows, filters out archived ones, fetches each workflow details, joins with folder details (including full path), then upserts into `workflows` table.  
**Nodes involved:**  
`get all workflows`, `extract each workflow`, `keep only workflows not archived`, `GET each workflow JSON data`, `Match workflow with folder details`, `Merge`, `Remove Duplicates`, `rename fields`, `Insert workflow details into "workflows" table`  
**Sticky note:** “# 5/ Get workflow + folder relation with full path and add it to "workflows" table”

#### Node: get all workflows
- **Type / role:** HTTP Request; `GET /rest/workflows`
- **Auth:** Cookie from `n8n-auth for n8n API`
- **Edge cases:** pagination not handled here (no skip/take); large instances may truncate depending on API defaults.

#### Node: extract each workflow
- **Type / role:** SplitOut `data`.

#### Node: keep only workflows not archived
- **Type / role:** Filter; `isArchived == false`.

#### Node: GET each workflow JSON data
- **Type / role:** HTTP Request; `GET /rest/workflows/{{id}}`
- **Outputs:** feeds both `Merge` and `Match workflow with folder details`.

#### Node: Match workflow with folder details
- **Type / role:** Data Table get from `folders` where folderId equals workflow parentFolder.id (or `'0'` if none).
- **Edge cases:** if folder row missing or folderId stored as string mismatched.

#### Node: Merge
- **Type / role:** Merge enrichInput1 joining workflow’s `data.parentFolder.id` to folder row `folderId`.

#### Node: Remove Duplicates
- **Type / role:** Remove duplicates by `data.name`.

#### Node: rename fields
- **Type / role:** Set; creates workflow table schema:
  - workflow_id/name
  - folder_name/id (root folder fallback)
  - parentFolderName/Id
  - `full_path = {{$json.full_path}}/{{$json.data.name}}`
  - placeholders for google IDs
- **Edge cases:** if folder full_path missing, full_path becomes `undefined/...`.

#### Node: Insert workflow details into "workflows" table
- **Type / role:** Data Table upsert by `workflow_id`.

---

### Block 10 — Export Workflows and Upload JSON Files into Correct Google Drive Folder
**Overview:** Retrieves workflows again for uploading, determines whether each workflow is in root or in a folder, converts JSON to file, uploads, then updates the `workflows` table with Google IDs.  
**Nodes involved:**  
`get all workflows1`, `Split Out3`, `keep only workflows not archived1`, `If there parentFolder.isEmpty()`, `Aggregate`, `one item per workflow at main_root_folder`, `Loop Over Items`, `Get row(s)3`, `get main root folder details1`, `Merge5`, `rename fields7`, `Get each workflow JSON data`, `remove the [] wrapper to re-create imported wf later`, `Merge2`, `Convert workflow to JSON file`, `If parentFolder is null`, `root folder : backup_n8n`, `Upload file to gdrive main root folder`, `add workflow file and folder google IDs2`, `root folder : backup_n8n1`, `Upload file to its folder`, `add workflow file and folder google IDs`, `get related folder details1`, `exclude field "id"`, `add wf_folder_name`, `Merge1`, `Remove Duplicates2`, `wf_id + wf_name`  
**Sticky notes:**  
- “# 8/ Upload each JSON workflow file into its google drive folder”  
- “## Branch for workflow(s) at the main_root of your project”  
- “## Branch for workflow(s) under at list one folder of your project”  
- “## workflows at the main root folder location”  
- “## workflows in other folders”

#### Node: get all workflows1 → Split Out3 → keep only workflows not archived1
- Same role as earlier but used for upload pipeline.

#### Node: If there parentFolder.isEmpty()
- **Type / role:** If; splits workflows into:
  - root workflows (no parentFolder)
  - workflows in folders
- **Expression:** `{{$json.parentFolder.isEmpty()}}` (assumes `parentFolder` exists and has `.isEmpty()`).
- **Edge cases:** parentFolder may be `null` (not an object) → expression failure.

##### Root-workflow branch (main root)
- `Aggregate` aggregates items.
- `one item per workflow at main_root_folder` generates index items 1..N (unusual approach).
- `Loop Over Items` loops (SplitInBatches).
- `Get row(s)3` fetches first folder row (limit 1) and triggers:
  - `Insert one row in "folders" for the main_root_backup_folder` (executeOnce)
  - `get main root folder details1`
  - `Merge5`

- `Merge5` combine-by-position → `rename fields7`
- `rename fields7` re-maps fields and sets `google_folder_id = main_root_folder_google_folder_id`
- `Get each workflow JSON data` fetches JSON by workflow id.
- `remove the [] wrapper...` Code node returns `$json.data` (to flatten structure).
- `Merge2` enrichInput2 joining `wf_name` to `name`.
- `Convert workflow to JSON file` ConvertToFile (toJson), fileName = workflow name.
- `If parentFolder is null` routes:
  - root: `root folder : backup_n8n` → `Upload file to gdrive main root folder` → `add workflow file and folder google IDs2`
  - else: goes to folder branch logic.

##### Folder-workflow branch
- `get related folder details1` queries `folders` table by folderName = workflow parentFolder.name.
- `exclude field "id"` removes row `id` field.
- `add wf_folder_name` sets `wf_folder_name = parentFolder.name`.
- `Merge1` joins folder details to workflows by `wf_folder_name` ↔ `folderName`.
- `Remove Duplicates2` by `name`.
- Then converges with `Get each workflow JSON data` and conversion/upload:
  - `root folder : backup_n8n1` sets main_root_folder from Merge1
  - `Upload file to its folder` uploads to `$('Merge1').item.json.google_folder_id` else fallback to `$json.main_root_folder`
  - Updates `workflows` table via `add workflow file and folder google IDs`

#### Upload nodes
- **Upload file to gdrive main root folder**
  - Google Drive upload; folderId = `$('rename fields7').item.json.google_folder_id`
- **Upload file to its folder**
  - Google Drive upload; folderId = `$('Merge1').item.json.google_folder_id` or fallback.

#### Table update nodes
- **add workflow file and folder google IDs / ...IDs2**
  - Data Table update on `workflows` where `workflow_name == {{$json.name}}`
  - Sets:
    - `workflow_google_file_id = {{$json.id}}`
    - `folder_google_id = {{$json.parents[0]}}`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Starts periodic backup run | — | n8n instance/project access details | ## Backup n8n workflows to Google Drive with automatic cleanup (full note content applies) |
| n8n instance/project access details | set | Stores n8n URL + login credentials | Schedule Trigger | n8n login API1 | ## Add your n8n URL and credentials here |
| n8n login API1 | httpRequest | Login to n8n API, get session cookie | n8n instance/project access details | n8n-auth for n8n API | ## Connect to the account using API to get the "set-cookie" token value… |
| n8n-auth for n8n API | set | Extract `set-cookie` header into `n8n-auth` | n8n login API1 | get projectId; get subfolder(s) structure | ## Connect to the account using API to get the "set-cookie" token value… |
| get projectId | httpRequest | Fetch projects list to use projectId | n8n-auth for n8n API | create n8n table "folders"; create n8n table "workflows" | # 2/ Create the n8n data tables "folders" & "workflows" |
| create n8n table "folders" | httpRequest | Create Data Table `folders` (if missing) | get projectId | "folders" table columns list; get n8n table "id" | # 2/ Create the n8n data tables "folders" & "workflows" |
| "folders" table columns list | code | Emit folder table column definitions | create n8n table "folders" | add columns to "folders" table | ## n8n tables used… https://developers.google.com/workspace/sheets/api/limits |
| add columns to "folders" table | httpRequest | Add columns to `folders` table | "folders" table columns list | dataTableId "folders" | # 2/ Create the n8n data tables "folders" & "workflows" |
| get n8n table "id" | httpRequest | List data tables to resolve `folders` table ID | create n8n table "folders" | Split Out | ## Capture both n8n dataTableId… |
| Split Out | splitOut | Split `data.data` list of tables | get n8n table "id" | Filter on "folders" table | ## Capture both n8n dataTableId… |
| Filter on "folders" table | filter | Keep row where table name == folders | Split Out | Purge data in "folders"; dataTableId "folders" | ## Capture both n8n dataTableId… |
| Purge data in "folders" | dataTable | Delete all rows in folders table | Filter on "folders" table | — |  |
| dataTableId "folders" | set | Store `folders_dataTableId` | Filter on "folders" table; add columns to "folders" table | Merge dataTableIds for "folders" & "workflows" | ## Capture both n8n dataTableId… |
| create n8n table "workflows" | httpRequest | Create Data Table `workflows` (if missing) | get projectId | "workflows" table columns list; get n8n table "id"1 | # 2/ Create the n8n data tables "folders" & "workflows" |
| "workflows" table columns list | code | Emit workflow table column definitions | create n8n table "workflows" | add columns to "workflows" table | ## n8n tables used… https://developers.google.com/workspace/sheets/api/limits |
| add columns to "workflows" table | httpRequest | Add columns to `workflows` table | "workflows" table columns list | dataTableId "workflows" | # 2/ Create the n8n data tables "folders" & "workflows" |
| get n8n table "id"1 | httpRequest | List data tables to resolve `workflows` table ID | create n8n table "workflows" | Split Out2 | ## Capture both n8n dataTableId… |
| Split Out2 | splitOut | Split `data.data` list of tables | get n8n table "id"1 | Filter on "workflows" table | ## Capture both n8n dataTableId… |
| Filter on "workflows" table | filter | Keep row where table name == workflows | Split Out2 | Purge data in "workflows"; dataTableId "workflows" | ## Capture both n8n dataTableId… |
| Purge data in "workflows" | dataTable | Delete all rows in workflows table | Filter on "workflows" table | — |  |
| dataTableId "workflows" | set | Store `workflows_dataTableId` | Filter on "workflows" table; add columns to "workflows" table | Merge dataTableIds for "folders" & "workflows" | ## Capture both n8n dataTableId… |
| Merge dataTableIds for "folders" & "workflows" | merge | Combine both IDs into one item | dataTableId "folders"; dataTableId "workflows" | Use dataTableIds for "folders" & "workflows" | ## Capture both n8n dataTableId… |
| Use dataTableIds for "folders" & "workflows" | merge | Provide IDs for later Data Table nodes | Merge dataTableIds for "folders" & "workflows"; get subfolder(s) structure | Split Out1 | ## Capture both n8n dataTableId… |
| get subfolder(s) structure | httpRequest | Fetch workflows list incl folders metadata | n8n-auth for n8n API | Use dataTableIds for "folders" & "workflows" | # 3/ Get project folder(s)… |
| Split Out1 | splitOut | Split API `data` array | Use dataTableIds… | If resource == "folder" | # 3/ Get project folder(s)… |
| If resource == "folder" | if | Keep only folder resource rows | Split Out1 | rename fields1 | # 3/ Get project folder(s)… |
| rename fields1 | set | Normalize folder fields for table | If resource == "folder" | Insert data into table "folders" | # 3/ Get project folder(s)… |
| Insert data into table "folders" | dataTable | Upsert folder rows | rename fields1 | Aggregate3 | # 3/ Get project folder(s)… |
| Aggregate3 | aggregate | Fan-out aggregated data to multiple branches | Insert data into table "folders" | get all workflows; Get all folders; Create backup folder…; get all folders1; get all workflows1 |  |
| Get all folders | dataTable | Load folders table rows | Aggregate3 | Define & assign folder's fullPath value | # 4/ Assign the folder's fullPath value |
| Define & assign folder's fullPath value | code | Compute folder `full_path` recursively | Get all folders | Insert "full_path" value for each folder | # 4/ Assign the folder's fullPath value |
| Insert "full_path" value for each folder | dataTable | Upsert folder rows with full_path | Define & assign… | — | # 4/ Assign the folder's fullPath value |
| Create backup folder "n8n_backup_folder_structure_ddMMyyyy_HHmmss" | googleDrive | Create timestamped backup root folder | Aggregate3 | get all folders2 | # 6/ Upload the n8n folder structure in gdrive… |
| get all folders2 | dataTable | Load folders table rows (for update) | Create backup folder…; Aggregate3 | Merge3 (index 1); rename main_root_folder fields | # 6/ Upload the n8n folder structure in gdrive… |
| rename main_root_folder fields | set | Attach backup root folder name/id | get all folders2 | add backup_folder google_id to "folders" |  |
| add backup_folder google_id to "folders" | dataTable | Upsert backup root folder id into folders rows | rename main_root_folder fields | Merge3 | # 6/ Upload the n8n folder structure in gdrive… |
| Merge3 | merge | Combine to prepare ordering | add backup_folder…; get all folders2 | define full_path length | ## We order the folders… (sorting explanation) |
| define full_path length | set | Compute path depth `order` | Merge3 | Sort to get root folders first | ## We order the folders… (sorting explanation) |
| Sort to get root folders first | sort | Sort folders by order asc | define full_path length | One run per folder | ## We order the folders… (sorting explanation) |
| One run per folder | splitInBatches | Loop through folders for creation | Sort to get root folders first; Add data about folder…; Add data about folder…1 | Get the parentFolderName if it exists | # 6/ Upload the n8n folder structure in gdrive… |
| Get the parentFolderName if it exists | dataTable | Lookup parent folder row | One run per folder | get parentFolderId if it exists |  |
| get parentFolderId if it exists | set | Choose Google parent folder ID | Get the parentFolderName… | Search if folder already exists |  |
| Search if folder already exists | googleDrive | Search folder by name under parent | get parentFolderId… | if folder not found => create it |  |
| if folder not found => create it | if | If search is empty, create folder | Search if folder already exists | Attach the parentFolder if any |  |
| Attach the parentFolder if any | set | Prepare `subfolder` and `parentFolder` | if folder not found… | if parentFolder empty |  |
| if parentFolder empty | if | Root vs subfolder creation branch | Attach the parentFolder… | Create folder under the static…; Merge6 & get parent_google_folderid value |  |
| Create folder under the static "backup_n8n" folder | googleDrive | Create root folder under backup run root | if parentFolder empty (true) | rename fields2 | ## root folder(s) : full_path = 1 |
| rename fields2 | set | Map created folder name/id | Create folder under static… | Add data about folder : google_folder_id1 |  |
| Add data about folder : google_folder_id1 | dataTable | Upsert google_folder_id for root folder | rename fields2 | One run per folder | ## We get the id of each created folder… |
| get  parent_google_folderid value | dataTable | Lookup parent’s google_folder_id | if parentFolder empty (false) | Merge6 |  |
| Merge6 | merge | Combine current folder with parent lookup | if parentFolder empty (false); get parent_google_folderid value | Create folder under the parentFolder |  |
| Create folder under the parentFolder | googleDrive | Create subfolder under parent google_folder_id | Merge6 | rename fields3 | ## subfolders |
| rename fields3 | set | Map created folder name/id | Create folder under the parentFolder | Add data about folder : google_folder_id |  |
| Add data about folder : google_folder_id | dataTable | Upsert google_folder_id for subfolder | rename fields3 | One run per folder | ## We get the id of each created folder… |
| get all folders1 | dataTable | Load folders rows for parent mapping | Aggregate3 | get directParentFolderName | # 7/ Assign "parent_google_folderid"… |
| get directParentFolderName | set | Extract direct parent name from full_path | get all folders1 | Split root vs subfolders | # 7/ Assign "parent_google_folderid"… |
| Split root vs subfolders | if | Root vs subfolder mapping | get directParentFolderName | rename fields6; parentFolder & Merge4 | # 7/ Assign "parent_google_folderid"… |
| rename fields6 | set | Set parent_google_folderid for root folders | Split root vs subfolders (true) | Upsert row(s)4 | ## Assign "parent_google_folderid" value to root folders |
| Upsert row(s)4 | dataTable | Update folders rows (root mapping) | rename fields6 | — | ## Assign "parent_google_folderid" value to root folders |
| parentFolder | dataTable | Lookup parent row by directParentFolderName | Split root vs subfolders (false) | rename fields5 | ## Assign "parent_google_folderid" value to sub folders |
| rename fields5 | set | Prep merge keys with parent google id | parentFolder | Merge4 |  |
| Merge4 | merge | Enrich child folder with parent google id | rename fields5; Split root vs subfolders | Remove Duplicates1 |  |
| Remove Duplicates1 | removeDuplicates | Deduplicate folder rows by folderName | Merge4 | rename fields4 |  |
| rename fields4 | set | Output folderName + parent_google_folderid | Remove Duplicates1 | Upsert row(s)5 |  |
| Upsert row(s)5 | dataTable | Update folders rows (subfolder mapping) | rename fields4 | — | ## Assign "parent_google_folderid" value to sub folders |
| get all workflows | httpRequest | List workflows | Aggregate3 | extract each workflow | # 5/ Get workflow + folder relation… |
| extract each workflow | splitOut | Split workflows list | get all workflows | keep only workflows not archived | # 5/ Get workflow + folder relation… |
| keep only workflows not archived | filter | Remove archived workflows | extract each workflow | GET each workflow JSON data | # 5/ Get workflow + folder relation… |
| GET each workflow JSON data | httpRequest | Get workflow details (for table) | keep only workflows not archived | Merge; Match workflow with folder details | # 5/ Get workflow + folder relation… |
| Match workflow with folder details | dataTable | Lookup folder by parentFolder.id | GET each workflow JSON data | Merge |  |
| Merge | merge | Join workflow with folder full_path | GET each workflow JSON data; Match workflow… | Remove Duplicates |  |
| Remove Duplicates | removeDuplicates | Deduplicate workflows by name | Merge | rename fields |  |
| rename fields | set | Map workflow fields for workflows table | Remove Duplicates | Insert workflow details into "workflows" table |  |
| Insert workflow details into "workflows" table | dataTable | Upsert workflow rows | rename fields | — | # 5/ Get workflow + folder relation… |
| get all workflows1 | httpRequest | List workflows (for upload) | Aggregate3 | Split Out3 | # 8/ Upload each JSON workflow file… |
| Split Out3 | splitOut | Split workflows list | get all workflows1 | keep only workflows not archived1 | # 8/ Upload each JSON workflow file… |
| keep only workflows not archived1 | filter | Remove archived workflows | Split Out3 | If there parentFolder.isEmpty() | # 8/ Upload each JSON workflow file… |
| If there parentFolder.isEmpty() | if | Root vs folder workflow branch | keep only workflows not archived1 | Aggregate; add wf_folder_name & get related folder details1 |  |
| Aggregate | aggregate | Aggregate root workflows | If there parentFolder.isEmpty() (true) | one item per workflow at main_root_folder |  |
| one item per workflow at main_root_folder | code | Generate index items 1..N | Aggregate | Loop Over Items |  |
| Loop Over Items | splitInBatches | Iterate root workflows | one item per workflow… | Get row(s)3 (loop) |  |
| Get row(s)3 | dataTable | Get one folders row (limit 1) | Loop Over Items | Insert one row…; get main root folder details1; Merge5 |  |
| Insert one row in "folders" for the main_root_backup_folder | dataTable | Insert synthetic main root row | Get row(s)3 | — |  |
| get main root folder details1 | dataTable | Get workflow table rows with folder_name=root folder | Get row(s)3 | Merge5 |  |
| Merge5 | merge | Combine data needed for root workflow upload | Get row(s)3; get main root folder details1 | rename fields7 |  |
| rename fields7 | set | Prepare workflow id + google root folder id | Merge5 | Get each workflow JSON data |  |
| Get each workflow JSON data | httpRequest | Get workflow JSON (for upload) | Remove Duplicates2; rename fields7 | wf_id + wf_name; remove the [] wrapper… |  |
| remove the [] wrapper to re-create imported wf later | code | Flatten `$json.data` | Get each workflow JSON data | Merge2 | remove the [] wrapper to re-create imported wf later |
| wf_id + wf_name | set | Extract wf_id + wf_name | Get each workflow JSON data | Merge2 |  |
| Merge2 | merge | Join wf metadata to workflow JSON | wf_id + wf_name; remove the [] wrapper… | Convert workflow to JSON file |  |
| Convert workflow to JSON file | convertToFile | Create JSON file binary | Merge2 | If parentFolder is null |  |
| If parentFolder is null | if | Route upload root vs folder | Convert workflow to JSON file | root folder : backup_n8n; root folder : backup_n8n1 |  |
| root folder : backup_n8n | set | Set main_root_folder for root upload | If parentFolder is null (true) | Upload file to gdrive main root folder | ## workflows at the main root folder location |
| Upload file to gdrive main root folder | googleDrive | Upload JSON file to backup root | root folder : backup_n8n | add workflow file and folder google IDs2 | ## workflows at the main root folder location |
| add workflow file and folder google IDs2 | dataTable | Update workflows table with Google IDs | Upload file to gdrive main root folder | — |  |
| root folder : backup_n8n1 | set | Set main_root_folder for folder upload | If parentFolder is null (false) | Upload file to its folder | ## workflows in other folders |
| Upload file to its folder | googleDrive | Upload JSON file to mapped folder | root folder : backup_n8n1 | add workflow file and folder google IDs | ## workflows in other folders |
| add workflow file and folder google IDs | dataTable | Update workflows table with Google IDs | Upload file to its folder | — |  |
| get related folder details1 | dataTable | Lookup folder row for workflow parent | If there parentFolder.isEmpty() (false) | exclude field "id" |  |
| exclude field "id" | set | Remove `id` from folder row | get related folder details1 | Merge1 |  |
| add wf_folder_name | set | Add wf_folder_name for join | If there parentFolder.isEmpty() (false) | Merge1 |  |
| Merge1 | merge | Join workflow with folder mapping | add wf_folder_name; exclude field "id" | Remove Duplicates2 |  |
| Remove Duplicates2 | removeDuplicates | Deduplicate workflows by name | Merge1 | Get each workflow JSON data |  |

> Sticky Note nodes themselves are omitted from the table above **only because they are not executable logic**; all sticky note contents have been preserved in sections and rows where relevant.

---

## 4. Reproducing the Workflow from Scratch (Step-by-Step)

1. **Create Trigger**
   1) Add **Schedule Trigger**  
      - Interval: every **4 hours** (adjust as needed)

2. **Add n8n instance configuration**
   1) Add **Set** node: `n8n instance/project access details`  
      - Fields:
        - `n8n_instance_URL` (no trailing slash)
        - `emailOrLdapLoginId`
        - `password`

3. **Login to n8n API and capture cookie**
   1) Add **HTTP Request** node: `n8n login API1`  
      - Method: POST  
      - URL: `{{$json.n8n_instance_URL}}/rest/login`  
      - Body (JSON): email/password from Set node  
      - Response: enable **Full Response** (so headers are available)
   2) Add **Set** node: `n8n-auth for n8n API`  
      - Field `n8n-auth` = `{{$json.headers["set-cookie"][0]}}`

4. **Get Project ID**
   1) Add **HTTP Request**: `get projectId`  
      - GET `{{$('n8n instance/project access details').item.json.n8n_instance_URL}}/rest/projects`  
      - Header: `Cookie` = `{{$json["n8n-auth"]}}`

5. **Create Data Tables (folders + workflows)**
   1) Add **HTTP Request**: `create n8n table "folders"` (continue on error)  
      - POST `/rest/projects/{{projectId}}/data-tables`  
      - Body: `{ "name":"folders","columns":[],"hasHeaders":true }`
   2) Add **Code**: `"folders" table columns list`  
      - Emit column objects: `name` + `type:"string"` for all folder columns.
   3) Add **HTTP Request**: `add columns to "folders" table"`  
      - POST `/columns` endpoint for the created folders table ID.
   4) Repeat 1–3 similarly for **workflows** (`create n8n table "workflows"`, `"workflows" table columns list`, `add columns to "workflows" table"`)

6. **Resolve Data Table IDs even if already existed**
   1) Add **HTTP Request**: `get n8n table "id"`  
      - GET `/rest/projects/{{projectId}}/data-tables`
   2) **SplitOut**: split `data.data`
   3) **Filter**: keep `.name == "folders"`
   4) **Set**: `dataTableId "folders"` with `folders_dataTableId`
   5) Repeat for workflows table (`get n8n table "id"1`, etc.)
   6) **Merge**: `Merge dataTableIds for "folders" & "workflows"` (combine by position)
   7) **Merge**: `Use dataTableIds for "folders" & "workflows"` to carry these forward

7. **Purge both tables**
   1) Add **Data Table** deleteRows nodes:
      - `Purge data in "folders"`
      - `Purge data in "workflows"`

8. **Fetch folders structure and fill folders table**
   1) Add **HTTP Request**: `get subfolder(s) structure`  
      - GET `/rest/workflows?includeScopes=true&includeFolders=true&skip=0&take=400`  
      - Cookie header from `n8n-auth`
   2) Add **SplitOut** on `data`
   3) Add **If**: keep `data.resource == "folder"`
   4) Add **Set**: normalize fields (folderId, folderName, parentFolderName/Id, counts, placeholders)
   5) Add **Data Table upsert**: `Insert data into table "folders"` (match by folderName, as in original)

9. **Compute full_path for folders**
   1) Add **Data Table get**: `Get all folders`
   2) Add **Code**: `Define & assign folder's fullPath value` (recursive build)
   3) Add **Data Table upsert**: `Insert "full_path" value for each folder`

10. **Create Google Drive backup run root folder**
   1) Create a static Drive folder manually (e.g., `backup_n8n`)
   2) Add **Google Drive** node: `Create backup folder "n8n_backup_folder_structure_ddMMyyyy_HHmmss"`  
      - Resource: Folder  
      - Parent Folder: the static `backup_n8n` folder  
      - Name: `n8n_backup_folder_structure_{{$now.format('ddMMyyyy_HHmmss')}}`  
      - Configure **Google Drive OAuth2** credentials
   3) Load folders table again and write `main_root_folder_google_folder_id` into each row (upsert)

11. **Create Google Drive folders in correct order**
   1) Compute `order = full_path depth`
   2) Sort ascending
   3) Split in batches (1 per iteration)
   4) For each folder:
      - Lookup parent folder’s google_folder_id (or default to backup run root)
      - Search folder under that parent by name
      - Create if missing
      - Upsert created `google_folder_id` into folders table

12. **Store workflow metadata into `workflows` table**
   1) GET `/rest/workflows`
   2) SplitOut `data`
   3) Filter `isArchived == false`
   4) GET `/rest/workflows/{{id}}` for each
   5) Join with folders table by folderId and compute `full_path` for workflow
   6) Upsert into `workflows` table

13. **Export and upload workflows**
   1) Re-list workflows and filter non-archived
   2) For each workflow:
      - GET full JSON
      - Convert to JSON file (ConvertToFile node)
      - If workflow has no parent folder → upload to backup run root folder
      - Else → upload to mapped google_folder_id
      - Update `workflows` table with google file ID + parent folder ID

**Credentials needed**
- **Google Drive OAuth2** (Drive access to create folders/upload files)
- n8n login is done with email/password via API (no n8n credential object in this design)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheets API quotas are restrictive (60 requests/minute/project/user), so n8n Data Tables are used instead | https://developers.google.com/workspace/sheets/api/limits |
| Example: for uploaded folder `https://drive.google.com/drive/u/1/folders/1bbirmn7VdcZknQ0hjzVmthiu4XaAAAB`, the google folder id is `1bbirmn7VdcZknQ0hjzVmthiu4XaAAAB` | Included in sticky note “# 6/ Upload the n8n folder structure in gdrive” |
| You must create the Google Drive folder `backup_n8n` manually and select it as the parent folder in the “Create backup folder …” node | From sticky note near that node |
| The workflow mirrors nested structure such as `Utilities/Error_management/error_alerting.json` into Google Drive | From the main description sticky note |

---

### Important Implementation Warnings (from analysis)
- **Folder name collisions:** The workflow frequently upserts/joins folder rows by `folderName` only. If you have duplicate folder names under different branches, this will mis-map parents and overwrite rows. A robust design should match on **folderId** or **full_path** (unique), not name alone.
- **Potential field-name bug:** `get parentFolderId if it exists` creates a field named `=parent_google_folder_Id` (leading `=`). Downstream uses `$json.parent_google_folder_Id`. Verify and fix to a consistent field name.
- **Pagination limits:** `get subfolder(s) structure` uses `take=400`. If you have more than 400 workflows/folders, you need pagination.

If you want, I can propose a corrected version that eliminates folder-name collisions (by using `folderId` + `parentFolderId` as keys, and storing Drive IDs keyed by `folderId`).