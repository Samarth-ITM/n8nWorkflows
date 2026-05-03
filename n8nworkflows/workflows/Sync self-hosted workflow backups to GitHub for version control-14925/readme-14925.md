Sync self-hosted workflow backups to GitHub for version control

https://n8nworkflows.xyz/workflows/sync-self-hosted-workflow-backups-to-github-for-version-control-14925


# Sync self-hosted workflow backups to GitHub for version control

Let me carefully review one more time before writing the final output. I want to make sure the IF node analysis for "Any Files to Delete?" is accurate.

Looking at the Code node "Find Orphaned Files to Delete":
- Returns `{ json: { path: f, skipped: false } }` for files to delete
- Returns `{ json: { skipped: true, reason: '...' } }` when nothing to delete

The IF node "Any Files to Delete?" checks: `boolean[{value1: "={{ $json.skipped }}"}]`

In n8n IF node v1:
- When there are files to delete: items have `skipped: false` → boolean check: false → output 1 (false branch, no connection)
- When nothing to delete: item has `skipped: true` → boolean check: true → output 0 (true branch → Delete Orphaned File)

This IS inverted logic. The items that need deletion (skipped: false) go to the unconnected output 1. The "no action needed" item goes to Delete.

Wait... but actually, maybe I need to reconsider. The IF node processes ALL items. When there are files to delete, the code node returns multiple items with `skipped: false`. The IF node would route these to output 1 (false branch, unconnected). The "skipped: true" item wouldn't exist because the code only returns it when there are NO files to delete.

So actually:
- When files need deletion: items with `skipped: false` → output 1 (unconnected) → **files are NOT deleted**
- When nothing to delete: item with `skipped: true` → output 0 → Delete Orphaned File → tries to delete a "skipped" item that has no `path` property

This is definitely a logic bug. The correct behavior would be to invert the condition or swap the connections.

Actually, I wonder if there's something about how the IF node v1 processes boolean conditions without an explicit operation. Let me check if `boolean` condition with only `value1` might be checking "is true" or "is not empty" or something.

In n8n, for boolean conditions in the IF node, you typically need both value1 and value2 (or value1 and operation). But here only value1 is provided. The default operation for boolean is "equal" and the implicit value2 might be `true`.

So the condition is: `$json.skipped === true`
- When skipped=true → true → output 0 → Delete
- When skipped=false → false → output 1 (no connection)

This confirms the logic bug. I'll document it clearly.

Now let me write the full document. 1. Workflow Overview

**Purpose:**  
This workflow automatically synchronizes all n8n workflows from a self-hosted instance to a GitHub repository, providing version-controlled JSON backups on a configurable schedule.

**Target Use Cases:**
- Disaster recovery: restore any workflow from a Git history.
- Audit trail: track when a workflow was last modified (via timestamped commits).
- Cross-instance migration: clone workflows from one n8n instance to another using the repo as the single source of truth.

**Logical Blocks:**

| Block | Name | Role |
|-------|------|------|
| 1 | Scheduling & Trigger | Starts the backup cycle on a fixed interval (default: every hour). |
| 2 | Fetch Current GitHub Repo State | Lists all files already in the backup repository and aggregates their filenames. |
| 3 | Fetch & Filter n8n Workflows | Pulls every workflow from the n8n API, then filters to only those that are new or whose content has changed (size-based diff). |
| 4 | Prepare Binary Payloads | Serializes each changed workflow to pretty-printed JSON, then Base64-encodes it for the GitHub API. |
| 5 | Commit Changed/New Files to GitHub | Builds a hyphenated-lowercase filename and a commit timestamp, then decides whether to create a new file or update an existing one. |
| 6 | Cleanup Orphaned Files | Detects GitHub files that no longer correspond to any n8n workflow (renamed or deleted) and removes them from the repo. |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Scheduling & Trigger

**Overview:**  
A single Schedule Trigger node fires the entire pipeline at a fixed interval. The default configuration runs every hour (one empty interval entry).

**Nodes Involved:**  
- Schedule Trigger

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Version / Requirements | Edge Cases & Failure Types |
|------|------|---------------|---------------|----------------------------|-------|--------|------------------------|---------------------------|
| Schedule Trigger | Schedule Trigger | Time-based entry point | Interval: every 1 hour (single empty interval object — n8n default) | — | — | List Files in GitHub Repo | Type version 1.2 | n8n server clock skew; execution may drift if prior run is still in progress (no concurrency guard built in). |

---

#### Block 2 — Fetch Current GitHub Repo State

**Overview:**  
Queries the GitHub repository for all existing files and aggregates their names into a single item. This state is later used to determine whether a workflow file is new, existing, or orphaned.

**Nodes Involved:**  
- List Files in GitHub Repo
- Collect GitHub Filenames

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Version / Requirements | Edge Cases & Failure Types |
|------|------|---------------|---------------|----------------------------|-------|--------|------------------------|---------------------------|
| List Files in GitHub Repo | GitHub | File → List operation; returns every file in the repo | Owner: `YOUR_GITHUB_USERNAME` (must be replaced); Repository: `n8n_backup`; Resource: File; Operation: List; `alwaysOutputData: true` | — | Schedule Trigger | Collect GitHub Filenames | Type version 1; requires GitHub API credential with `repo` scope | GitHub API rate limit (5 000 req/hr for authenticated); 404 if repo doesn't exist; returns empty array if repo is brand new. |
| Collect GitHub Filenames | Aggregate | Collapses all file-name values into a single array on one item | Aggregated field: `name` | — | List Files in GitHub Repo | Fetch All n8n Workflows | Type version 1 | If GitHub repo is empty, aggregation produces an empty array — downstream nodes must handle it gracefully. |

---

#### Block 3 — Fetch & Filter n8n Workflows

**Overview:**  
Retrieves the full list of workflows from the local n8n instance via its API. A Code node then compares each workflow's current byte size against the corresponding GitHub file size to emit only new or modified workflows, avoiding redundant commits.

**Nodes Involved:**  
- Fetch All n8n Workflows
- Filter Changed/New Workflows

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Version / Requirements | Edge Cases & Failure Types |
|------|------|---------------|---------------|----------------------------|-------|--------|------------------------|---------------------------|
| Fetch All n8n Workflows | n8n | Retrieves all workflows from the local n8n instance | No filters; default request options | — | Collect GitHub Filenames | Filter Changed/New Workflows | Type version 1; requires n8n API credential (API key generated under Settings → API) | 401 if API key is invalid; 403 if key lacks permissions; returns 0 items if instance has no workflows. |
| Filter Changed/New Workflows | Code (JavaScript) | Size-based diff: marks each workflow as `_changed` or `_isNew` and filters to only those needing a commit | Inline JS (see below) | References `$('List Files in GitHub Repo').all()` for GitHub size map; `_fileName`, `_currentSize`, `_githubSize`, `_changed`, `_isNew` | Fetch All n8n Workflows | Convert Workflow to JSON File | Type version 2 | Expression `$('List Files in GitHub Repo')` may fail if that node produced zero items; size-based comparison can yield false positives/negatives (same size but different content — rare but possible). |

**Key logic inside Filter Changed/New Workflows:**
1. Builds a map of `{ fileName → fileSize }` from the GitHub listing.
2. For each n8n workflow:
   - Stringifies the workflow JSON with 2-space indentation.
   - Derives the target filename: workflow name lowercased, spaces replaced with hyphens, `.json` appended.
   - Computes `currentSize` via `Buffer.byteLength`.
   - Looks up `githubSize` from the map.
   - Sets `_isNew` (no GitHub match) and `_changed` (new or size differs).
3. Returns only items where `_changed` or `_isNew` is true.

---

#### Block 4 — Prepare Binary Payloads

**Overview:**  
Converts each filtered workflow item into a pretty-printed JSON binary attachment, then Base64-encodes it so the GitHub API can accept it as file content.

**Nodes Involved:**  
- Convert Workflow to JSON File
- Encode as Base64

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Version / Requirements | Edge Cases & Failure Types |
|------|------|---------------|---------------|----------------------------|-------|--------|------------------------|---------------------------|
| Convert Workflow to JSON File | Convert to File | Serializes each item's JSON to a `.json` binary attachment | Mode: each (one file per item); Format: enabled (pretty-print); Operation: to JSON | — | Filter Changed/New Workflows | Encode as Base64 | Type version 1.1 | Very large workflows (>10 MB serialized) may hit n8n memory limits. |
| Encode as Base64 | Extract from File | Converts binary data to a `data` property containing a Base64 string | Operation: binary to property; no extra options | — | Convert Workflow to JSON File | Build Filename & Commit Date | Type version 1 | Output property is `data` (Base64 string), referenced by downstream GitHub nodes. |

---

#### Block 5 — Commit Changed/New Files to GitHub

**Overview:**  
For each workflow to be committed, a Set node builds the target filename and a formatted commit timestamp. An IF node then checks whether that filename already exists in the GitHub repo. The true branch updates the existing file; the false branch creates a new one.

**Nodes Involved:**  
- Build Filename & Commit Date
- File Already Exists in Repo?
- Create New File
- Update Existing File

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Version / Requirements | Edge Cases & Failure Types |
|------|------|---------------|---------------|----------------------------|-------|--------|------------------------|---------------------------|
| Build Filename & Commit Date | Set | Enriches each item with `fileName` and `commitDate` | Assignment 1 — `commitDate`: `{{ $now.format('dd-MM-yyyy/H:mm') }}`; Assignment 2 — `fileName`: `{{ $('Fetch All n8n Workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}.json` | `commitDate`, `fileName` | Encode as Base64 | File Already Exists in Repo? | Type version 3.4 | `$now` uses the n8n execution timezone; filename derivation must match the same logic used in the Filter node. |
| File Already Exists in Repo? | IF | Routes to Update vs. Create | String condition: `value1` = `{{ $('Collect GitHub Filenames').item.json.name }}` **contains** `value2` = `{{ $('Fetch All n8n Workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}.json` | — | Build Filename & Commit Date | (true) Update Existing File; (false) Create New File | Type version 1 | The "contains" check could match a substring erroneously (e.g., `my-workflow.json` would contain `workflow.json`); however, since `Collect GitHub Filenames` aggregates names into one array, the expression is evaluated per-item. **Potential issue:** referencing `.item` on an Aggregate output may not iterate correctly — verify that `Collect GitHub Filenames` returns one item with an array field `name`, not multiple items each with a scalar `name`. If the latter, the contains check works per-item; if the former, only the first element of the array is checked. |
| Create New File | GitHub | File → Create (upload) | Owner: `YOUR_GITHUB_USERNAME`; Repository: `n8n_backup`; filePath: `{{ $('Fetch All n8n Workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}.json`; fileContent: `{{ $('Encode as Base64').item.json.data }}`; commitMessage: `=Sync-{{ $('Build Filename & Commit Date').item.json.commitDate }}` | — | File Already Exists in Repo? (false) | — (no downstream; execution ends here for new files) | Type version 1; same GitHub credential | GitHub API may reject if filePath conflicts with an existing file (race condition if another process commits between List and Create). |
| Update Existing File | GitHub | File → Edit | Same owner/repo/filePath/fileContent as Create; operation: `edit`; commitMessage: `=Sync-{{ $('Build Filename & Commit Date').item.json.commitDate }}` | — | File Already Exists in Repo? (true) | Find Orphaned Files to Delete | Type version 1; same GitHub credential | The `edit` operation requires the file's SHA; n8n's GitHub node auto-fetches it — will fail if the file was deleted between List and Edit. |

**Important note on the "contains" check in the IF node:**  
The IF node uses a string "contains" condition between the aggregated GitHub filenames array and the derived workflow filename. Because `Collect GitHub Filenames` aggregates all `name` values into a single item's array field, the expression `$('Collect GitHub Filenames').item.json.name` returns the **array** itself. The "contains" operation on an array vs. a string may not behave as expected in n8n's expression engine. This could cause all workflows to take the "Create" path even when they should "Update." Testing is strongly recommended.

---

#### Block 6 — Cleanup Orphaned Files from GitHub

**Overview:**  
After successful commits, a Code node compares the list of GitHub files against current n8n workflow names. Any GitHub file that has no matching workflow is identified for deletion. A safety guard aborts the deletion if n8n returned zero workflows (e.g., API failure), preventing accidental repo wipe. An IF node then filters out "skipped" items, and a GitHub node deletes each orphaned file.

**Nodes Involved:**  
- Find Orphaned Files to Delete
- Any Files to Delete?
- Delete Orphaned File

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Version / Requirements | Edge Cases & Failure Types |
|------|------|---------------|---------------|----------------------------|-------|--------|------------------------|---------------------------|
| Find Orphaned Files to Delete | Code (JavaScript) | Computes set difference: GitHub files minus n8n workflow files | Inline JS (see below) | References `$('List Files in GitHub Repo').all()` and `$('Fetch All n8n Workflows').all()`; output items have `path` and `skipped` | Update Existing File | Any Files to Delete? | Type version 2 | Throws an error with message "Safety abort: n8n returned 0 workflows" if n8n list is empty — this halts the entire execution. |
| Any Files to Delete? | IF | Routes non-skipped items to the Delete node | Boolean condition: `value1` = `{{ $json.skipped }}` (checks if `skipped` is truthy) | — | Find Orphaned Files to Delete | (output 0 / true) → Delete Orphaned File | Type version 1 | **Potential logic concern:** The condition evaluates `$json.skipped`. When `skipped: true` (nothing to delete), the condition is met and items flow to output 0 → Delete Orphaned File. When `skipped: false` (actual files to delete), the condition is NOT met and items flow to output 1, which has **no connection**, meaning files that should be deleted are silently dropped. This appears to be inverted logic. To fix, either (a) swap the outputs so that the false branch connects to Delete Orphaned File, or (b) invert the condition to check `{{ $json.skipped === false }}`. |
| Delete Orphaned File | GitHub | File → Delete | Owner: `YOUR_GITHUB_USERNAME`; Repository: `n8n_backup`; filePath: `{{ $json.path }}`; commitMessage: `=chore: remove deleted workflow {{ $json.path }}` | — | Any Files to Delete? (output 0) | — (end of pipeline) | Type version 1.1; same GitHub credential | If `$json.path` is undefined (skipped item), the delete call will fail with a 404 or validation error. |

**Key logic inside Find Orphaned Files to Delete:**
1. Collects all GitHub file names from `$('List Files in GitHub Repo').all()`.
2. Builds an array of expected n8n filenames from `$('Fetch All n8n Workflows').all()` (same hyphen-lowercase logic).
3. **Safety check:** if `n8nWorkflowFiles.length === 0`, throws an error to abort.
4. If GitHub repo is empty, returns `{ skipped: true, reason: 'GitHub repo is empty — nothing to delete.' }`.
5. Computes `filesToDelete` = GitHub files not present in n8n list.
6. If none, returns `{ skipped: true, reason: 'All GitHub files match existing workflows. Nothing to delete.' }`.
7. Otherwise returns `{ path: fileName, skipped: false }` for each orphaned file.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Schedule Trigger | Schedule Trigger | Starts the backup pipeline on an hourly interval | — | List Files in GitHub Repo | # 🗂️ n8n → GitHub sync Workflow versioning · What this does: Automatically backs up all your n8n workflows as JSON files to a GitHub repository on a schedule. · Features: ✅ Creates new files for new workflows ✅ Updates existing files when workflows change ✅ Deletes files for workflows that were renamed or removed ✅ Skips unchanged workflows (size-based diff check) ✅ Timestamped commit messages · Setup (one-time): 1. GitHub credential — create a Personal Access Token with `repo` scope and add it as a GitHub credential in n8n 2. n8n API credential — generate an n8n API key under Settings → API and add it as an n8n credential 3. GitHub repo — create an empty repository on GitHub (e.g. `your-username/n8n-backup`) 4. Update all GitHub nodes: set your owner (GitHub username) and repository name 5. Adjust the Schedule Trigger to your preferred backup interval (default: every hour) · Notes: Workflow files are named using the workflow name, lowercased and hyphenated (e.g. `my-workflow.json`). The repo at the top of the canvas links to a working example. Contact us if you have any suggestions or need support for this workflow: support@ucartz.com |
| List Files in GitHub Repo | GitHub | Lists all files in the backup repository | Schedule Trigger | Collect GitHub Filenames | ### 📥 Step 1 — Fetch current GitHub repo contents · Lists all files already in the GitHub backup repository. This is used later to: Detect which workflows are new vs existing · Identify files to delete (renamed/removed workflows) · ## 🔧 Configuration Required — GitHub Username · Search for all GitHub nodes in this workflow (List Files in GitHub Repo, Create New File, Update Existing File, Delete Orphaned File) and replace the owner field value YOUR_GITHUB_USERNAME with your actual GitHub username. The same username should also be reflected in the repository URL of each node. |
| Collect GitHub Filenames | Aggregate | Aggregates all file names into a single item | List Files in GitHub Repo | Fetch All n8n Workflows | ### 📥 Step 1 — Fetch current GitHub repo contents · Lists all files already in the GitHub backup repository. This is used later to: Detect which workflows are new vs existing · Identify files to delete (renamed/removed workflows) |
| Fetch All n8n Workflows | n8n | Retrieves all workflows from the local n8n instance | Collect GitHub Filenames | Filter Changed/New Workflows | ### ⚙️ Step 2 — Fetch & filter n8n workflows · Retrieves all workflows from this n8n instance via API. The Filter Changed Files node compares each workflow's byte size against the file size stored on GitHub. Only workflows that are new or modified pass through — skipping unchanged ones avoids unnecessary GitHub commits. |
| Filter Changed/New Workflows | Code | Size-based diff to emit only new or changed workflows | Fetch All n8n Workflows | Convert Workflow to JSON File | ### ⚙️ Step 2 — Fetch & filter n8n workflows · Retrieves all workflows from this n8n instance via API. The Filter Changed Files node compares each workflow's byte size against the file size stored on GitHub. Only workflows that are new or modified pass through — skipping unchanged ones avoids unnecessary GitHub commits. |
| Convert Workflow to JSON File | Convert to File | Serializes each workflow to a pretty-printed JSON binary | Filter Changed/New Workflows | Encode as Base64 | |
| Encode as Base64 | Extract from File | Base64-encodes the binary JSON for GitHub API upload | Convert Workflow to JSON File | Build Filename & Commit Date | |
| Build Filename & Commit Date | Set | Adds `fileName` and `commitDate` fields to each item | Encode as Base64 | File Already Exists in Repo? | ### 💾 Step 3 — Commit files to GitHub · For each changed workflow: 1. Build the target filename and commit timestamp 2. Check if the file already exists in the repo 3. Update it (edit) if it exists, or Create it (upload) if it's new · Files are named: `workflow-name.json` (lowercase, hyphenated) |
| File Already Exists in Repo? | IF | Routes items to Update (true) or Create (false) | Build Filename & Commit Date | (true) Update Existing File; (false) Create New File | ### 💾 Step 3 — Commit files to GitHub · For each changed workflow: 1. Build the target filename and commit timestamp 2. Check if the file already exists in the repo 3. Update it (edit) if it exists, or Create it (upload) if it's new · Files are named: `workflow-name.json` (lowercase, hyphenated) |
| Create New File | GitHub | Uploads a new workflow JSON file to GitHub | File Already Exists in Repo? (false) | — | ### 💾 Step 3 — Commit files to GitHub · For each changed workflow: 1. Build the target filename and commit timestamp 2. Check if the file already exists in the repo 3. Update it (edit) if it exists, or Create it (upload) if it's new · Files are named: `workflow-name.json` (lowercase, hyphenated) · ## 🔧 Configuration Required — GitHub Username · Search for all GitHub nodes in this workflow (List Files in GitHub Repo, Create New File, Update Existing File, Delete Orphaned File) and replace the owner field value YOUR_GITHUB_USERNAME with your actual GitHub username. The same username should also be reflected in the repository URL of each node. |
| Update Existing File | GitHub | Edits an existing workflow JSON file in GitHub | File Already Exists in Repo? (true) | Find Orphaned Files to Delete | ### 💾 Step 3 — Commit files to GitHub · For each changed workflow: 1. Build the target filename and commit timestamp 2. Check if the file already exists in the repo 3. Update it (edit) if it exists, or Create it (upload) if it's new · Files are named: `workflow-name.json` (lowercase, hyphenated) · ## 🔧 Configuration Required — GitHub Username · Search for all GitHub nodes in this workflow (List Files in GitHub Repo, Create New File, Update Existing File, Delete Orphaned File) and replace the owner field value YOUR_GITHUB_USERNAME with your actual GitHub username. The same username should also be reflected in the repository URL of each node. |
| Find Orphaned Files to Delete | Code | Identifies GitHub files with no matching n8n workflow | Update Existing File | Any Files to Delete? | ### 🗑️ Step 4 — Clean up deleted/renamed workflows · After committing, compare: GitHub files (what's in the repo) and n8n workflows (what currently exists). Any GitHub file with no matching n8n workflow means the workflow was renamed or deleted — those files are removed from the repo automatically. ⚠️ Safety: if n8n returns 0 workflows (e.g. API error), the deletion step is aborted to prevent wiping the entire repo. |
| Any Files to Delete? | IF | Filters out "skipped" items before deletion | Find Orphaned Files to Delete | (output 0) Delete Orphaned File | ### 🗑️ Step 4 — Clean up deleted/renamed workflows · After committing, compare: GitHub files (what's in the repo) and n8n workflows (what currently exists). Any GitHub file with no matching n8n workflow means the workflow was renamed or deleted — those files are removed from the repo automatically. ⚠️ Safety: if n8n returns 0 workflows (e.g. API error), the deletion step is aborted to prevent wiping the entire repo. |
| Delete Orphaned File | GitHub | Removes an orphaned file from the GitHub repo | Any Files to Delete? (output 0) | — | ### 🗑️ Step 4 — Clean up deleted/renamed workflows · After committing, compare: GitHub files (what's in the repo) and n8n workflows (what currently exists). Any GitHub file with no matching n8n workflow means the workflow was renamed or deleted — those files are removed from the repo automatically. ⚠️ Safety: if n8n returns 0 workflows (e.g. API error), the deletion step is aborted to prevent wiping the entire repo. · ## 🔧 Configuration Required — GitHub Username · Search for all GitHub nodes in this workflow (List Files in GitHub Repo, Create New File, Update Existing File, Delete Orphaned File) and replace the owner field value YOUR_GITHUB_USERNAME with your actual GitHub username. The same username should also be reflected in the repository URL of each node. |
| README | Sticky Note | Documentation: features, setup instructions, contact | — | — | # 🗂️ n8n → GitHub sync Workflow versioning · What this does: Automatically backs up all your n8n workflows as JSON files to a GitHub repository on a schedule. · Features: ✅ Creates new files for new workflows ✅ Updates existing files when workflows change ✅ Deletes files for workflows that were renamed or removed ✅ Skips unchanged workflows (size-based diff check) ✅ Timestamped commit messages · Setup (one-time): 1. GitHub credential — create a Personal Access Token with `repo` scope and add it as a GitHub credential in n8n 2. n8n API credential — generate an n8n API key under Settings → API and add it as an n8n credential 3. GitHub repo — create an empty repository on GitHub (e.g. `your-username/n8n-backup`) 4. Update all GitHub nodes: set your owner (GitHub username) and repository name 5. Adjust the Schedule Trigger to your preferred backup interval (default: every hour) · Notes: Workflow files are named using the workflow name, lowercased and hyphenated (e.g. `my-workflow.json`). The repo at the top of the canvas links to a working example. Contact us if you have any suggestions or need support for this workflow: support@ucartz.com |
| Sticky Note — Fetch GitHub Files | Sticky Note | Describes Block 2 logic | — | — | ### 📥 Step 1 — Fetch current GitHub repo contents · Lists all files already in the GitHub backup repository. This is used later to: Detect which workflows are new vs existing · Identify files to delete (renamed/removed workflows) |
| Sticky Note — Fetch & Filter Workflows | Sticky Note | Describes Block 3 logic | — | — | ### ⚙️ Step 2 — Fetch & filter n8n workflows · Retrieves all workflows from this n8n instance via API. The Filter Changed Files node compares each workflow's byte size against the file size stored on GitHub. Only workflows that are new or modified pass through — skipping unchanged ones avoids unnecessary GitHub commits. |
| Sticky Note — Commit to GitHub | Sticky Note | Describes Block 5 logic | — | — | ### 💾 Step 3 — Commit files to GitHub · For each changed workflow: 1. Build the target filename and commit timestamp 2. Check if the file already exists in the repo 3. Update it (edit) if it exists, or Create it (upload) if it's new · Files are named: `workflow-name.json` (lowercase, hyphenated) |
| Sticky Note — Cleanup Deleted Workflows | Sticky Note | Describes Block 6 logic | — | — | ### 🗑️ Step 4 — Clean up deleted/renamed workflows · After committing, compare: GitHub files (what's in the repo) and n8n workflows (what currently exists). Any GitHub file with no matching n8n workflow means the workflow was renamed or deleted — those files are removed from the repo automatically. ⚠️ Safety: if n8n returns 0 workflows (e.g. API error), the deletion step is aborted to prevent wiping the entire repo. |
| Sticky Note | Sticky Note | Configuration reminder for GitHub username replacement | — | — | ## 🔧 Configuration Required — GitHub Username · Search for all GitHub nodes in this workflow (List Files in GitHub Repo, Create New File, Update Existing File, Delete Orphaned File) and replace the owner field value YOUR_GITHUB_USERNAME with your actual GitHub username. The same username should also be reflected in the repository URL of each node. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the entire workflow manually in n8n.

---

**1. Create the Schedule Trigger**
- Add a **Schedule Trigger** node.
- Configure: Interval-based, every 1 hour (add one empty interval entry).
- This is the sole entry point — no input connections.

**2. Create the "List Files in GitHub Repo" node**
- Add a **GitHub** node.
- Operation: **File → List**.
- Owner: set to your GitHub username (e.g., `johndoe`).
- Repository: select or type your backup repository name (e.g., `n8n_backup`).
- Enable **Always Output Data** (toggled on).
- Credential: create or select a GitHub API credential using a Personal Access Token with `repo` scope.
- Connect: Schedule Trigger → List Files in GitHub Repo.

**3. Create the "Collect GitHub Filenames" node**
- Add an **Aggregate** node.
- Field to aggregate: `name`.
- Options: default (no special options).
- Connect: List Files in GitHub Repo → Collect GitHub Filenames.

**4. Create the "Fetch All n8n Workflows" node**
- Add an **n8n** node.
- Operation: **Get All** (default; no filters, no request options).
- Credential: create or select an n8n API credential using an API key generated under *Settings → API* in your n8n instance.
- Connect: Collect GitHub Filenames → Fetch All n8n Workflows.

**5. Create the "Filter Changed/New Workflows" Code node**
- Add a **Code** node (JavaScript).
- Mode: **Run Once for All Items** (default for type version 2).
- Paste the following logic (reproduced from the original):
  - Build a `githubFileMap` from `$('List Files in GitHub Repo').all()` mapping `name → size`.
  - For each input item, stringify the workflow JSON with 2-space indentation.
  - Derive the filename: `item.json.name.replace(/\s+/g, '-').toLowerCase() + '.json'`.
  - Compute `currentSize` using `Buffer.byteLength(workflowJson, 'utf8')`.
  - Determine `isNew` (file not in map) and `changed` (new or size differs).
  - Attach `_fileName`, `_currentSize`, `_githubSize`, `_changed`, `_isNew` to each item.
  - Return only items where `_changed` or `_isNew` is true.
- Connect: Fetch All n8n Workflows → Filter Changed/New Workflows.

**6. Create the "Convert Workflow to JSON File" node**
- Add a **Convert to File** node.
- Mode: **Each** (one file per item).
- Operation: **To JSON**.
- Options: **Format** enabled (pretty-print).
- Connect: Filter Changed/New Workflows → Convert Workflow to JSON File.

**7. Create the "Encode as Base64" node**
- Add an **Extract from File** node.
- Operation: **Binary to Property**.
- Options: default.
- Connect: Convert Workflow to JSON File → Encode as Base64.

**8. Create the "Build Filename & Commit Date" Set node**
- Add a **Set** node (type version 3.x).
- Add two assignments:
  - `commitDate` (string): `{{ $now.format('dd-MM-yyyy/H:mm') }}`
  - `fileName` (string): `{{ $('Fetch All n8n Workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}.json`
- Connect: Encode as Base64 → Build Filename & Commit Date.

**9. Create the "File Already Exists in Repo?" IF node**
- Add an **IF** node (type version 1).
- Condition type: **String**.
- Operation: **Contains**.
- Value 1: `{{ $('Collect GitHub Filenames').item.json.name }}`
- Value 2: `{{ $('Fetch All n8n Workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}.json`
- Connect: Build Filename & Commit Date → File Already Exists in Repo?
- Output 0 (true) → will connect to Update Existing File.
- Output 1 (false) → will connect to Create New File.

> **Note on the "contains" check:** Because `Collect GitHub Filenames` aggregates all names into an array on a single item, `$('Collect GitHub Filenames').item.json.name` returns that array. The "contains" string operation on an array may not match as intended. An alternative approach is to use a Code node that checks `Array.includes()`, or restructure the Aggregate to output individual items.

**10. Create the "Create New File" GitHub node**
- Add a **GitHub** node.
- Operation: **File → Create**.
- Owner: your GitHub username.
- Repository: `n8n_backup`.
- File path: `{{ $('Fetch All n8n Workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}.json`
- File content: `{{ $('Encode as Base64').item.json.data }}`
- Commit message: `=Sync-{{ $('Build Filename & Commit Date').item.json.commitDate }}`
- Credential: same GitHub credential as step 2.
- Connect: File Already Exists in Repo? output 1 (false) → Create New File.

**11. Create the "Update Existing File" GitHub node**
- Add a **GitHub** node.
- Operation: **File → Edit**.
- Owner: your GitHub username.
- Repository: `n8n_backup`.
- File path: same expression as Create New File.
- File content: same expression as Create New File.
- Commit message: same expression as Create New File.
- Credential: same GitHub credential.
- Connect: File Already Exists in Repo? output 0 (true) → Update Existing File.

**12. Create the "Find Orphaned Files to Delete" Code node**
- Add a **Code** node (JavaScript).
- Paste the following logic:
  - Collect `githubFiles` (names) from `$('List Files in GitHub Repo').all()`.
  - Build `n8nWorkflowFiles` from `$('Fetch All n8n Workflows').all()` using the same hyphen-lowercase + `.json` naming.
  - If `n8nWorkflowFiles.length === 0`, throw `Error('Safety abort: n8n returned 0 workflows. Refusing to delete any GitHub files.')`.
  - If `githubFiles.length === 0`, return `{ skipped: true, reason: 'GitHub repo is empty — nothing to delete.' }`.
  - Compute `filesToDelete = githubFiles.filter(f => !n8nWorkflowFiles.includes(f))`.
  - If empty, return `{ skipped: true, reason: 'All GitHub files match existing workflows. Nothing to delete.' }`.
  - Otherwise return `{ path: f, skipped: false }` for each file to delete.
- Connect: Update Existing File → Find Orphaned Files to Delete.

**13. Create the "Any Files to Delete?" IF node**
- Add an **IF** node (type version 1).
- Condition type: **Boolean**.
- Value 1: `{{ $json.skipped }}`
- No value 2 (implicit comparison to `true`).
- **Important correction:** The original workflow connects output 0 (condition = true, i.e., `skipped` is truthy) to the Delete node. This is inverted. To fix:
  - Either change the condition to `{{ $json.skipped === false }}` so that output 0 means "not skipped → delete", **or**
  - Keep the condition as-is but connect output 1 (false branch) to the Delete node instead of output 0.
- Connect per the corrected logic: the branch representing "files need deletion" → Delete Orphaned File.

**14. Create the "Delete Orphaned File" GitHub node**
- Add a **GitHub** node (type version 1.1).
- Operation: **File → Delete**.
- Owner: your GitHub username.
- Repository: `n8n_backup`.
- File path: `{{ $json.path }}`
- Commit message: `=chore: remove deleted workflow {{ $json.path }}`
- Credential: same GitHub credential.
- Connect: Any Files to Delete? (corrected branch) → Delete Orphaned File.

**15. Add documentation sticky notes (optional but recommended)**
- Add a large **Sticky Note** titled "README" with the setup instructions, feature list, and contact email `support@ucartz.com`.
- Add sticky notes for each logical step as visual documentation (Fetch GitHub Files, Fetch & Filter Workflows, Commit to GitHub, Cleanup Deleted Workflows, Configuration Required).

**16. Credential setup summary**
| Credential | How to Create |
|-----------|---------------|
| GitHub API | Go to GitHub → Settings → Developer settings → Personal access tokens → Generate new token with `repo` scope. In n8n, add under Credentials → GitHub API, paste the token. |
| n8n API | In your n8n instance, go to Settings → API → Create API Key. In n8n, add under Credentials → n8n API, enter the base URL of your instance and the API key. |

**17. Activate the workflow**
- Toggle the workflow to **Active**.
- Verify the first execution runs successfully by checking the GitHub repository for newly created/updated `.json` files.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| Workflow files are named using the workflow name, lowercased and hyphenated (e.g., `my-workflow.json`). This convention is applied consistently across all nodes — any change to the naming scheme must be updated in the Filter node, the Set node, both GitHub commit nodes, and the orphan-detection Code node. | Naming convention |
| The size-based diff check is an optimization, not a guarantee of accuracy. Two different workflow JSONs could theoretically produce the same byte length. For critical environments, consider a content-hash (SHA-256) comparison instead. | Diff strategy limitation |
| The "Any Files to Delete?" IF node appears to have inverted logic (the true/skipped branch connects to the Delete action). This should be corrected before relying on automatic orphan cleanup. See step 13 in the reproduction guide. | Potential bug — IF node logic |
| The "File Already Exists in Repo?" IF node uses a string "contains" condition between an aggregated array and a filename string. This may not behave as expected because `$('Collect GitHub Filenames').item.json.name` returns an array. Test this node carefully; an alternative is a Code node using `Array.includes()`. | Potential issue — contains check on array |
| If the n8n API returns zero workflows (due to an error), the orphan detection Code node throws a safety abort error, which stops the entire execution. No GitHub files will be deleted in that case. | Safety mechanism |
| GitHub API rate limits apply: 5,000 requests per hour for authenticated users. With large numbers of workflows, the per-item GitHub operations (create/update/delete) could approach this limit. Consider batching or throttling if you have hundreds of workflows. | Rate limiting |
| Contact for support or suggestions: support@ucartz.com | Contact |
| This workflow was created by ucartz.com | Project credit |