Version Control n8n Workflows in GitLab with Customer Tag Organization

https://n8nworkflows.xyz/workflows/version-control-n8n-workflows-in-gitlab-with-customer-tag-organization-8042


# Version Control n8n Workflows in GitLab with Customer Tag Organization

### 1. Workflow Overview

This workflow automates the version control backup of n8n workflows into a GitLab repository, organizing backups with customer-specific tagging. It targets organizations that manage multiple workflows, needing traceable, stable, and customer-aware backup solutions. The workflow runs either manually or on a daily schedule, filters workflows tagged for backup, normalizes their metadata, compares with existing GitLab files, and performs conditional create/update operations to maintain a clean, versioned repository.

The workflow logic is divided into the following blocks:

- **1.1 Start & Initialization:** Triggers (manual or scheduled) and global variable setup for GitLab and execution context.
- **1.2 Workflow Selection & Normalization:** Fetches workflows tagged for backup, normalizes names to standardize customer tags.
- **1.3 Workflow JSON Preparation:** Cleans workflow JSON to match n8n UI export format and prepares GitLab file paths using workflow IDs and customer tags.
- **1.4 GitLab File Management:** Checks for existing files in GitLab, compares JSON content, and decides to create, update, or skip backups.
- **1.5 Output Normalization & Summary:** Tags backup results with statuses, merges outputs, and summarizes execution statistics for reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Start & Initialization

**Overview:**  
This block starts the workflow either manually or via a daily schedule at 03:00 server time. It then sets global variables used throughout the workflow, including GitLab repository details, branch, backup tag, and execution metadata.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Schedule Trigger  
- Set Global GitLab Variables

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing or immediate backup runs.  
  - Configuration: No parameters; triggers when user clicks execute.  
  - Inputs: None  
  - Outputs: To `Set Global GitLab Variables`  
  - Edge Cases: None expected; manual trigger always available.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow daily at 03:00 server time.  
  - Configuration: Trigger at hour 3 (3 AM) daily.  
  - Inputs: None  
  - Outputs: To `Set Global GitLab Variables`  
  - Edge Cases: Timezone differences may affect exact trigger time; ensure server timezone matches expectations.

- **Set Global GitLab Variables**  
  - Type: Set  
  - Role: Defines reusable GitLab and execution context variables across the workflow.  
  - Configuration:  
    - `gitlab_owner`: "n8n-ainexusone" (GitLab owner/group)  
    - `gitlab_project`: "n8n_workflow_backups" (repository)  
    - `gitlab_workflow_path`: "workflow_definitions" (root storage path)  
    - `gitlab_branch`: "main"  
    - `tag_backup`: "backup-workflows" (tag filter for workflows)  
    - `execution_type`: Expression evaluating if the Schedule Trigger executed → "Scheduled", else "Manual"  
  - Inputs: From triggers  
  - Outputs: To `Fetch Workflows from n8n`  
  - Edge Cases: If triggers are misconfigured, execution_type might not reflect actual run type.

---

#### 2.2 Workflow Selection & Normalization

**Overview:**  
Retrieves workflows tagged with "backup-workflows" using the n8n API, then normalizes workflow names to standardize `[customer]` tags, enabling customer-specific organization.

**Nodes Involved:**  
- Fetch Workflows from n8n  
- Clean & Normalize Workflow Name

**Node Details:**

- **Fetch Workflows from n8n**  
  - Type: n8n API node  
  - Role: Fetches only workflows tagged with the backup tag from n8n instance via API.  
  - Configuration: Filter workflows by tag equal to `{{$json.tag_backup}}` (passed from global variables)  
  - Credentials: Uses stored n8n API credentials  
  - Inputs: From `Set Global GitLab Variables`  
  - Outputs: To `Clean & Normalize Workflow Name`  
  - Edge Cases: API authentication errors, no workflows found, or tag mismatch.

- **Clean & Normalize Workflow Name**  
  - Type: Code (JavaScript)  
  - Role: Standardizes workflow names by detecting and normalizing `[customer]` tags to uppercase format `[customer : NAME]`. Removes empty tags.  
  - Key Logic:  
    - Regex extracts `[customer ...]` tag variants.  
    - If found and non-empty, converts customer name to uppercase and prepends it normalized.  
    - If empty, removes tag.  
  - Inputs: Workflows from previous node  
  - Outputs: To `Prepare Workflow JSON for UI-Compatible Export`  
  - Edge Cases: Workflow names without tags remain unchanged; malformed tags may not normalize correctly.

---

#### 2.3 Workflow JSON Preparation

**Overview:**  
Prepares workflow JSON in a clean format identical to n8n UI export, and generates a GitLab file path for storage based on workflow ID and customer tag slug.

**Nodes Involved:**  
- Prepare Workflow JSON for UI-Compatible Export  
- Prepare GitLab File Path

**Node Details:**

- **Prepare Workflow JSON for UI-Compatible Export**  
  - Type: Code  
  - Role: Cleans and normalizes the workflow JSON to contain only fields present in a native n8n export, ensuring compatibility with UI import/export.  
  - Key Output Fields: name, nodes, pinData, connections, active, settings, versionId, meta, id, tags  
  - Inputs: Normalized workflow JSON from previous node  
  - Outputs: To `Prepare GitLab File Path`  
  - Edge Cases: Missing fields defaulted appropriately; malformed JSON could cause errors.

- **Prepare GitLab File Path**  
  - Type: Code  
  - Role: Builds a normalized file path for GitLab backup: `workflow_definitions/<customerSlug>/<workflowId>.json`.  
  - Key Logic:  
    - Extract customer tag from workflow name.  
    - Slugifies customer name (lowercase, no accents, hyphenated) or uses `unassigned` if missing.  
    - Combines with base path from global variables and workflow ID for stable, rename-proof filename.  
  - Inputs: Workflow JSON from previous node  
  - Outputs: To `Fetch Existing File from GitLab`  
  - Edge Cases: Customer tag missing → defaults to `unassigned`. Slugify handles special characters gracefully.

---

#### 2.4 GitLab File Management

**Overview:**  
Manages file existence and content comparison in GitLab repository to decide whether to create a new backup file, update an existing one, or skip unchanged workflows.

**Nodes Involved:**  
- Fetch Existing File from GitLab  
- Compare Workflow with GitLab Version  
- Create New File in GitLab  
- Update Existing File in GitLab  
- Mark as Created  
- Mark as Updated  
- Mark as Unchanged  
- Merge

**Node Details:**

- **Fetch Existing File from GitLab**  
  - Type: GitLab node  
  - Role: Attempts to fetch the existing backup file from GitLab for the workflow path.  
  - Configuration: Parameters sourced from global variables and current item JSON.  
  - On error (file not found): Continues to `Create New File in GitLab` via error output.  
  - Inputs: Workflow JSON with file path from previous node  
  - Outputs:  
    - Success: To `Compare Workflow with GitLab Version`  
    - Error: To `Create New File in GitLab`  
  - Edge Cases: Network issues, authentication failure, file missing (expected for new workflows).

- **Compare Workflow with GitLab Version**  
  - Type: If node  
  - Role: Compares current exported workflow JSON with GitLab's existing file content (decoded from base64).  
  - Comparison: JSON stringified objects, strict and case-sensitive.  
  - Outputs:  
    - True (not equal): To `Update Existing File in GitLab`  
    - False (equal): To `Mark as Unchanged`  
  - Inputs: Output from `Fetch Existing File from GitLab` and current workflow JSON  
  - Edge Cases: JSON parsing errors, encoding inconsistencies.

- **Create New File in GitLab**  
  - Type: GitLab node  
  - Role: Creates a new file in the GitLab repo with exported workflow JSON if the file does not exist.  
  - Commit message: "Add backup for workflow: <normalized name> (<file path>)"  
  - Inputs: From error output of `Fetch Existing File from GitLab`  
  - Outputs: To `Mark as Created`  
  - Edge Cases: Permission or quota issues on GitLab, invalid file path.

- **Update Existing File in GitLab**  
  - Type: GitLab node  
  - Role: Updates the existing GitLab file only if JSON content differs.  
  - Commit message: "Update backup for workflow: <normalized name> (<file path>)"  
  - Inputs: From `Compare Workflow with GitLab Version` (true branch)  
  - Outputs: To `Mark as Updated`  
  - Edge Cases: Conflicts if repository changed externally, permissions.

- **Mark as Created**  
  - Type: Set  
  - Role: Tags workflow output with status `created` for reporting.  
  - Inputs: From `Create New File in GitLab`  
  - Outputs: To `Merge`

- **Mark as Updated**  
  - Type: Set  
  - Role: Tags workflow output with status `updated`.  
  - Inputs: From `Update Existing File in GitLab`  
  - Outputs: To `Merge`

- **Mark as Unchanged**  
  - Type: Set  
  - Role: Tags workflow output with status `unchanged`.  
  - Inputs: From `Compare Workflow with GitLab Version` (false branch)  
  - Outputs: To `Merge`

- **Merge**  
  - Type: Merge  
  - Role: Combines three streams (created, updated, unchanged) into a single output stream.  
  - Inputs: From `Mark as Updated`, `Mark as Unchanged`, `Mark as Created`  
  - Outputs: To `Normalize Backup Output`

---

#### 2.5 Output Normalization & Summary

**Overview:**  
Consolidates all backup outputs, adds global execution data, and summarizes counts of created, updated, and unchanged workflows for monitoring or logging purposes.

**Nodes Involved:**  
- Normalize Backup Output  
- Summarize Backup Results

**Node Details:**

- **Normalize Backup Output**  
  - Type: Set  
  - Role: Enriches each workflow output with additional metadata for reporting: status, workflow name, file path, GitLab details, execution type, and execution time.  
  - Inputs: From `Merge` node  
  - Outputs: To `Summarize Backup Results`  
  - Edge Cases: Missing fields in input JSON could result in incomplete metadata.

- **Summarize Backup Results**  
  - Type: Code  
  - Role: Summarizes all backup results by counting how many workflows were created, updated, unchanged, and total processed. Also includes execution context.  
  - Outputs: A single item with recap and execution metadata.  
  - Inputs: From `Normalize Backup Output`  
  - Edge Cases: No workflows processed (empty input), status fields missing.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                                  | Input Node(s)                      | Output Node(s)                                   | Sticky Note                                                                                                                      |
|----------------------------------|---------------------------|-------------------------------------------------|----------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual workflow start trigger                    | None                             | Set Global GitLab Variables                      | Manual trigger for testing the workflow execution.                                                                              |
| Schedule Trigger                 | Schedule Trigger          | Scheduled daily workflow start trigger           | None                             | Set Global GitLab Variables                      | Runs the workflow daily at 03:00 (server time).                                                                                  |
| Set Global GitLab Variables       | Set                       | Defines global GitLab and execution variables    | When clicking ‘Execute workflow’, Schedule Trigger | Fetch Workflows from n8n                      | Defines global GitLab variables (owner, project, branch, paths, tags, execution type) for reuse across the workflow.            |
| Fetch Workflows from n8n          | n8n API                   | Retrieves workflows tagged for backup            | Set Global GitLab Variables       | Clean & Normalize Workflow Name                  | Fetches only workflows tagged "backup-workflows" via n8n API.                                                                   |
| Clean & Normalize Workflow Name   | Code                      | Standardizes workflow names and customer tags    | Fetch Workflows from n8n          | Prepare Workflow JSON for UI-Compatible Export   | Cleans and normalizes workflow name: applies customer tag (uppercase) or removes it if missing.                                  |
| Prepare Workflow JSON for UI-Compatible Export | Code                   | Cleans and normalizes workflow JSON for export   | Clean & Normalize Workflow Name   | Prepare GitLab File Path                          | Cleans and normalizes workflow JSON to match n8n export format (only required fields).                                           |
| Prepare GitLab File Path          | Code                      | Creates stable GitLab file path based on ID and customer tag | Prepare Workflow JSON for UI-Compatible Export | Fetch Existing File from GitLab                   | Builds a normalized GitLab file path for the workflow backup (workflowId + .json)                                               |
| Fetch Existing File from GitLab   | GitLab                    | Fetches existing backup file from GitLab         | Prepare GitLab File Path          | Compare Workflow with GitLab Version, Create New File in GitLab (on error) | Fetches the existing workflow backup file from GitLab.                                                                           |
| Compare Workflow with GitLab Version | If                      | Compares current workflow JSON with GitLab copy  | Fetch Existing File from GitLab   | Update Existing File in GitLab, Mark as Unchanged | Compares exported workflow JSON with the GitLab version to detect changes.                                                      |
| Create New File in GitLab         | GitLab                    | Creates new backup file in GitLab                  | Fetch Existing File from GitLab (error output) | Mark as Created                                  | Creates a new workflow backup file in GitLab if it does not already exist.                                                      |
| Update Existing File in GitLab    | GitLab                    | Updates existing backup file if changed           | Compare Workflow with GitLab Version (true) | Mark as Updated                                  | Updates the existing workflow backup file in GitLab with the latest JSON export.                                                |
| Mark as Created                  | Set                       | Tags output as "created"                           | Create New File in GitLab        | Merge                                            | Tags workflow as "created" (new file added in GitLab).                                                                          |
| Mark as Updated                  | Set                       | Tags output as "updated"                           | Update Existing File in GitLab   | Merge                                            | Tags workflow as "updated" after backup comparison.                                                                              |
| Mark as Unchanged                | Set                       | Tags output as "unchanged"                         | Compare Workflow with GitLab Version (false) | Merge                                            | Tags workflow as "unchanged" (no differences found).                                                                            |
| Merge                           | Merge                     | Combines created, updated, unchanged outputs     | Mark as Updated, Mark as Unchanged, Mark as Created | Normalize Backup Output                          | Merges outputs from "Mark as Updated/Unchanged/Created" into a single stream.                                                    |
| Normalize Backup Output          | Set                       | Enriches outputs with GitLab and execution metadata | Merge                           | Summarize Backup Results                          | Normalizes backup output: adds GitLab path, branch, owner, project, execution type & timestamp.                                  |
| Summarize Backup Results         | Code                      | Summarizes backup status counts and execution context | Normalize Backup Output          | None                                             | Summarizes backup results: counts created/updated/unchanged workflows and adds execution metadata.                              |
| Sticky Note (multiple nodes)     | Sticky Note               | Documentation and best practices notes            | None                             | None                                             | See detailed notes in workflow describing best practices, goals, and functional blocks.                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" with default settings.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" configured to run daily at 03:00 server time.

2. **Set Global Variables:**  
   - Add a **Set** node named "Set Global GitLab Variables" with fields:  
     - `gitlab_owner`: "n8n-ainexusone"  
     - `gitlab_project`: "n8n_workflow_backups"  
     - `gitlab_workflow_path`: "workflow_definitions"  
     - `gitlab_branch`: "main"  
     - `tag_backup`: "backup-workflows"  
     - `execution_type`: Expression: `{{$node["Schedule Trigger"].isExecuted ? "Scheduled" : "Manual"}}`  
   - Connect both triggers to this node.

3. **Fetch Tagged Workflows:**  
   - Add an **n8n** node named "Fetch Workflows from n8n".  
   - Configure to fetch workflows filtered by tag equal to `{{$json.tag_backup}}` (from previous Set node).  
   - Set credentials for n8n API access.  
   - Connect "Set Global GitLab Variables" to this node.

4. **Normalize Workflow Names:**  
   - Add a **Code** node named "Clean & Normalize Workflow Name".  
   - Paste the JavaScript code that detects `[customer]` tags, normalizes to uppercase, removes empty tags, and updates the workflow name accordingly.  
   - Connect "Fetch Workflows from n8n" to this node.

5. **Prepare Workflow JSON for Export:**  
   - Add a **Code** node named "Prepare Workflow JSON for UI-Compatible Export".  
   - Paste code that extracts only necessary fields matching the native n8n export format from the workflow JSON.  
   - Connect "Clean & Normalize Workflow Name" to this node.

6. **Prepare GitLab File Path:**  
   - Add a **Code** node named "Prepare GitLab File Path".  
   - Paste code that builds the file path as `workflow_definitions/<customerSlug>/<workflowId>.json`, slugifying the customer tag or using `unassigned` if missing.  
   - Connect "Prepare Workflow JSON for UI-Compatible Export" to this node.

7. **Fetch Existing GitLab File:**  
   - Add a **GitLab** node named "Fetch Existing File from GitLab".  
   - Configure to get file by path from current JSON (`gitlab_file_path`), using owner, project, branch from global variables.  
   - Set GitLab API credentials.  
   - Enable "On Error → Continue" to handle missing files gracefully.  
   - Connect "Prepare GitLab File Path" to this node.

8. **Compare Workflow JSONs:**  
   - Add an **If** node named "Compare Workflow with GitLab Version".  
   - Condition: Compare JSON.stringify() of current workflow JSON and decoded content from GitLab file; branch "true" if different, "false" if identical.  
   - Connect success output of "Fetch Existing File from GitLab" to this node.

9. **Create or Update Files in GitLab:**  
   - Add a **GitLab** node named "Create New File in GitLab" configured to create a file at `gitlab_file_path` with current JSON content and a commit message referencing workflow name and path.  
   - Connect error output (file not found) of "Fetch Existing File from GitLab" to this node.  
   - Add a **GitLab** node named "Update Existing File in GitLab" configured similarly but for edit operation.  
   - Connect "true" output of "Compare Workflow with GitLab Version" to this node.

10. **Mark Status for Outputs:**  
    - Add three **Set** nodes:  
      - "Mark as Created" sets `status = "created"` (connect from "Create New File in GitLab")  
      - "Mark as Updated" sets `status = "updated"` (connect from "Update Existing File in GitLab")  
      - "Mark as Unchanged" sets `status = "unchanged"` (connect "false" output of "Compare Workflow with GitLab Version")  

11. **Merge Status Outputs:**  
    - Add a **Merge** node named "Merge" configured to merge all three status streams.  
    - Connect all three "Mark as ..." nodes to this node.

12. **Normalize Backup Output:**  
    - Add a **Set** node named "Normalize Backup Output".  
    - Assign fields: `status`, `workflow_name` (from prepared workflow JSON), `file_path`, `branch`, `gitlab_owner`, `gitlab_project`, `execution_type`, `execution_time` (`$now`).  
    - Connect "Merge" to this node.

13. **Summarize Backup Results:**  
    - Add a **Code** node named "Summarize Backup Results".  
    - Paste code that counts created, updated, unchanged workflows, totals them, and includes execution metadata.  
    - Connect "Normalize Backup Output" to this node.

14. **Activate Workflow:**  
    - Verify all connections and credentials.  
    - Activate workflow and test manual and scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Uses workflow ID for GitLab file naming to ensure rename-proof, stable backup paths.                                                                                                           | Best practice for version control and file management.                                          |
| Customer tags in workflow names are normalized to uppercase `[customer : NAME]` for consistency and used to organize backups in subfolders.                                                  | Ensures clean organization and easier identification.                                           |
| Backup only workflows tagged with `backup-workflows` to limit scope and avoid unnecessary backups.                                                                                             | Tag management is critical to backup correctness.                                               |
| Commit messages include normalized workflow names and file paths for traceability.                                                                                                             | Improves GitLab commit clarity and auditing.                                                   |
| JSON comparison is done on parsed objects (via stringified JSON) to avoid false positives due to formatting differences.                                                                       | Prevents unnecessary commits and noise in version history.                                     |
| GitLab token scope should be limited to repository access only for security.                                                                                                                   | Security best practice.                                                                         |
| Scheduled trigger uses server timezone; confirm timezone to align with operational requirements.                                                                                               | Avoids unexpected execution times.                                                             |
| Final backup output includes a recap summary for monitoring dashboards or alerts.                                                                                                              | Facilitates operational monitoring and auditing.                                               |
| For additional setup, see crontab examples: https://crontab.guru/examples.html                                                                                                                 | Reference for schedule trigger configuration.                                                  |
| Maintains human-readable and n8n re-import compatible JSON exports to support recovery and migration scenarios.                                                                                | Ensures backups are usable beyond version control.                                             |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.