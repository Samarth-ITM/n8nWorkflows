Selectively import workflows from GitHub, including nested folders

https://n8nworkflows.xyz/workflows/selectively-import-workflows-from-github--including-nested-folders-13300


# Selectively import workflows from GitHub, including nested folders

## 1. Workflow Overview

**Purpose:**  
This workflow lets a user **select a GitHub organization/user**, then **choose one repository**, then **discover all `.json` files across nested folders** in that repository, and finally **select which workflow JSON files to import into n8n** via the n8n API. It avoids recursion by using a **queue-based loop** (state object) in a single execution.

**Typical use cases:**
- Import a curated subset of workflow JSON files stored in GitHub (including nested directories).
- Bulk-create workflows in a target n8n instance while avoiding API-incompatible fields.

### 1.1 Input Reception & Repo Owner Selection
User selects the GitHub owner/org from a form; the workflow lists repositories for that org.

### 1.2 Repository Options Preparation & Repo Selection
Repo list is transformed into dynamic JSON-based form options; user selects a single repo.

### 1.3 Queue-Based Repository Traversal (Nested Folder Support)
A loop iterates through directories using `pendingPaths` (queue), accumulating `.json` file entries in `allFiles`.

### 1.4 Workflow File Selection UI
Collected `.json` file names become dynamic checkbox options; user selects which workflow files to import.

### 1.5 Retrieve, Validate, Sanitize & Create Workflows in n8n
Selected files are fetched from GitHub, parsed as JSON, validated, stripped to an API-safe payload, then created in n8n. Success/failure is aggregated and displayed.

---

## 2. Block-by-Block Analysis

### Block 1 — Repo Owner Input & Repository Listing
**Overview:** Collects the GitHub “Repo Owner” from the user and lists all repositories under that organization/user.

**Nodes involved:**
- **Sticky Note** (Configure Repo Owner)
- **On form submission**
- **Get repositories for an organization**
- **Set Repository Name and URL**

#### Sticky Note (Configure Repo Owner)
- **Type / role:** Sticky Note (documentation)
- **Content (used by multiple nodes):**  
  “Configure Repo Owner… Keep Field Name `Repo Owner`… Do not rename…”
- **Failure modes:** none (non-executable)

#### On form submission
- **Type / role:** `formTrigger` (entry point)
- **Key configuration:**
  - Form title: **Repo Loader**
  - Field: **Repo Owner** (dropdown) with options (e.g., `Granite-Marketing`, `sanindooo`)
  - Custom CSS for dark theme styling
  - Path: `github-for-n8n`
- **Key variables:**
  - Output JSON contains `$json["Repo Owner"]`
- **Outputs:** to **Get repositories for an organization**
- **Edge cases / failures:**
  - If the form field name is changed, downstream expressions break.

#### Get repositories for an organization
- **Type / role:** `github` node, resource `organization`, list repositories
- **Key configuration:**
  - Owner: `={{ $json["Repo Owner"] }}`
  - Return all: `true`
  - Credential: GitHub API credential (PAT/OAuth)
- **Input:** from **On form submission**
- **Output:** multiple repo items to **Set Repository Name and URL**
- **Failure modes:**
  - Auth/permission issues (401/403), org visibility limits, rate limiting (403), network timeouts.

#### Set Repository Name and URL
- **Type / role:** `set` node to normalize fields
- **Key configuration:**
  - `repo_name = {{$json.name}}`
  - `repo_url = {{$json.full_name}}`
- **Input:** repo items from GitHub
- **Output:** to **Aggregate Repositories**
- **Edge cases:**
  - Missing `name/full_name` if GitHub API response shape changes (unlikely).

---

### Block 2 — Dynamic Repo Selection Form
**Overview:** Aggregates repository items and builds JSON-mode form options for a single-choice repo selection.

**Nodes involved:**
- **Sticky Note1** (Repository form structure)
- **Aggregate Repositories**
- **Create JSON Repo Options**
- **Select Repos**

#### Sticky Note1
- **Type / role:** Sticky Note
- **Content:** “Create Repository Form Data Structure…”
- **Failure modes:** none

#### Aggregate Repositories
- **Type / role:** `aggregate` (aggregate all items into one)
- **Key configuration:** `aggregateAllItemData`
- **Input:** many repo items
- **Output:** single item containing `data` array to **Create JSON Repo Options**
- **Edge cases:**
  - Very large orgs could create large payloads; UI option list can become unwieldy.

#### Create JSON Repo Options
- **Type / role:** `code` node to format JSON-mode options
- **Key logic:**
  - Reads: `const data = $input.first().json.data;`
  - Produces `options` string like: `{"option":"repo1"},{"option":"repo2"}`
- **Output:** `{ json: { options } }` to **Select Repos**
- **Edge cases / failures:**
  - Repository names containing quotes or special characters could break string-templated JSON.
  - If `data` is undefined (aggregation failed), options creation fails.

#### Select Repos
- **Type / role:** `form` node (dynamic form in JSON mode)
- **Key configuration:**
  - Title: **Select Repos**
  - JSON Output (form definition): creates a **radio** field labelled `Radio`
  - Injects `{{ $json.options }}` into the values array
- **Output:** selected value in `$('Select Repos').first().json.Radio`
- **Next:** to **Initialise Loop State**
- **Edge cases:**
  - If `options` is malformed JSON fragment, the form rendering fails.
  - The downstream expects the field label/key to be exactly `Radio`.

---

### Block 3 — Queue-Based Traversal of Nested Repo Folders
**Overview:** Walks the selected repository starting at root, listing directory contents, queueing subdirectories, and collecting `.json` files into a single `allFiles` array.

**Nodes involved:**
- **Sticky Note7** (Option 1 explanation)
- **Sticky Note3** (Option 2 explanation)
- **Initialise Loop State**
- **If Has Next Page**
- **Set Current Path**
- **List Files**
- **Process Directory Contents**
- **No Operation, do nothing**

#### Sticky Note7
- **Type / role:** Sticky Note
- **Content:** Detailed explanation of state structure and loop.
- **Failure modes:** none

#### Sticky Note3
- **Type / role:** Sticky Note
- **Content:** High-level explanation + setup steps (GitHub credentials, n8n API key, run workflow).
- **Failure modes:** none

#### Initialise Loop State
- **Type / role:** `code` node to create traversal state
- **Key output structure:**
  - `pendingPaths: [""]` (start at repo root)
  - `currentPath: ""`
  - `allFiles: []`
- **Output:** to **If Has Next Page**
- **Edge cases:**
  - None significant; always produces one item state.

#### If Has Next Page
- **Type / role:** `if` node controlling the loop
- **Condition:** `={{ $json.pendingPaths.length > 0 }}`
- **True output:** to **Set Current Path**
- **False output:** to **No Operation, do nothing** (loop finished)
- **Edge cases:**
  - If `pendingPaths` missing/not array: expression error.

#### Set Current Path
- **Type / role:** `code` node that dequeues next directory
- **Logic:**
  - `state.currentPath = state.pendingPaths.shift();`
  - returns updated state
- **Output:** to **List Files**
- **Edge cases:**
  - If `pendingPaths` empty, `currentPath` becomes `undefined`; GitHub node uses `|| ''` later in traversal but here it’s carried forward.

#### List Files
- **Type / role:** `github` node, resource `file`, operation `list` (list directory contents)
- **Key configuration:**
  - Owner: `={{ $('On form submission').first().json["Repo Owner"] }}`
  - Repository: `={{ $('Select Repos').first().json.Radio }}`
  - File path: `={{ $json.currentPath || '' }}`
  - **onError:** `continueRegularOutput` (important: loop continues even if a directory list fails)
- **Output:** entries to **Process Directory Contents**
- **Failure modes:**
  - 404 on directories (path mismatch), permission issues, rate limiting.
  - Because errors continue, you can silently miss files if listing fails.

#### Process Directory Contents
- **Type / role:** `code` node to update traversal state
- **Key logic:**
  - Takes state from `$('Set Current Path').all()[0].json`
  - Reads directory listing entries from `$input.all()`
  - For each entry:
    - if `type === "dir"` → push `entry.path` into `pendingPaths`
    - if `type === "file"` and `name.endsWith(".json")` → push to `allFiles`
- **Output:** updated state back to **If Has Next Page** (loop)
- **Edge cases:**
  - Very large repos: state object can grow big (memory/time).
  - Only files ending in `.json` are collected (intended).
  - If GitHub returns unexpected shapes, filtering may fail.

#### No Operation, do nothing
- **Type / role:** `noOp` node used as the loop exit junction
- **Role in this workflow:** When traversal is complete, it fans out to:
  - **Create JSON Workflow Options**
  - **Split Out All Workflows**
- **Edge cases:** none

---

### Block 4 — Build Workflow Selection Form & Match Selections Back to File Metadata
**Overview:** Converts discovered `.json` files to checkbox options, collects user choices, and merges the selections with the original file list so only chosen workflows continue.

**Nodes involved:**
- **Sticky Note4** (Workflow file form data structure)
- **Split Out All Workflows**
- **Create JSON Workflow Options**
- **Select Workflow(s)**
- **Split Out Selected Workflow(s)**
- **Get Target Repo(s) Data**

#### Sticky Note4
- **Type / role:** Sticky Note
- **Content:** Explains splitting, dynamic options, selection, merge back.
- **Failure modes:** none

#### Split Out All Workflows
- **Type / role:** `splitOut` to turn `allFiles` array into individual items
- **Key configuration:** `fieldToSplitOut = allFiles`
- **Input:** the final traversal state object
- **Output:** each item resembles a GitHub file entry (`name`, `path`, `type`, etc.)
- **Next:** to **Get Target Repo(s) Data** (merge input 1)
- **Edge cases:**
  - If `allFiles` missing/empty, produces no items (selection list will be empty).

#### Create JSON Workflow Options
- **Type / role:** `code` to build checkbox options for selection form
- **Key logic:**
  - Reads: `const data = $input.first().json.allFiles;`
  - Creates `options` string of `{"option":"<filename.json>"}`
- **Output:** to **Select Workflow(s)**
- **Edge cases:**
  - Same quote/escaping issue as repo options if filenames contain quotes.
  - If `allFiles` is large, form becomes long (CSS sets `.multiselect` max-height but checkboxes may still be extensive).

#### Select Workflow(s)
- **Type / role:** `form` node in JSON mode (checkbox selection)
- **Key configuration:**
  - Title: **Create Workflows**
  - Form definition: checkbox field labeled `Checkboxes`
  - Values injected from `{{ $json.options }}`
- **Output:** `Checkboxes` is typically an array of selected option strings (filenames)
- **Next:** **Split Out Selected Workflow(s)**
- **Edge cases:**
  - If user selects none, downstream merge may produce no matches.

#### Split Out Selected Workflow(s)
- **Type / role:** `splitOut` to emit one item per selected checkbox entry
- **Key configuration:** `fieldToSplitOut = Checkboxes`
- **Output:** each item has `Checkboxes = "<filename.json>"`
- **Next:** to **Get Target Repo(s) Data** (merge input 2)
- **Edge cases:**
  - No selections → no items → nothing imported.

#### Get Target Repo(s) Data
- **Type / role:** `merge` node (combine) to match selected names with file entries
- **Key configuration:**
  - Mode: `combine` with **merge by fields**
  - Field mapping: `name` (from file entries) == `Checkboxes` (from selections)
  - Advanced merge enabled
- **Inputs:**
  - Input 1: from **Split Out All Workflows** (file metadata)
  - Input 2: from **Split Out Selected Workflow(s)** (selected names)
- **Output:** matched items (only selected files) to **Get a file**
- **Edge cases / failures:**
  - If filenames are not unique across folders, matching by `name` alone can pick the wrong file or merge multiple paths incorrectly.
  - Safer key would be `path` rather than `name` (not implemented here).

---

### Block 5 — Fetch JSON Files, Parse, Validate, Sanitize for n8n API, Create Workflows
**Overview:** Downloads each selected `.json` file, parses it, ensures it exists, strips to an n8n-API-safe payload, then creates the workflow via n8n API.

**Nodes involved:**
- **Sticky Note5** (Format workflows)
- **Get a file**
- **Extract JSON From File**
- **Set Name and Data**
- **Found Matching Workflow?**
- **Strip Incompatible API Fields**
- **Create a workflow**
- **Merge Success and Failed Workflows**

#### Sticky Note5
- **Type / role:** Sticky Note
- **Content:** Verifies retrieval, removes incompatible fields, creates/updates, merges results.
- **Failure modes:** none

#### Get a file
- **Type / role:** `github` node, resource `file`, operation `get`
- **Key configuration:**
  - Owner: `={{ $('On form submission').first().json["Repo Owner"] }}`
  - Repository: `={{ $('Select Repos').first().json.Radio }}`
  - File path: `={{ $json.path }}`
- **Input:** merged selected file item containing `path`
- **Output:** file content (binary or data depending on GitHub node behavior) to **Extract JSON From File**
- **Failure modes:**
  - 404 if file moved, permission errors, rate limit.
  - GitHub file API may return base64 content; n8n GitHub node typically handles this, but mismatches can break parsing.

#### Extract JSON From File
- **Type / role:** `extractFromFile` operation `fromJson` to parse JSON
- **Input:** output of **Get a file**
- **Output:** parsed JSON object (the workflow JSON) to **Set Name and Data**
- **Edge cases:**
  - Invalid JSON content → parsing failure.
  - If GitHub node returns unexpected format (not a file/binary), this node can fail.

#### Set Name and Data
- **Type / role:** `set` node to normalize fields for downstream
- **Key configuration:**
  - `name = {{ $('Get Target Repo(s) Data').item.json.name }}` (the filename)
  - `data = {{ $json.data }}` (parsed workflow JSON payload)
  - `is_empty = {{ $json.data.isEmpty() }}` (marks missing/empty)
- **Output:** to **Found Matching Workflow?**
- **Edge cases / failures:**
  - `isEmpty()` assumes `data` is an object/array with that helper available in n8n expressions; if `data` is null/undefined it can error.
  - If multiple items are processed, referencing `$('Get Target Repo(s) Data').item` assumes correct item pairing.

#### Found Matching Workflow?
- **Type / role:** `if` node validating that `data` exists
- **Condition:** `{{$json.data}}` is **not empty**
- **True:** to **Strip Incompatible API Fields**
- **False:** to **Merge Success and Failed Workflows** (input index 1)
- **Edge cases:**
  - “Not empty” checks object presence, but doesn’t validate it’s a valid n8n workflow structure.

#### Strip Incompatible API Fields
- **Type / role:** `code` node (run once per item) to build strict allowlisted workflow payload
- **Key behavior:**
  - Constructs workflow payload with only:
    - `name`
    - `nodes` (cleaned nodes containing: `id,name,type,typeVersion,position,parameters,credentials,disabled,notes,notesInFlow`)
    - `connections`
    - optional `settings` allowlist: `timezone, executionTimeout, saveExecutionProgress, saveManualExecutions, errorWorkflowId`
    - optional `staticData`
- **Output:** to **Create a workflow**
- **Failure modes / edge cases:**
  - If source workflow JSON uses required properties outside the allowlist (rare but possible), imported workflow may behave differently.
  - Credentials: copying `credentials` blocks can create references to credential names/ids that don’t exist in the target n8n (workflow imports but nodes may be unconfigured).
  - Some nodes may require additional properties depending on n8n version; strict allowlist could omit them.

#### Create a workflow
- **Type / role:** `n8n` node, operation `create` (calls n8n REST API)
- **Key configuration:**
  - Workflow object: `={{ $json.toJsonString() }}`
  - Credential: n8n API credential (API Key / base URL)
- **Output:** to **Merge Success and Failed Workflows** (input index 0)
- **Failure modes:**
  - 401/403 if API key invalid or lacks permissions.
  - 400 if payload invalid (missing nodes/connections mismatch).
  - Version mismatches: older/newer node types may not exist in target instance.

#### Merge Success and Failed Workflows
- **Type / role:** `merge` node to unify both success and failure paths
- **Inputs:**
  - Input 0: successful create responses
  - Input 1: failed/empty data path
- **Output:** to **Aggregate Selected Workflows**
- **Edge cases:**
  - Mixed shapes: success output differs from “failed path” output; later nodes rely on consistent `name`, `data`, `is_empty` fields—if success path doesn’t preserve them, summary may be inconsistent. (In this workflow, success path returns n8n API response; the summary later uses `item.is_empty`/`item.name`, so ensure these fields still exist or are reconstructed.)

---

### Block 6 — Aggregate Results & Display Completion Message
**Overview:** Aggregates all processed items and renders a final “Results” completion screen listing success/failure per workflow.

**Nodes involved:**
- **Sticky Note6** (Structure success message)
- **Aggregate Selected Workflows**
- **Results**

#### Sticky Note6
- **Type / role:** Sticky Note
- **Content:** Explains formatting summary.
- **Failure modes:** none

#### Aggregate Selected Workflows
- **Type / role:** `aggregate` to collect all items into a single `data` array
- **Key configuration:** `aggregateAllItemData`
- **Input:** from **Merge Success and Failed Workflows**
- **Output:** one item to **Results**
- **Edge cases:**
  - If upstream outputs very large items, aggregation can be heavy.

#### Results
- **Type / role:** `form` node in **completion** mode (final output screen)
- **Key configuration:**
  - Completion title: “Results”
  - Completion message expression:
    - Iterates: `$json.data.map(item => ...)`
    - Marks failed if `item.is_empty === true`, else success
- **Edge cases / failures:**
  - If aggregated items do not contain `is_empty` and `name` consistently, message may be incorrect or throw expression errors.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Documentation: configure Repo Owner field |  |  | ## Configure Repo Owner… Do not rename the field or change its type, as other nodes depend on it. |
| On form submission | formTrigger | Entry point; collects Repo Owner |  | Get repositories for an organization | ## Configure Repo Owner… Do not rename the field or change its type, as other nodes depend on it. |
| Get repositories for an organization | github | List repos in selected org/owner | On form submission | Set Repository Name and URL |  |
| Set Repository Name and URL | set | Normalize repo fields for UI | Get repositories for an organization | Aggregate Repositories |  |
| Aggregate Repositories | aggregate | Combine repo items into one array | Set Repository Name and URL | Create JSON Repo Options | ## Create Repository Form Data Structure… |
| Create JSON Repo Options | code | Build JSON-mode options for repo radio form | Aggregate Repositories | Select Repos | ## Create Repository Form Data Structure… |
| Select Repos | form | Dynamic repo selection (radio) | Create JSON Repo Options | Initialise Loop State |  |
| Initialise Loop State | code | Initialize traversal queue/state | Select Repos | If Has Next Page | ## How it works 🧠 (Option 1)… Traversal controlled through a single state object. |
| If Has Next Page | if | Loop condition: pendingPaths remaining | Initialise Loop State; Process Directory Contents | Set Current Path (true); No Operation, do nothing (false) | ## How it works 🧠 (Option 1)… |
| Set Current Path | code | Dequeue next directory path | If Has Next Page (true) | List Files | ## How it works 🧠 (Option 1)… |
| List Files | github | List directory contents in repo | Set Current Path | Process Directory Contents | ## How it works 🧠 (Option 1)… |
| Process Directory Contents | code | Queue subdirs; collect `.json` files | List Files | If Has Next Page | ## How it works 🧠 (Option 1)… |
| No Operation, do nothing | noOp | Loop exit junction; fan-out | If Has Next Page (false) | Create JSON Workflow Options; Split Out All Workflows |  |
| Sticky Note7 | stickyNote | Documentation: traversal technical breakdown |  |  | (same as content shown in workflow) |
| Sticky Note3 | stickyNote | Documentation: overall explanation & setup |  |  | (same as content shown in workflow) |
| Sticky Note1 | stickyNote | Documentation: repo form JSON structure |  |  | Builds the JSON for the dynamic repository selection form (JSON mode). |
| Sticky Note4 | stickyNote | Documentation: workflow file selection block |  |  | Only selected workflow files continue downstream. |
| Split Out All Workflows | splitOut | Convert `allFiles` array to items | No Operation, do nothing | Get Target Repo(s) Data (input 1) | ## Create Workflow File Form Data Structure… Only selected workflow files continue downstream. |
| Create JSON Workflow Options | code | Build checkbox options from discovered files | No Operation, do nothing | Select Workflow(s) | ## Create Workflow File Form Data Structure… Only selected workflow files continue downstream. |
| Select Workflow(s) | form | Dynamic workflow file selection (checkbox) | Create JSON Workflow Options | Split Out Selected Workflow(s) | ## Create Workflow File Form Data Structure… Only selected workflow files continue downstream. |
| Split Out Selected Workflow(s) | splitOut | One item per selected filename | Select Workflow(s) | Get Target Repo(s) Data (input 2) | ## Create Workflow File Form Data Structure… Only selected workflow files continue downstream. |
| Get Target Repo(s) Data | merge | Match selections to file entries (by name) | Split Out All Workflows; Split Out Selected Workflow(s) | Get a file | ## Create Workflow File Form Data Structure… Only selected workflow files continue downstream. |
| Get a file | github | Download selected workflow JSON file | Get Target Repo(s) Data | Extract JSON From File | ## Format Workflows… Builds the correct workflow payload… |
| Extract JSON From File | extractFromFile | Parse file content as JSON | Get a file | Set Name and Data | ## Format Workflows… |
| Set Name and Data | set | Normalize fields; mark empty | Extract JSON From File | Found Matching Workflow? | ## Format Workflows… |
| Found Matching Workflow? | if | Validate workflow JSON exists | Set Name and Data | Strip Incompatible API Fields (true); Merge Success and Failed Workflows (false) | ## Format Workflows… |
| Strip Incompatible API Fields | code | Allowlist-only payload for n8n API | Found Matching Workflow? (true) | Create a workflow | ## Format Workflows… |
| Create a workflow | n8n | Create workflow via n8n API | Strip Incompatible API Fields | Merge Success and Failed Workflows | ## Format Workflows… |
| Merge Success and Failed Workflows | merge | Combine success + failure paths | Create a workflow; Found Matching Workflow? (false) | Aggregate Selected Workflows | ## Format Workflows… |
| Aggregate Selected Workflows | aggregate | Aggregate all results for summary | Merge Success and Failed Workflows | Results | ## Structure Success Message… |
| Results | form | Completion screen with summary text | Aggregate Selected Workflows |  | ## Structure Success Message… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create `On form submission` (Form Trigger)**
   - Node type: **Form Trigger**
   - Set **Path**: `github-for-n8n`
   - Title: `Repo Loader`
   - Add a **Dropdown** field:
     - Field Name: `Repo Owner` (must match exactly)
     - Label: `Repo Owner`
     - Add options for your org/user(s)
     - Set a default value
   - (Optional) paste the provided custom CSS to match styling.

2. **Add GitHub node: `Get repositories for an organization`**
   - Node type: **GitHub**
   - Resource: **Organization**
   - Operation: “Get Many/List” (return all)
   - Owner expression: `={{ $json["Repo Owner"] }}`
   - Credentials: create/select **GitHub API** credential (PAT with repo read access, or OAuth).

3. **Add `Set Repository Name and URL` (Set node)**
   - Set `repo_name = {{$json.name}}`
   - Set `repo_url = {{$json.full_name}}`

4. **Add `Aggregate Repositories` (Aggregate node)**
   - Operation: **Aggregate All Item Data** (one item with `data` array).

5. **Add `Create JSON Repo Options` (Code node)**
   - Build a string of JSON option objects from `data[]` into `json.options` (as in the workflow).
   - Ensure output is: `{ json: { options: "..." } }`.

6. **Add `Select Repos` (Form node)**
   - Node type: **Form**
   - Use **Define Form = JSON**
   - Create a **radio** field (label `Radio`) whose values are injected from `{{$json.options}}`.
   - Button label: “Select Repos”.

7. **Add traversal state init: `Initialise Loop State` (Code node)**
   - Output a single item:
     - `pendingPaths: [""]`, `currentPath: ""`, `allFiles: []`.

8. **Add `If Has Next Page` (IF node)**
   - Condition: `={{ $json.pendingPaths.length > 0 }}`
   - True → continue loop; False → exit loop.

9. **Add `Set Current Path` (Code node)**
   - Dequeue `pendingPaths.shift()` into `currentPath`.

10. **Add `List Files` (GitHub node)**
   - Resource: **File**
   - Operation: **List**
   - Owner: `={{ $('On form submission').first().json["Repo Owner"] }}`
   - Repo: `={{ $('Select Repos').first().json.Radio }}`
   - Path: `={{ $json.currentPath || '' }}`
   - Set **On Error**: *Continue (regular output)* to avoid stopping traversal on a single folder error.

11. **Add `Process Directory Contents` (Code node)**
   - Read all listed entries from `$input.all()`
   - Push dirs to `pendingPaths`, `.json` files to `allFiles`
   - Return the updated **single** state item.

12. **Close the loop**
   - Connect: `Process Directory Contents` → `If Has Next Page`
   - `If Has Next Page` True → `Set Current Path` → `List Files` → `Process Directory Contents`
   - `If Has Next Page` False → `No Operation, do nothing` (NoOp node)

13. **After loop exit, create workflow selection options**
   - Add `Create JSON Workflow Options` (Code):
     - Input: final state item
     - Build `json.options` from `allFiles[]` (typically by filename).
   - Add `Select Workflow(s)` (Form node, JSON mode):
     - Checkbox field label: `Checkboxes`
     - Values injected from `{{$json.options}}`

14. **Split discovered files and selected checkboxes**
   - Add `Split Out All Workflows` (Split Out):
     - Split field: `allFiles`
   - Add `Split Out Selected Workflow(s)` (Split Out):
     - Split field: `Checkboxes`

15. **Merge selections back to file metadata**
   - Add `Get Target Repo(s) Data` (Merge node):
     - Mode: **Combine**
     - Merge by fields: `name` (file entry) equals `Checkboxes` (selection)
   - Note: to support duplicates, consider merging by `path` instead (requires changing the selection to use paths).

16. **Fetch and parse each selected file**
   - Add `Get a file` (GitHub node):
     - Resource: **File**
     - Operation: **Get**
     - Owner: from `On form submission`
     - Repo: from `Select Repos`
     - File path: `={{ $json.path }}`
   - Add `Extract JSON From File` (Extract From File):
     - Operation: **From JSON**

17. **Normalize and validate**
   - Add `Set Name and Data` (Set):
     - `name` from the merged file entry (often filename)
     - `data` from parsed JSON
     - `is_empty` boolean (true when no data)
   - Add `Found Matching Workflow?` (IF):
     - Condition: `data` is not empty
     - False path goes to result merge as “failed”.

18. **Sanitize payload for n8n API**
   - Add `Strip Incompatible API Fields` (Code, run once per item):
     - Build an allowlisted object containing:
       - `name`, `nodes` (cleaned), `connections`
       - optional allowlisted `settings`
       - optional `staticData`

19. **Create workflows in n8n**
   - Add `Create a workflow` (n8n node):
     - Operation: **Create**
     - Workflow object: expression producing JSON string (e.g. `{{$json.toJsonString()}}`)
   - Credentials: configure **n8n API** credential (base URL of your n8n + API key from n8n settings).

20. **Merge results, aggregate, and display**
   - Add `Merge Success and Failed Workflows` (Merge) to combine:
     - Success outputs from n8n create
     - Failure outputs from validation false path
   - Add `Aggregate Selected Workflows` (Aggregate all item data)
   - Add `Results` (Form node, **Completion** mode) with a completion message summarizing success/fail per item.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow depends on the form field name `Repo Owner` and the repo selection field name `Radio`. Changing these requires updating expressions in GitHub nodes. | Configuration dependency |
| Nested folders are supported using a single-execution queue loop (`pendingPaths`) rather than recursion. | Design/architecture note |
| GitHub credentials must have permission to list repository contents and fetch files. | GitHub integration requirement |
| n8n API credential is required to create workflows via `/api/v1/workflows`. | n8n integration requirement |
| The selection merge matches by **filename** (`name`). Duplicate filenames in different folders can cause wrong matches; using `path` is safer if you adapt the selection UI accordingly. | Edge case / improvement suggestion |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.