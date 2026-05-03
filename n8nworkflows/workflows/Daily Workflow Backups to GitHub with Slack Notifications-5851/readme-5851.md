Daily Workflow Backups to GitHub with Slack Notifications

https://n8nworkflows.xyz/workflows/daily-workflow-backups-to-github-with-slack-notifications-5851


# Daily Workflow Backups to GitHub with Slack Notifications

### 1. Workflow Overview

This workflow automates daily backups of all n8n instance workflows to a specified GitHub repository, organizing them into folders and files named by workflow IDs. It compares the current workflow definitions with those already stored on GitHub to decide whether to create new files or update existing ones. After processing all workflows, it optionally sends a Slack notification confirming completion.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Initialization:** Starts the workflow on a schedule, sends a starting notification, and sets configuration parameters.
- **1.2 Workflow Retrieval & Filtering:** Fetches all current workflows from the n8n instance and filters those updated within the last 24 hours.
- **1.3 Iterative Processing & Comparison:** Processes each workflow item individually, fetching its corresponding backup file from GitHub and comparing contents to detect changes.
- **1.4 GitHub File Management:** Depending on comparison results, either updates existing files or creates new files in the GitHub repo.
- **1.5 Completion & Notification:** Sends a Slack notification after all workflows are processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Initialization

**Overview:**  
This block initiates the workflow execution on a regular schedule and sets up essential configuration data and optional start notifications.

**Nodes Involved:**  
- Schedule Trigger  
- Starting Message (Slack) [disabled]  
- Config (Set node)

**Node Details:**  

- **Schedule Trigger**  
  - Type: scheduleTrigger  
  - Role: Initiates the workflow at a defined interval (default every 24 hours).  
  - Configuration: Default interval set, triggers workflow execution.  
  - Inputs: None  
  - Outputs: Triggers "Starting Message" node.  
  - Edge Cases: Misconfiguration of schedule may cause no triggering or too frequent runs.

- **Starting Message**  
  - Type: Slack node  
  - Role: Posts a start message to a Slack channel to indicate workflow start (currently disabled).  
  - Configuration: Message text includes workflow execution id; posts to #notifications channel; Slack OAuth2 credentials required.  
  - Inputs: Trigger from Schedule Trigger node.  
  - Outputs: Proceeds to Config node.  
  - Edge Cases: Slack API authentication failures, network issues could prevent notification.

- **Config**  
  - Type: Set  
  - Role: Defines GitHub repository parameters like owner, repo name, and folder path for storing backups.  
  - Configuration:  
    - repo_owner: (empty string, user must set)  
    - repo_name: "n8n-workflows"  
    - repo_path: "n8n-workflows"  
    - sub_path: "folder"  
  - Inputs: From Starting Message node.  
  - Outputs: Passes configuration to "Get Workflows" node.  
  - Edge Cases: Missing or incorrect repo_owner will cause GitHub API failures.

---

#### 2.2 Workflow Retrieval & Filtering

**Overview:**  
Retrieves all workflows from the n8n instance and filters workflows updated in the last 24 hours to limit backup scope.

**Nodes Involved:**  
- Get Workflows (n8n API)  
- Filter (Date filter)  
- Loop Over Items (splitInBatches)

**Node Details:**  

- **Get Workflows**  
  - Type: n8n (n8n API node)  
  - Role: Fetches all workflows from the n8n instance using internal API.  
  - Configuration: No filters applied, fetches all workflows.  
  - Credentials: Uses n8n API credentials.  
  - Inputs: Receives configuration from previous node.  
  - Outputs: Sends all workflows to Filter node.  
  - Edge Cases: API failures, auth errors, large volume may affect performance.

- **Filter**  
  - Type: Filter  
  - Role: Filters workflows to only those updated after or equal to current time minus 1 day (last 24 hours).  
  - Configuration: Condition: workflow's updatedAt >= now - 1 day.  
  - Inputs: From Get Workflows node.  
  - Outputs: Passes filtered workflows to Loop Over Items.  
  - Edge Cases: Workflows without updatedAt field or with malformed dates may be incorrectly filtered.

- **Loop Over Items**  
  - Type: splitInBatches  
  - Role: Processes workflows one by one or in batches to reduce memory usage and manage API requests.  
  - Configuration: Default batch size (likely 1, not explicitly stated).  
  - Inputs: Filtered workflows.  
  - Outputs: Sends each workflow item to "Completed Notification" and "Get a file" nodes.  
  - Edge Cases: Batch size misconfiguration may cause performance issues.

---

#### 2.3 Iterative Processing & Comparison

**Overview:**  
For each workflow, this block attempts to fetch the existing backup file from GitHub, checks file size, downloads full content if needed, merges data, and compares the existing backup with the current workflow JSON to detect changes or new workflows.

**Nodes Involved:**  
- Get a file (GitHub)  
- Is File too large? (If node)  
- Get File (HTTP Request)  
- Merge Items (Merge)  
- isDiffOrNew (Code)  
- Switch

**Node Details:**  

- **Get a file**  
  - Type: GitHub node (get file)  
  - Role: Attempts to retrieve the workflow backup file from GitHub repo.  
  - Configuration:  
    - Owner, repo, and file path dynamically set from Config and current workflow name.  
    - Operation: get  
  - Credentials: GitHub API OAuth2.  
  - Inputs: From Loop Over Items.  
  - Outputs: Branches to "Is File too large?" node.  
  - Edge Cases: File not found, API limits, auth errors. On error, continues regular output flow.

- **Is File too large?**  
  - Type: If node  
  - Role: Checks if the content field is empty and no error present, indicating the file is too large to be returned directly.  
  - Configuration: Checks if content is empty AND error not exists.  
  - Inputs: From "Get a file".  
  - Outputs:  
    - True: Calls "Get File" node to download via HTTP request.  
    - False: Forwards data to "Merge Items".  
  - Edge Cases: False negatives may cause unnecessary HTTP calls.

- **Get File**  
  - Type: HTTP Request  
  - Role: Downloads the file content from GitHub's download_url (raw file).  
  - Configuration: URL dynamically set from JSON property "download_url".  
  - Inputs: From "Is File too large?" (true branch).  
  - Outputs: To "Merge Items".  
  - Edge Cases: Network failures, URL changes, rate limits.

- **Merge Items**  
  - Type: Merge  
  - Role: Combines the GitHub file data with the current workflow JSON data for comparison.  
  - Configuration: Default merge (merge by index).  
  - Inputs: Two inputs: one from "Get File" or "Get a file", and one from Loop Over Items (current workflow JSON).  
  - Outputs: To "isDiffOrNew" code node.  
  - Edge Cases: Mismatched input counts may cause errors.

- **isDiffOrNew**  
  - Type: Code (JavaScript)  
  - Role: Compares the JSON content of the existing GitHub file and the current workflow export.  
  - Configuration:  
    - Orders JSON keys to normalize structure.  
    - Decodes base64 content if present.  
    - Sets github_status field to "same", "different", or "new" accordingly.  
    - Stores stringified new JSON if different or new.  
  - Inputs: From "Merge Items".  
  - Outputs: To "Switch" node.  
  - Edge Cases: Parsing errors if JSON malformed, base64 decoding errors.

- **Switch**  
  - Type: Switch  
  - Role: Routes based on comparison result (github_status).  
  - Configuration:  
    - Outputs:  
      - "Different" → Edit existing file node  
      - "New" → Create new file node  
      - "Same" → Return node (no action)  
  - Inputs: From "isDiffOrNew".  
  - Outputs: To respective nodes for file operations or workflow continuation.  
  - Edge Cases: Unexpected github_status values.

---

#### 2.4 GitHub File Management

**Overview:**  
This block either updates the existing file on GitHub if the workflow changed or creates a new file if it is new. If no change is detected, it returns control to continue with the next item.

**Nodes Involved:**  
- Edit existing file (GitHub)  
- Create new file (GitHub)  
- Return (Set node)

**Node Details:**  

- **Edit existing file**  
  - Type: GitHub node (edit file)  
  - Role: Updates the existing JSON file in the GitHub repo with new content.  
  - Configuration:  
    - Owner, repo, file path dynamically set.  
    - Operation: edit  
    - Commit message includes workflow name and "different" status.  
  - Credentials: GitHub API OAuth2.  
  - Inputs: From Switch (Different branch).  
  - Outputs: To Return node.  
  - Edge Cases: Conflicts if file changed in GitHub meanwhile, auth issues.

- **Create new file**  
  - Type: GitHub node (create file)  
  - Role: Creates a new JSON file in the GitHub repo for a new workflow.  
  - Configuration:  
    - Owner, repo, file path dynamically set.  
    - Commit message includes workflow name and "new" status.  
  - Credentials: GitHub API OAuth2.  
  - Inputs: From Switch (New branch).  
  - Outputs: To Return node.  
  - Edge Cases: File path conflicts, permission issues.

- **Return**  
  - Type: Set  
  - Role: Sets a boolean flag "Done" to true to signal completion of file processing for the current item.  
  - Configuration: Assigns "Done" = true (boolean).  
  - Inputs: From Edit existing file, Create new file, or Switch (Same branch).  
  - Outputs: Back to Loop Over Items (batch iterator).  
  - Edge Cases: None significant.

---

#### 2.5 Completion & Notification

**Overview:**  
After all workflows have been processed, this block sends an optional Slack notification summarizing the backup operation.

**Nodes Involved:**  
- Completed Notification (Slack)

**Node Details:**  

- **Completed Notification**  
  - Type: Slack node  
  - Role: Sends a message to Slack indicating how many workflows were processed in the backup run.  
  - Configuration:  
    - Text includes total count of workflows processed.  
    - Posts to #notifications channel.  
    - Slack OAuth2 credentials required.  
    - Set to execute once after processing all batches.  
  - Inputs: From Loop Over Items node (after all batches).  
  - Outputs: None (end of workflow).  
  - Edge Cases: Slack API failures, disabled in current config.

---

### 3. Summary Table

| Node Name            | Node Type               | Functional Role                                   | Input Node(s)                | Output Node(s)                    | Sticky Note                          |
|----------------------|-------------------------|-------------------------------------------------|------------------------------|----------------------------------|------------------------------------|
| Schedule Trigger      | scheduleTrigger         | Initiates workflow execution on schedule        | None                         | Starting Message                 |                                    |
| Starting Message      | Slack                   | Sends start notification to Slack (disabled)    | Schedule Trigger             | Config                          |                                    |
| Config               | Set                     | Defines GitHub repo parameters                    | Starting Message             | Get Workflows                   |                                    |
| Get Workflows         | n8n API                 | Fetches all current workflows                     | Config                       | Filter                         |                                    |
| Filter               | Filter                  | Filters workflows updated in last 24h             | Get Workflows                | Loop Over Items                 |                                    |
| Loop Over Items       | splitInBatches          | Iterates workflows one by one                      | Filter                       | Completed Notification, Get a file, Merge Items |                                    |
| Get a file           | GitHub get file         | Fetches backup file from GitHub                    | Loop Over Items              | Is File too large?              |                                    |
| Is File too large?    | If                      | Checks if GitHub file content is missing or large | Get a file                   | Get File (true), Merge Items (false) |                                    |
| Get File             | HTTP Request            | Downloads full file content via download_url       | Is File too large? (true)    | Merge Items                    |                                    |
| Merge Items           | Merge                   | Merges GitHub file data and current workflow data | Get File, Loop Over Items    | isDiffOrNew                   |                                    |
| isDiffOrNew          | Code                    | Compares GitHub and current workflow JSON          | Merge Items                  | Switch                        |                                    |
| Switch               | Switch                  | Routes based on comparison result                  | isDiffOrNew                  | Edit existing file, Create new file, Return |                                    |
| Edit existing file   | GitHub edit file        | Updates existing workflow file in GitHub           | Switch (Different)           | Return                        |                                    |
| Create new file      | GitHub create file      | Creates new workflow file in GitHub                 | Switch (New)                 | Return                        |                                    |
| Return               | Set                     | Marks processing done for current workflow item    | Edit existing file, Create new file, Switch (Same) | Loop Over Items               |                                    |
| Completed Notification| Slack                   | Sends completion notification to Slack             | Loop Over Items (final)      | None                          |                                    |
| Sticky Note3         | Sticky Note             | Provides GitHub folder link                        | None                         | None                          | # Links - [Github Folder](https://github.com/AndrewBoichenko/n8n-workflows/) |
| Sticky Note4         | Sticky Note             | Explains workflow functionality and usage          | None                         | None                          | # How it works \n This workflow will backup all instance workflows to GitHub every 24 hours.\nThe files are saved into folders using `repo_path` for the directory path and `ID.json` for the filename.\nThe Repo Owner, Repo Name and Main folder are set using the `Config` node in the subworkflow. \nThe workflow runs calls itself to help reduce memory usage, Once the workflow has completed it will send an optional notification to Slack.\nPlease check out my other items on [gumroad](https://boanse.gumroad.com/?section=k_Sn6LcT_dzJFnp5jmsM5A%3D%3D)\nYou might also like something else☺️ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: scheduleTrigger  
   - Configure interval to run every 24 hours (default).  

2. **Create Slack node "Starting Message":**  
   - Type: Slack  
   - Set channel to `#notifications`.  
   - Message text: `Information_source:  Starting Workflow Backup [{{ $execution.id }}]`  
   - Use Slack OAuth2 credentials.  
   - Connect Schedule Trigger → Starting Message.  
   - (Optional: Disable if not using Slack notification.)

3. **Create "Config" Set node:**  
   - Add fields:  
     - `repo_owner` (string) — set this to your GitHub username/org.  
     - `repo_name` (string) — default: `n8n-workflows`  
     - `repo_path` (string) — default: `n8n-workflows`  
     - `sub_path` (string) — default: `folder`  
   - Connect Starting Message → Config.

4. **Create "Get Workflows" n8n API node:**  
   - Operation: Get all workflows (no filters).  
   - Use n8n API credentials.  
   - Connect Config → Get Workflows.

5. **Create "Filter" node:**  
   - Condition: `updatedAt >= now - 1 day`.  
   - Connect Get Workflows → Filter.

6. **Create "Loop Over Items" node:**  
   - Type: splitInBatches, default batch size.  
   - Connect Filter → Loop Over Items.

7. **Create "Completed Notification" Slack node:**  
   - Channel: `#notifications`  
   - Message: `✅ Backup has completed - {{ $('Get Workflows').all().length }} workflows have been processed.`  
   - Use Slack OAuth2 credentials.  
   - Set to execute once after all batches.  
   - Connect Loop Over Items → Completed Notification.

8. **Create "Get a file" GitHub node:**  
   - Operation: get file  
   - Owner: `={{ $('Config').item.json.repo_owner }}`  
   - Repository: `={{ $('Config').item.json.repo_name }}`  
   - File Path: `={{ $('Config').item.json.sub_path }}/{{$('Loop Over Items').item.json.name}}.json`  
   - Use GitHub OAuth2 credentials.  
   - Connect Loop Over Items → Get a file.

9. **Create "Is File too large?" If node:**  
   - Condition: content field is empty AND error field does not exist.  
   - Connect Get a file → Is File too large?  
     - True branch → Get File node.  
     - False branch → Merge Items node.

10. **Create "Get File" HTTP Request node:**  
    - URL: `={{ $json.download_url }}` (from GitHub file metadata)  
    - Method: GET  
    - Connect Is File too large? (true) → Get File.

11. **Create "Merge Items" node:**  
    - Merge mode: merge by index (default).  
    - Connect:  
      - Get File (or Is File too large? false branch) → input 1  
      - Loop Over Items → input 2  
    - Output to isDiffOrNew node.

12. **Create "isDiffOrNew" Code node:**  
    - JavaScript code to:  
      - Normalize JSON keys order.  
      - Decode base64 GitHub content if present.  
      - Compare existing GitHub workflow JSON with current workflow JSON.  
      - Set `github_status` to "same", "different", or "new".  
      - Store formatted JSON string for changed/new workflows.  
    - Input: Merge Items.  
    - Output: Switch node.

13. **Create "Switch" node:**  
    - Rule based on `github_status` field:  
      - "different" → Edit existing file node  
      - "new" → Create new file node  
      - "same" → Return node  
    - Connect isDiffOrNew → Switch.

14. **Create "Edit existing file" GitHub node:**  
    - Operation: edit file  
    - Owner, repo, file path: same dynamic expressions as "Get a file".  
    - File content: `={{$('isDiffOrNew').item.json["n8n_data_stringy"]}}`  
    - Commit message: `={{$('Loop Over Items').item.json.name}} ({{$json.github_status}})`  
    - Use GitHub OAuth2 credentials.  
    - Connect Switch ("different") → Edit existing file → Return node.

15. **Create "Create new file" GitHub node:**  
    - Operation: create file  
    - Owner, repo, file path: same dynamic expressions.  
    - File content: `={{$('isDiffOrNew').item.json["n8n_data_stringy"]}}`  
    - Commit message: `={{$('Loop Over Items').item.json.name}} ({{$json.github_status}})`  
    - Use GitHub OAuth2 credentials.  
    - Connect Switch ("new") → Create new file → Return node.

16. **Create "Return" Set node:**  
    - Assign: field "Done" = true (boolean).  
    - Connect Switch ("same"), Edit existing file, Create new file → Return.  
    - Connect Return → Loop Over Items (to continue batch processing).

17. **Add Sticky Notes (optional):**  
    - Add a note with link to your GitHub repository.  
    - Add a descriptive note explaining the workflow logic and usage.

18. **Credential Setup:**  
    - GitHub OAuth2 credentials with repo access.  
    - Slack OAuth2 credentials with post message permissions (optional).  
    - n8n API credentials for internal workflow access.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow backs up all n8n instance workflows daily to GitHub, organizing files by workflow ID inside a configurable folder path.                                                                                             | General workflow description                                                                         |
| The workflow fetches current workflows, compares them with GitHub backups, and only commits changes if differences exist.                                                                                                          | Efficiency and GitHub API usage                                                                       |
| Optional Slack notifications can be enabled to signal start and completion of the backup process.                                                                                                                                 | Slack integration                                                                                     |
| The workflow uses batch processing and self-calling patterns to optimize memory and API rate limit usage.                                                                                                                        | Performance optimization                                                                              |
| For more workflows and automation tools, visit the author's Gumroad page: https://boanse.gumroad.com/?section=k_Sn6LcT_dzJFnp5jmsM5A%3D%3D                                                                                      | External resource                                                                                     |
| GitHub repository link for backups: https://github.com/AndrewBoichenko/n8n-workflows/                                                                                                                                              | GitHub repo link                                                                                      |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.*