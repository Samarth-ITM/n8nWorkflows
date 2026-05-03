Daily Workflow Backup to GitLab with Slack Notifications

https://n8nworkflows.xyz/workflows/daily-workflow-backup-to-gitlab-with-slack-notifications-9458


# Daily Workflow Backup to GitLab with Slack Notifications

### 1. Workflow Overview

This workflow automates the daily backup of all active n8n workflows into a GitLab repository and sends Slack notifications about the backup status. It is designed for n8n administrators or DevOps teams who want to maintain versioned backups of their workflows in GitLab and receive real-time alerts on Slack.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Configuration**: Initiates the backup process daily at a set time and loads GitLab project configuration.
- **1.2 Existing Backup Files Retrieval**: Lists all files currently present in the GitLab repository to compare against workflows.
- **1.3 Workflow Retrieval & Filtering**: Fetches all workflows from n8n, filters out archived ones, and prepares them for backup.
- **1.4 Workflow Processing Loop**: Iterates over each workflow, cleans unnecessary fields, converts workflow JSON to a file format, checks if a corresponding file exists in GitLab, and either updates or creates the file accordingly.
- **1.5 Error Handling and Notifications**: For any GitLab file creation or update failure, sends detailed Slack messages; upon successful completion, sends a success notification to Slack.
- **1.6 Utility and Helper Nodes**: Includes nodes for transforming data formats and aggregating workflow names.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration

- **Overview**  
  Starts the workflow on a daily schedule and sets project-specific GitLab parameters.

- **Nodes Involved**  
  - Daily Trigger  
  - Configuration

- **Node Details**

  - **Daily Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Fires daily at 12:00 (noon) server time  
    - Inputs: None (trigger node)  
    - Outputs: Configuration node  
    - Edge Cases: Workflow will not run if n8n instance is down at trigger time

  - **Configuration**  
    - Type: Set  
    - Purpose: Stores GitLab project owner, project name, and branch name as static parameters for downstream nodes  
    - Parameters:  
      - `project_owner` (string): e.g., "mookielian"  
      - `project_name` (string): e.g., "n8n"  
      - `branch` (string): e.g., "main"  
    - Inputs: Daily Trigger  
    - Outputs: List All Files - GitLab  
    - Edge Cases: Misconfiguration leads to failed GitLab API calls  
    - Notes: Requires manual update per project

#### 1.2 Existing Backup Files Retrieval

- **Overview**  
  Retrieves all files currently stored in the specified GitLab repository branch to detect which workflows already have backup files.

- **Nodes Involved**  
  - List All Files - GitLab  
  - List of Names

- **Node Details**

  - **List All Files - GitLab**  
    - Type: GitLab  
    - Operation: List files in the configured repository and branch  
    - Inputs: Configuration node  
    - Outputs: List of Names  
    - Credentials: GitLab API credential configured with access token and project info  
    - Edge Cases: API failures, permission errors, rate limits

  - **List of Names**  
    - Type: Aggregate  
    - Purpose: Extracts just the file names from the list of files for quick lookup  
    - Inputs: List All Files - GitLab  
    - Outputs: Get All Workflows  
    - Configuration: Aggregates "name" field into an array  
    - Edge Cases: Empty repository returns empty array

#### 1.3 Workflow Retrieval & Filtering

- **Overview**  
  Gets all workflows from n8n, filters out archived workflows (optional), and prepares the list for further processing.

- **Nodes Involved**  
  - Get All Workflows  
  - Discard Archived Workflows

- **Node Details**

  - **Get All Workflows**  
    - Type: n8n API node (n8n)  
    - Operation: Fetches all workflows using the n8n internal API with an API key credential  
    - Inputs: List of Names  
    - Outputs: Discard Archived Workflows  
    - Credentials: Internal n8n API credential with API key and base URL  
    - Edge Cases: API server unavailability, authentication errors

  - **Discard Archived Workflows**  
    - Type: Filter  
    - Purpose: Removes workflows where `isArchived` flag is `true` to avoid backing up archived workflows  
    - Inputs: Get All Workflows  
    - Outputs: Process Each File  
    - Logic: Condition `isArchived` equals false  
    - Edge Cases: If all workflows are archived, downstream processing is skipped

#### 1.4 Workflow Processing Loop

- **Overview**  
  Iterates over each active workflow to clean unnecessary metadata, convert it to a formatted JSON file, check if the file already exists in GitLab, and then update or create the file accordingly.

- **Nodes Involved**  
  - Process Each File  
  - Remove Unwanted Fields  
  - Convert to File  
  - If File Exists  
  - Update File - GitLab  
  - Create New File - GitLab

- **Node Details**

  - **Process Each File**  
    - Type: SplitInBatches  
    - Purpose: Processes workflows one by one to avoid API overload and manage flow control  
    - Inputs: Discard Archived Workflows  
    - Outputs: Send Message to Channel (on completion), Remove Unwanted Fields (per item)  
    - Edge Cases: Large number of workflows may slow down processing

  - **Remove Unwanted Fields**  
    - Type: Code (JavaScript)  
    - Purpose: Cleans workflow JSON by removing metadata fields such as `createdAt`, `updatedAt`, `id`, `name`, `active`, and `isArchived` to reduce file size and avoid redundancy  
    - Inputs: Process Each File  
    - Outputs: Convert to File  
    - Key Script: Returns cleaned JSON excluding specified fields  
    - Edge Cases: If input data structure changes, script may fail or omit fields incorrectly

  - **Convert to File**  
    - Type: ConvertToFile  
    - Operation: Converts each cleaned workflow JSON into a formatted JSON file per workflow (mode: each, operation: toJson)  
    - Inputs: Remove Unwanted Fields  
    - Outputs: If File Exists  
    - Edge Cases: Large workflows may produce large files; conversion errors if JSON malformed

  - **If File Exists**  
    - Type: If  
    - Purpose: Checks if the workflow's file name exists in the GitLab repository file list to decide update vs. create path  
    - Inputs: Convert to File  
    - Outputs:  
      - True branch: Update File - GitLab  
      - False branch: Create New File - GitLab  
    - Logic: Checks if current workflow name exists in the aggregated file names array  
    - Edge Cases: Name mismatches or case sensitivity may cause wrong routing

  - **Update File - GitLab**  
    - Type: GitLab  
    - Operation: Edits the existing file in GitLab with new content (binaryData from Convert to File)  
    - Inputs: If File Exists (true branch)  
    - Outputs: Process Each File (continue) and Update File - Failed (on error)  
    - Commit message includes the file name and "File was updated!"  
    - Error Handling: Continues on error and triggers Slack notification node  
    - Edge Cases: GitLab API errors, file locked, no changes (GitLab may reject commit if no diff)

  - **Create New File - GitLab**  
    - Type: GitLab  
    - Operation: Creates a new file in the GitLab repository with the workflow JSON content  
    - Inputs: If File Exists (false branch)  
    - Outputs: Process Each File (continue) and New File - Failed (on error)  
    - Commit message includes the file name and "File created!"  
    - Error Handling: Continues on error and triggers Slack notification node  
    - Edge Cases: Duplicate file creation, permission errors

#### 1.5 Error Handling and Notifications

- **Overview**  
  Sends real-time Slack notifications on errors during GitLab operations and a success notification when the entire backup process completes.

- **Nodes Involved**  
  - Update File - Failed  
  - New File - Failed  
  - Send Message to Channel

- **Node Details**

  - **Update File - Failed**  
    - Type: Slack  
    - Purpose: Sends Slack message if updating a GitLab file fails  
    - Inputs: Update File - GitLab (error output)  
    - Parameters: Channel ID fixed, message includes file name, error details, execution ID, mode, and timestamp  
    - Credentials: Slack API credential with bot token  
    - Edge Cases: Slack API rate limits or invalid token

  - **New File - Failed**  
    - Type: Slack  
    - Purpose: Sends Slack message if creating a new GitLab file fails  
    - Inputs: Create New File - GitLab (error output)  
    - Parameters: Similar to Update File - Failed, customized message text  
    - Credentials: Slack API credential  
    - Edge Cases: Same as above

  - **Send Message to Channel**  
    - Type: Slack  
    - Purpose: Sends a success notification to Slack after processing all workflows  
    - Inputs: Process Each File (completion path)  
    - Parameters: Includes execution ID, mode, and timestamp in message, posts to the configured Slack channel  
    - Edge Cases: Slack API errors, message formatting issues

#### 1.6 Utility and Helper Nodes

- **Overview**  
  Support nodes for data aggregation and user documentation via sticky notes.

- **Nodes Involved**  
  - Sticky Notes (multiple)

- **Node Details**

  - Sticky Notes provide extensive documentation on node summaries, credential setup instructions for GitLab, n8n internal API, and Slack, as well as workflow structure explanation. They do not affect the workflow execution.

---

### 3. Summary Table

| Node Name                | Node Type                 | Functional Role                                   | Input Node(s)             | Output Node(s)                                  | Sticky Note                                                                                                          |
|--------------------------|---------------------------|-------------------------------------------------|---------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Daily Trigger            | Schedule Trigger          | Starts workflow daily at noon                     |                           | Configuration                                   |                                                                                                                      |
| Configuration           | Set                       | Defines GitLab project owner, repo, and branch   | Daily Trigger             | List All Files - GitLab                         | See "Configuration Node" sticky note for setup instructions                                                         |
| List All Files - GitLab  | GitLab                    | Lists all files in GitLab repo branch             | Configuration             | List of Names                                   | See "GitLab" sticky note with credentials instructions                                                               |
| List of Names            | Aggregate                 | Aggregates file names into an array                | List All Files - GitLab   | Get All Workflows                               |                                                                                                                      |
| Get All Workflows        | n8n API                   | Retrieves all workflows from n8n                   | List of Names             | Discard Archived Workflows                      | See "n8n Internal" sticky note for API credential setup                                                               |
| Discard Archived Workflows| Filter                   | Filters out archived workflows                      | Get All Workflows         | Process Each File                               |                                                                                                                      |
| Process Each File        | SplitInBatches            | Iterates workflows one by one                       | Discard Archived Workflows| Send Message to Channel, Remove Unwanted Fields|                                                                                                                      |
| Remove Unwanted Fields   | Code                      | Cleans unnecessary workflow metadata                | Process Each File         | Convert to File                                 |                                                                                                                      |
| Convert to File          | ConvertToFile             | Converts JSON to formatted file                     | Remove Unwanted Fields    | If File Exists                                  |                                                                                                                      |
| If File Exists           | If                        | Checks if workflow backup file exists in GitLab    | Convert to File           | Update File - GitLab (true), Create New File - GitLab (false) |                                                                                                                      |
| Update File - GitLab     | GitLab                    | Updates existing file in GitLab                      | If File Exists            | Process Each File, Update File - Failed         |                                                                                                                      |
| Create New File - GitLab | GitLab                    | Creates new file in GitLab                           | If File Exists            | Process Each File, New File - Failed             |                                                                                                                      |
| Update File - Failed     | Slack                     | Notifies Slack on GitLab file update failure        | Update File - GitLab      | Process Each File                               | See "Slack" sticky note for OAuth and token setup instructions                                                       |
| New File - Failed        | Slack                     | Notifies Slack on GitLab file creation failure      | Create New File - GitLab  | Process Each File                               | See "Slack" sticky note                                                                                                |
| Send Message to Channel  | Slack                     | Sends success notification after backup completion | Process Each File         |                                                | See "Slack" sticky note                                                                                                |
| Sticky Note              | Sticky Note               | Documentation and summary                           |                           |                                                | Covers entire workflow explanation and node summaries                                                                |
| Sticky Note1             | Sticky Note               | Empty (placeholder)                                 |                           |                                                |                                                                                                                      |
| Sticky Note2             | Sticky Note               | Credentials header                                  |                           |                                                |                                                                                                                      |
| Sticky Note3             | Sticky Note               | Configuration node instructions                     |                           |                                                |                                                                                                                      |
| Sticky Note4             | Sticky Note               | GitLab credential instructions                      |                           |                                                |                                                                                                                      |
| Sticky Note5             | Sticky Note               | n8n internal API credential instructions            |                           |                                                |                                                                                                                      |
| Sticky Note6             | Sticky Note               | Slack credential instructions and OAuth setup       |                           |                                                | [Slack Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/)                                |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create **Daily Trigger** node  
- Type: Schedule Trigger  
- Set to trigger daily at 12:00 (noon)  
- No credentials needed

**Step 2:** Create **Configuration** node  
- Type: Set  
- Add three string fields:  
  - `project_owner` (e.g., "mookielian")  
  - `project_name` (e.g., "n8n")  
  - `branch` (e.g., "main")  
- Connect output of Daily Trigger to input of Configuration node

**Step 3:** Create **List All Files - GitLab** node  
- Type: GitLab  
- Operation: List files  
- Resource: file  
- Repository: `={{ $json.project_name }}` from Configuration  
- Owner: `={{ $json.project_owner }}`  
- Branch: `={{ $json.branch }}`  
- Credential: GitLab API credential configured with a valid personal access token and project access  
- Connect output of Configuration to input of this node

**Step 4:** Create **List of Names** node  
- Type: Aggregate  
- Aggregate field: `name` (field from GitLab file list)  
- Connect output of List All Files - GitLab to input of this node

**Step 5:** Create **Get All Workflows** node  
- Type: n8n API (n8n)  
- Operation: Get all workflows (no filter)  
- Credential: Internal n8n API credential with API key and base URL (e.g., https://your-n8n-domain/api/v1)  
- Connect output of List of Names to input of this node

**Step 6:** Create **Discard Archived Workflows** node  
- Type: Filter  
- Condition: `isArchived` equals false (boolean false)  
- Connect output of Get All Workflows to input of this node

**Step 7:** Create **Process Each File** node  
- Type: SplitInBatches  
- No special options, defaults suffice  
- Connect output of Discard Archived Workflows to input of this node

**Step 8:** Create **Remove Unwanted Fields** node  
- Type: Code (JavaScript)  
- Script: Remove fields `createdAt`, `updatedAt`, `id`, `name`, `active`, `isArchived` from each workflow JSON  
- Connect output of Process Each File to input of this node

**Step 9:** Create **Convert to File** node  
- Type: ConvertToFile  
- Mode: Each  
- Operation: toJson  
- Options: format JSON output  
- Connect output of Remove Unwanted Fields to input of this node

**Step 10:** Create **If File Exists** node  
- Type: If  
- Condition: Check if current workflow name (from `$('List of Names').item.json.name`) is contained in the array of existing file names (from `$('Process Each File').item.json.name`)  
- Connect output of Convert to File to input of this node

**Step 11:** Create **Update File - GitLab** node  
- Type: GitLab  
- Operation: Edit file  
- Resource: file  
- Owner, Repository, Branch: use expressions from Configuration node  
- FilePath: `={{ $('Process Each File').item.json.name }}`  
- Commit message: "File was updated! [filename]"  
- Use binary data from Convert to File node  
- Credential: GitLab API  
- Connect True output of If File Exists node to input of this node

**Step 12:** Create **Create New File - GitLab** node  
- Type: GitLab  
- Operation: Create file  
- Same configuration as Update File node but operation is create  
- Connect False output of If File Exists node to input of this node

**Step 13:** Create **Update File - Failed** node  
- Type: Slack  
- Post message on failure of Update File - GitLab node  
- Message includes file name, error, execution details, timestamp  
- Credential: Slack bot token with necessary scopes  
- Connect error output of Update File - GitLab node to input of this node

**Step 14:** Create **New File - Failed** node  
- Type: Slack  
- Post message on failure of Create New File - GitLab node  
- Similar message structure as above  
- Connect error output of Create New File - GitLab node to input of this node

**Step 15:** Connect success output of Update File and Create New File nodes back to **Process Each File** node to continue the loop

**Step 16:** Create **Send Message to Channel** node  
- Type: Slack  
- Sends a success notification after all files processed  
- Message includes execution ID, mode, timestamp  
- Credential: Slack bot token  
- Connect completion output of Process Each File node to this node

**Step 17:** Optional: Add Sticky Notes to document credentials setup, node purposes, and workflow overviews as per original workflow

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Create a GitLab project and generate a personal access token with appropriate repository permissions. Use this token in the GitLab API credential in n8n. | GitLab Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gitlab/ |
| Generate an API key in n8n user settings for the Internal API node. Ensure base URL is your n8n domain with `/api/v1` appended. | n8n Internal API Docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n/ |
| Create a Slack app with bot token scopes: chat:write, channels:join, channels:read, groups:read. Use the OAuth token in Slack API credential. | Slack API Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/ |
| Slack notifications include execution ID, mode (manual/triggered), and timestamp for traceability. | - |
| The workflow excludes archived workflows by default but can be modified to include them by adjusting the filter node. | - |
| Commit messages in GitLab are dynamic and include the file name and operation type (created/updated). | - |
| The workflow uses SplitInBatches node to safely process workflows one at a time, avoiding API overload and rate limits. | - |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow. It fully complies with current content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.