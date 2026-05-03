Automated Workflow Backup with Intelligent Change Detection using GitHub

https://n8nworkflows.xyz/workflows/automated-workflow-backup-with-intelligent-change-detection-using-github-7304


# Automated Workflow Backup with Intelligent Change Detection using GitHub

### 1. Workflow Overview

This workflow automates the backup of n8n workflows to a GitHub repository with intelligent change detection. It is designed for users who want to maintain a clean, versioned backup of their n8n workflows in GitHub, minimizing unnecessary commits and handling renames seamlessly.

**Use Cases:**
- Automated, scheduled backup of n8n workflows to GitHub.
- Detection and management of workflow updates and renames.
- Optional notification via Telegram about backup activity.

**Logical Blocks:**

1.1 **Configuration and Scheduling**  
- Setup static parameters and schedule the workflow trigger.

1.2 **Data Collection**  
- Fetch all workflows from n8n and list existing workflow files on GitHub.

1.3 **Data Preparation and Merging**  
- Encode n8n workflows for comparison, extract GitHub file details, and merge datasets.

1.4 **Change Detection and Decision Making**  
- Compare encoded workflows and GitHub files to detect creates, updates, renames, or skips.

1.5 **Action Execution**  
- Branch logic to perform GitHub file creation, updates, renames (delete + create), or skip.

1.6 **Reporting**  
- Aggregate changes, build a summary message, and optionally send it via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Configuration and Scheduling

**Overview:**  
This block sets up essential parameters for GitHub repo, backup folder, and optional Telegram reporting. It also triggers the workflow on a scheduled interval.

**Nodes Involved:**  
- Configuration (Set)  
- Assert GitHub config (If)  
- Stop on empty config (StopAndError)  
- Schedule Trigger (ScheduleTrigger)

**Node Details:**

- **Configuration**  
  - Type: Set  
  - Role: Holds GitHub repo owner, repo name, repo path for backups, Telegram chat ID, and verbosity flag.  
  - Key fields:  
    - `repo.owner` (string)  
    - `repo.name` (string)  
    - `repo.path` (string, default "workflows/")  
    - `report.tg.chatID` (number or null)  
    - `report.verbose` (boolean)  
  - Output: Passes configuration to downstream nodes.

- **Assert GitHub config**  
  - Type: If  
  - Role: Validates that all required GitHub config parameters are set and non-empty.  
  - Conditions: `repo.owner`, `repo.name`, and `repo.path` must be non-empty strings.  
  - Outputs:  
    - True: Continue workflow  
    - False: Stop workflow with error.

- **Stop on empty config**  
  - Type: StopAndError  
  - Role: Stops workflow with error message if GitHub config is incomplete.  
  - Error message: "Incomplete GitHub configuration. Please check \"Configuration\" node."

- **Schedule Trigger**  
  - Type: ScheduleTrigger  
  - Role: Starts the workflow on a schedule (default every hour).  
  - Config: Interval set to hourly.  
  - Output: Triggers the workflow steps downstream.

**Edge Cases & Failures:**  
- Missing config parameters stop workflow immediately.  
- Schedule misconfiguration may delay or prevent triggering.

---

#### 1.2 Data Collection

**Overview:**  
Fetches all workflows from the n8n instance and gets the list of workflow files from the specified GitHub repo folder.

**Nodes Involved:**  
- Get all workflows (n8n API)  
- List files (GitHub)  
- Get files (GitHub)  
- Extract workflow parameters (Code)

**Node Details:**

- **Get all workflows**  
  - Type: n8n API  
  - Role: Retrieves all workflows from the connected n8n instance via API.  
  - Credentials: n8n API credentials.  
  - Output: Full workflow JSON objects.

- **List files**  
  - Type: GitHub  
  - Role: Lists all files under the configured GitHub repo folder (`repo.path`).  
  - Operation: List files in folder.  
  - Credentials: GitHub API credentials.  
  - Error handling: Continues even if folder doesn’t exist or is empty (edge case).

- **Get files**  
  - Type: GitHub  
  - Role: Retrieves file content and metadata for each file listed by "List files".  
  - Operation: Get file content.  
  - Credentials: GitHub API.  
  - Error handling: Continues on errors (e.g., missing files).

- **Extract workflow parameters**  
  - Type: Code  
  - Role: Decodes GitHub file content from base64, parses JSON, extracts workflow `id`, `name`, and file path, and stores base64 content for comparison.  
  - Handles errors in parsing gracefully by logging and marking the item with error info.

**Edge Cases & Failures:**  
- Non-JSON or corrupted files on GitHub are handled without stopping workflow.  
- Missing GitHub folder results in empty list, handled gracefully.

---

#### 1.3 Data Preparation and Merging

**Overview:**  
Encodes n8n workflows to base64 and merges the GitHub and n8n datasets into a unified structure for comparison.

**Nodes Involved:**  
- Encode N8N workflows (Code)  
- Merge (Merge node)

**Node Details:**

- **Encode N8N workflows**  
  - Type: Code  
  - Role: Converts each n8n workflow JSON to a base64 string to avoid data pollution and ease comparison.  
  - Output fields: `id`, `name`, and encoded `n8nWorkflowData`.

- **Merge**  
  - Type: Merge  
  - Role: Enriches the encoded n8n workflows with corresponding GitHub workflow data items matched by `id`.  
  - Mode: Combine with enrichInput1 (keeps original n8n data and adds GitHub data).

**Edge Cases & Failures:**  
- Mismatched workflows (existing only in one source) remain in combined output with missing counterparts.

---

#### 1.4 Change Detection and Decision Making

**Overview:**  
Determines if each workflow needs to be created, updated, renamed, or skipped based on a stable comparison of JSON content and filenames.

**Nodes Involved:**  
- Decide changes (Code)  
- Router (Switch)

**Node Details:**

- **Decide changes**  
  - Type: Code  
  - Role:  
    - Performs deep sorting of JSON keys for stable serialization.  
    - Compares base64-decoded n8n workflow and GitHub file content JSON for differences.  
    - Detects if file exists on GitHub, if the workflow name changed (rename), or if content changed (update).  
    - Sets operation context per item: `create`, `update`, `rename`, or `skip`.  
  - Outputs: Enriched items with context and flags.

- **Router**  
  - Type: Switch  
  - Role: Routes each item according to the determined operation context (`rename`, `update`, `create`, `skip`).  
  - Fallback: Sends unexpected operations to `Stop and Error`.

**Edge Cases & Failures:**  
- Parsing errors in JSON cause forced commit to avoid missing changes.  
- Unknown operation values cause workflow to stop with error message.

---

#### 1.5 Action Execution

**Overview:**  
Executes GitHub API operations based on the decision: create, update, rename (delete + create), or skip workflows.

**Nodes Involved:**  
- Update file content and commit (GitHub)  
- Delete old file (GitHub)  
- Create new file (GitHub)  
- Create new file (rename) (GitHub)  
- Merge after update, create, create(rename), delete(rename) (Merge nodes)  
- No Operation, do nothing (NoOp)  
- Stop and Error (StopAndError)

**Node Details:**

- **Update file content and commit**  
  - Type: GitHub  
  - Role: Edits existing file content on GitHub with updated workflow JSON.  
  - Commit message: `"update: <workflow name>"`  
  - Input: Uses `newFile.path` and decoded base64 workflow JSON from n8n side.

- **Delete old file**  
  - Type: GitHub  
  - Role: Deletes the old file on GitHub to handle renames (step 1).  
  - Commit message: `"rename: oldName -> newName (step 1/2: remove old)"`

- **Create new file (rename)**  
  - Type: GitHub  
  - Role: Creates a new file on GitHub with the new name (rename step 2).  
  - Commit message: `"rename: oldName -> newName (step 2/2: create new)"`

- **Create new file**  
  - Type: GitHub  
  - Role: Creates a new file for new workflows not present on GitHub.  
  - Commit message: `"create: <workflow name>"`

- **Merge nodes after GitHub operations**  
  - Type: Merge  
  - Role: Combine GitHub response data with original context to preserve flags and operation info for reporting and further processing.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Passes through items for workflows where no change is detected (skip operation).

- **Stop and Error**  
  - Type: StopAndError  
  - Role: Stops workflow for unsupported or unexpected operations with detailed error message.

**Edge Cases & Failures:**  
- GitHub API errors (auth, rate limits, file not found) are possible failure points.  
- Rename implemented as two-step delete and create to maintain history and clarity.  
- Merge nodes preserve context to avoid losing metadata after GitHub node execution.

---

#### 1.6 Reporting

**Overview:**  
Aggregates all workflow operations into categorized lists, formats a MarkdownV2 summary message, and optionally sends it to Telegram.

**Nodes Involved:**  
- Build summary arrays (Code)  
- Anything changed? (If)  
- Render summary (Code)  
- Is Telegram configured? (If)  
- Send a message (Telegram)

**Node Details:**

- **Build summary arrays**  
  - Type: Code  
  - Role: Aggregates workflow operation results into arrays categorized by `create`, `update`, `rename`, and `skip`.  
  - Outputs flags indicating if any changes occurred.

- **Anything changed?**  
  - Type: If  
  - Role: Decides if a summary should be generated and sent based on verbosity config or any detected changes.

- **Render summary**  
  - Type: Code  
  - Role:  
    - Builds a detailed MarkdownV2 formatted message summarizing the backup operation.  
    - Escapes special characters for Telegram compatibility.  
    - Includes clickable GitHub repo links and categorized workflow lists with counts.

- **Is Telegram configured?**  
  - Type: If  
  - Role: Checks if Telegram chat ID is set and valid for sending the message.

- **Send a message**  
  - Type: Telegram  
  - Role: Sends the summary message to the configured Telegram chat using MarkdownV2 formatting.  
  - Credentials: Telegram API credentials.

**Edge Cases & Failures:**  
- Telegram not configured or chat ID missing disables message sending.  
- Markdown escaping avoids message formatting errors.  
- Network or API issues might cause message sending failure.

---

### 3. Summary Table

| Node Name                    | Node Type                | Functional Role                          | Input Node(s)                        | Output Node(s)                        | Sticky Note                                                        |
|------------------------------|--------------------------|----------------------------------------|------------------------------------|-------------------------------------|-------------------------------------------------------------------|
| Configuration                | Set                      | Holds repo and report parameters       | Schedule Trigger                   | Assert GitHub config                 |                                                                   |
| Assert GitHub config         | If                       | Validates GitHub config parameters     | Configuration                     | Get all workflows, List files, Stop on empty config | Pre-provisioning safe fuse                                         |
| Stop on empty config         | StopAndError             | Stops if config incomplete              | Assert GitHub config (false path) |                                     |                                                                   |
| Schedule Trigger             | ScheduleTrigger          | Initiates workflow on schedule          |                                    | Configuration                      | Tune the schedule (default hourly)                                |
| Get all workflows            | n8n API                  | Fetches all n8n workflows               | Assert GitHub config (true path)  | Encode N8N workflows               | Collect data - N8N workflows list                                 |
| List files                  | GitHub                   | Lists files in GitHub repo folder       | Assert GitHub config (true path)  | Get files                         | Collect data - GitHub files list                                  |
| Get files                   | GitHub                   | Retrieves file content from GitHub      | List files                       | Extract workflow parameters         | Edge case handling, do not stop on missing folder                 |
| Extract workflow parameters  | Code                     | Parses GitHub file content for comparison | Get files                      | Merge                            |                                                                   |
| Encode N8N workflows         | Code                     | Encodes n8n workflows to base64         | Get all workflows                | Merge                            |                                                                   |
| Merge                       | Merge                    | Combines encoded n8n workflows and GitHub workflows | Encode N8N workflows, Extract workflow parameters | Decide changes                  |                                                                   |
| Decide changes              | Code                     | Detects create, update, rename, or skip | Loop Over Items (after merge)    | Router                          |                                                                   |
| Loop Over Items             | SplitInBatches           | Iterates over workflows for processing  | Merge after create/update/delete/skip | Is Telegram configured, Decide changes |                                                                   |
| Router                      | Switch                   | Routes workflows by decided operation   | Decide changes                   | Delete old file, Update file content, Create new file, No Operation, Stop and Error | Controller                                                        |
| Delete old file             | GitHub                   | Deletes old file on rename               | Router (rename path)             | Merge after delete (rename)          | Rename a file (two-step)                                          |
| Merge after delete (rename) | Merge                    | Merges context after delete step         | Delete old file                  | Create new file (rename)             |                                                                   |
| Create new file (rename)    | GitHub                   | Creates new file on rename step          | Merge after delete (rename)      | Merge after create (rename)          |                                                                   |
| Merge after create (rename) | Merge                    | Merges context after create (rename)     | Create new file (rename)         | Loop Over Items                     |                                                                   |
| Update file content and commit | GitHub                | Updates existing file content             | Router (update path)             | Merge after update                  | Update an existing file                                           |
| Merge after update          | Merge                    | Merges context after update               | Update file content and commit   | Loop Over Items                     |                                                                   |
| Create new file             | GitHub                   | Creates new file for new workflows        | Router (create path)             | Merge after create                  | Create a new file                                                |
| Merge after create          | Merge                    | Merges context after create                | Create new file                 | Loop Over Items                     |                                                                   |
| No Operation, do nothing    | NoOp                     | Passes through workflows with no changes | Router (skip path)              | Loop Over Items                     | Nothing to do                                                   |
| Stop and Error             | StopAndError             | Stops workflow for unexpected operations | Router (error path)             |                                     |                                                                   |
| Build summary arrays        | Code                     | Aggregates workflow operation results    | Is Telegram configured            | Anything changed?                   |                                                                   |
| Anything changed?           | If                       | Checks if report should be sent          | Build summary arrays             | Render summary                     |                                                                   |
| Render summary              | Code                     | Builds Telegram markdown summary message | Anything changed?              | Send a message                    | Connecting to a messenger                                        |
| Is Telegram configured?     | If                       | Checks if Telegram chat ID is configured  | Loop Over Items                 | Build summary arrays               |                                                                   |
| Send a message              | Telegram                 | Sends summary message to Telegram chat   | Render summary                  |                                     |                                                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create the **Configuration** node (Set type)  
- Add parameters:  
  - `repo.owner` (string)  
  - `repo.name` (string)  
  - `repo.path` (string, default: "workflows/")  
  - `report.tg.chatID` (number or null)  
  - `report.verbose` (boolean)  
- Position it near the start of the workflow.

**Step 2:** Create **Schedule Trigger** node  
- Set interval to hourly (or your desired schedule).  
- Connect output to **Configuration** node.

**Step 3:** Add **Assert GitHub config** (If node)  
- Conditions: Check that `repo.owner`, `repo.name`, and `repo.path` are non-empty strings from Configuration.  
- True output connects to both **Get all workflows** and **List files** nodes.  
- False output connects to **Stop on empty config** node.

**Step 4:** Add **Stop on empty config** (StopAndError node)  
- Set error message: "Incomplete GitHub configuration. Please check \"Configuration\" node."

**Step 5:** Add **Get all workflows** (n8n API node)  
- Use n8n API credentials.  
- No filters needed.  
- Connect output to **Encode N8N workflows** node.

**Step 6:** Add **List files** (GitHub node)  
- Owner: `={{ $("Configuration").item.json.repo.owner }}`  
- Repository: `={{ $("Configuration").item.json.repo.name }}`  
- File path: `={{ $("Configuration").item.json.repo.path }}`  
- Operation: list files  
- GitHub credentials required  
- Connect output to **Get files** node.

**Step 7:** Add **Get files** (GitHub node)  
- Owner: same as above  
- Repository: same as above  
- File path: `={{ $json.path }}` from List files  
- Operation: get file content  
- Credentials: GitHub  
- On error: continue execution  
- Connect output to **Extract workflow parameters** node.

**Step 8:** Add **Extract workflow parameters** (Code node)  
- JavaScript to decode base64 content, parse JSON, extract `id`, `name`, `path`, and base64 content for GitHub workflows.  
- Connect output to **Merge** node.

**Step 9:** Add **Encode N8N workflows** (Code node)  
- JavaScript to encode each n8n workflow JSON to base64 and return id, name, and encoded data.  
- Connect output to **Merge** node.

**Step 10:** Add **Merge** node  
- Mode: Combine  
- Join mode: enrichInput1  
- Match on `id` to merge encoded n8n workflows and GitHub workflows.  
- Connect output to **Decide changes** node.

**Step 11:** Add **Decide changes** (Code node)  
- JavaScript to compare workflows and decide operation: create, update, rename, skip.  
- Outputs enriched items with context and flags.  
- Connect output to **Router** node.

**Step 12:** Add **Router** (Switch node)  
- Rules based on `context.operation` field: rename, update, create, skip.  
- Fallback output leads to **Stop and Error** node.  
- Connect rename to **Delete old file**  
- Connect update to **Update file content and commit**  
- Connect create to **Create new file**  
- Connect skip to **No Operation, do nothing**

**Step 13:** Add **Delete old file** (GitHub node)  
- Deletes old file path for rename operation.  
- Commit message: `"rename: oldName -> newName (step 1/2: remove old)"`  
- Connect output to **Merge after delete (rename)** node.

**Step 14:** Add **Merge after delete (rename)** (Merge node)  
- Combine, enrichInput1, match on `id`.  
- Connect output to **Create new file (rename)** node.

**Step 15:** Add **Create new file (rename)** (GitHub node)  
- Creates new file with new name after delete.  
- Commit message: `"rename: oldName -> newName (step 2/2: create new)"`  
- Connect output to **Merge after create (rename)** node.

**Step 16:** Add **Merge after create (rename)** (Merge node)  
- Combine, enrichInput1, match on `id`.  
- Connect output back to **Loop Over Items** node.

**Step 17:** Add **Update file content and commit** (GitHub node)  
- Edits existing file content.  
- Commit message: `"update: <workflow name>"`  
- Connect output to **Merge after update** node.

**Step 18:** Add **Merge after update** (Merge node)  
- Combine, enrichInput1, match on `id`.  
- Connect output back to **Loop Over Items** node.

**Step 19:** Add **Create new file** (GitHub node)  
- Creates new file for new workflows.  
- Commit message: `"create: <workflow name>"`  
- Connect output to **Merge after create** node.

**Step 20:** Add **Merge after create** (Merge node)  
- Combine, enrichInput1, match on `id`.  
- Connect output back to **Loop Over Items** node.

**Step 21:** Add **No Operation, do nothing** (NoOp node)  
- For skipped workflows.  
- Connect output back to **Loop Over Items** node.

**Step 22:** Add **Stop and Error** (StopAndError node)  
- Error message: `"Invalid operation: \"{{ $json.context.operation }}\". You should look at the code in the \"Decide changes\" node."`

**Step 23:** Add **Loop Over Items** (SplitInBatches node)  
- Controls processing of each workflow item sequentially.  
- Connect outputs from **Merge after create**, **Merge after update**, **Merge after create (rename)**, **Merge after delete (rename)**, and **No Operation** nodes back to this node.  
- Connect first output (index 0) to **Is Telegram configured?** and second output (index 1) to **Decide changes** for processing.

**Step 24:** Add **Is Telegram configured?** (If node)  
- Condition: `report.tg.chatID` exists and is not zero.  
- True output connects to **Build summary arrays** node.

**Step 25:** Add **Build summary arrays** (Code node)  
- Aggregates counts and lists of created, updated, renamed, and skipped workflows.  
- Connect output to **Anything changed?** node.

**Step 26:** Add **Anything changed?** (If node)  
- Condition: Checks if verbose reporting is enabled or if any changes occurred.  
- True output connects to **Render summary** node.

**Step 27:** Add **Render summary** (Code node)  
- Builds detailed MarkdownV2 summary message suitable for Telegram.  
- Connect output to **Send a message** node.

**Step 28:** Add **Send a message** (Telegram node)  
- Uses Telegram API credentials.  
- Sends the summary message to the configured chat ID with MarkdownV2 parsing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automatically backs up your n8n workflows to a GitHub repository with intelligent change detection, rename support, and efficient commit handling to keep the repo clean. It supports optional Telegram reporting and is designed for easy configuration and scheduling. Future updates plan to include archiving inactive workflows and performance optimizations. | Detailed explanation and setup instructions in the sticky note titled "Advanced n8n Workflow Sync with GitHub". |
| For Telegram message formatting, MarkdownV2 escaping is implemented carefully to avoid formatting errors. See [Telegram MarkdownV2](https://core.telegram.org/bots/api#markdownv2-style).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Telegram API MarkdownV2 documentation                                                             |
| Rename operation on GitHub is implemented as a two-step process: first deleting the old file, then creating the new file with the updated name. This preserves history and avoids conflicts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note "Rename a file (two-step)"                                                             |
| The workflow handles missing GitHub folders or files gracefully by continuing execution rather than failing, ensuring robustness.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Notes on List files and Get files nodes                                                           |
| The workflow uses base64 encoding of JSON to avoid data pollution and allow reliable comparison of workflows between n8n and GitHub.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Explanation in Encode N8N workflows and Decide changes nodes                                       |
| To set up credentials, create and configure: GitHub API credential, n8n API credential (for fetching workflows), and optionally Telegram API credential for notifications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Workflow preconditions                                                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.