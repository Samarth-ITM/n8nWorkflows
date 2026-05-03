Automated Workflow Backup System with Google Drive, GitHub & Messaging Alerts

https://n8nworkflows.xyz/workflows/automated-workflow-backup-system-with-google-drive--github---messaging-alerts-8771


# Automated Workflow Backup System with Google Drive, GitHub & Messaging Alerts

### 1. Workflow Overview

This n8n workflow implements a comprehensive automated backup system for n8n workflows, targeting use cases where users want to secure their workflow definitions in two distinct storage solutions: Google Drive and GitHub. It also offers optional messaging notifications via Telegram and Discord to confirm backup completion.

The workflow runs either manually or on a schedule (every 12 hours) and performs the following logical blocks:

- **1.1 Trigger & Configuration Setup**: Accepts manual or scheduled trigger; sets configuration parameters including GitHub repo, Google Drive folder, backup timestamps, and messaging chat IDs.

- **1.2 Google Drive Backup Folder Creation**: Creates a timestamped folder in Google Drive as the container for all workflow backup files.

- **1.3 Fetching and Iterating Over Workflows**: Retrieves all workflows from the n8n instance and processes each workflow individually in batches.

- **1.4 Workflow Data Preparation and Storage**: Converts each workflow to a JSON file, uploads it to the Google Drive backup folder, and prepares data for GitHub operations with sanitized paths and ordered keys.

- **1.5 GitHub Backup Operations**: Checks if the workflow JSON file exists in the GitHub repo; if so, updates it; if not, creates a new file. This ensures version control with commit messages.

- **1.6 Finalization and Notifications**: After processing all workflows, consolidates results, marks backup completion, and sends status notifications to Telegram and Discord channels.

- **1.7 Scheduling**: A schedule trigger initiates the entire process every 12 hours automatically.

The workflow includes robust error handling (continue on fail for GitHub file checks) and uses best practices like sanitizing filenames and organizing backups by date in GitHub.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration Setup

**Overview:**  
Initiates the workflow manually or on a schedule. Then sets essential configuration variables such as GitHub repo details, Google Drive folder IDs, timestamps, and notification chat IDs.

**Nodes Involved:**  
- Manual Trigger  
- Schedule Every 12 Hours  
- Config  

**Node Details:**  
- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Allows manual start of the backup  
  - Config: No parameters, simple manual trigger  
  - Input: None  
  - Output: To Config node  
  - Edge Cases: None significant  

- **Schedule Every 12 Hours**  
  - Type: Schedule Trigger  
  - Role: Automatically trigger workflow every 12 hours  
  - Config: Interval set to 12 hours  
  - Input: None  
  - Output: To Config node  
  - Edge Cases: Scheduler misconfiguration or n8n downtime could delay backups  

- **Config**  
  - Type: Set node  
  - Role: Defines static and dynamic parameters for the workflow  
  - Config:  
    - `repo_owner` = "khmuhtadin" (GitHub username)  
    - `repo_name` = "n8n-backup" (GitHub repo)  
    - `repo_path` = "n8n-backup/" (folder path inside repo)  
    - `gdrive_folder_id` = Google Drive parent folder ID  
    - `backup_timestamp` = current timestamp formatted as `yyyy-MM-dd_HH-mm-ss`  
    - `backup_folder_name` = "wfBackup - <timestamp>" (for Google Drive folder)  
    - `telegram_chat_id` = numeric chat ID for Telegram notifications  
  - Input: From trigger nodes  
  - Output: To Create GDrive Folder node  
  - Edge Cases: Incorrect folder ID or missing credentials may cause errors  

---

#### 2.2 Google Drive Backup Folder Creation

**Overview:**  
Creates a new folder in Google Drive with a timestamped name to hold all workflow backup files for this backup run.

**Nodes Involved:**  
- Create GDrive Folder  

**Node Details:**  
- **Create GDrive Folder**  
  - Type: Google Drive node (Create folder)  
  - Role: Creates timestamped backup folder inside a specified Google Drive folder  
  - Config:  
    - Folder name from `backup_folder_name` variable  
    - Parent folder ID from `gdrive_folder_id`  
    - Drive selected as "My Drive"  
  - Credentials: Google Drive OAuth2  
  - Input: From Config node  
  - Output: To Get All Workflows node  
  - Edge Cases:  
    - OAuth token expiration or permission errors  
    - Invalid folder ID  
  - Version: n8n v0.153+ recommended for Drive node v3  

---

#### 2.3 Fetching and Iterating Over Workflows

**Overview:**  
Retrieves all workflows from the n8n instance via API and splits them into individual items for batch processing.

**Nodes Involved:**  
- Get All Workflows  
- Loop Over Workflows  

**Node Details:**  
- **Get All Workflows**  
  - Type: n8n node (API)  
  - Role: Fetches all workflows from n8n instance  
  - Config: No filters, retrieves all workflows  
  - Credentials: n8n API credentials with read access  
  - Input: From Create GDrive Folder node  
  - Output: To Loop Over Workflows node  
  - Edge Cases: API rate limits, auth failures, empty workflows list  

- **Loop Over Workflows**  
  - Type: SplitInBatches  
  - Role: Processes workflows one by one (batch size = 1 by default)  
  - Config: Default batch settings (likely batch size 1)  
  - Input: From Get All Workflows  
  - Output: Two outputs: one to Final Summary (execute once) and one to Convert to JSON File  
  - Edge Cases: Large number of workflows may slow processing  

---

#### 2.4 Workflow Data Preparation and Storage

**Overview:**  
Converts each workflow JSON data into a file, uploads it to Google Drive, then prepares the data structure and paths for GitHub backup.

**Nodes Involved:**  
- Convert to JSON File  
- Upload to Google Drive  
- Prepare GitHub Data  

**Node Details:**  
- **Convert to JSON File**  
  - Type: ConvertToFile  
  - Role: Converts workflow JSON object into a formatted JSON file with `.json` extension  
  - Config: Format JSON, filename from workflow name  
  - Input: From Loop Over Workflows  
  - Output: To Upload to Google Drive and Prepare GitHub Data nodes  
  - Edge Cases: Invalid JSON structure could cause conversion errors  

- **Upload to Google Drive**  
  - Type: Google Drive node (Upload file)  
  - Role: Uploads the converted JSON file to the newly created Google Drive folder  
  - Config:  
    - Filename from workflow name + `.json`  
    - Folder ID from Create GDrive Folder output  
  - Credentials: Google Drive OAuth2  
  - Input: From Convert to JSON File  
  - Output: To Merge GitHub Results node (one branch)  
  - Edge Cases: Upload failures due to API, permissions, or network issues  

- **Prepare GitHub Data**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Takes workflow data and config variables  
    - Sanitizes workflow name for safe filename (removes special chars, replaces spaces with underscores)  
    - Generates GitHub file path: `${repo_path}${year}/${month}/${sanitizedName}.json`  
    - Orders JSON keys alphabetically for consistency  
    - Outputs prepared JSON string and metadata for GitHub usage  
  - Input: From Convert to JSON File  
  - Output: To Check if File Exists node  
  - Edge Cases:  
    - Unexpected workflow names causing sanitization issues  
    - Date function errors (unlikely)  
  - Version: Compatible with n8n v0.153+  

---

#### 2.5 GitHub Backup Operations

**Overview:**  
Checks if each workflow backup JSON file exists in the GitHub repo. If it exists, updates the file with a new commit. Otherwise, creates a new file. Both operations include commit messages with workflow names and timestamps.

**Nodes Involved:**  
- Check if File Exists  
- File Exists? (If node)  
- Update GitHub File  
- Create GitHub File  
- Merge GitHub Results  

**Node Details:**  
- **Check if File Exists**  
  - Type: GitHub node  
  - Role: Attempts to retrieve file metadata from GitHub repo by path  
  - Config:  
    - Owner, repo, path from Config and Prepare GitHub Data outputs  
    - Authentication: OAuth2  
  - Credentials: GitHub OAuth2 token with repo access  
  - Input: From Prepare GitHub Data  
  - Output: To File Exists? node  
  - Edge Cases:  
    - File not found returns error but workflow continues (continueOnFail enabled)  
    - OAuth token expiry or permission issues  

- **File Exists?**  
  - Type: If node  
  - Role: Checks if file content exists in GitHub response to decide file update or creation  
  - Condition: `$json.content` exists (file found)  
  - Input: From Check if File Exists  
  - Output:  
    - True branch: Update GitHub File  
    - False branch: Create GitHub File  
  - Edge Cases: False positives if GitHub API response changes  

- **Update GitHub File**  
  - Type: GitHub node  
  - Role: Updates existing file content with new workflow JSON  
  - Config:  
    - Owner, repo, path from Config and Prepare GitHub Data  
    - File content: workflow JSON prepared earlier  
    - Commit message includes timestamp and workflow name  
    - Authentication: OAuth2  
  - Credentials: GitHub OAuth2  
  - Input: From File Exists? (true)  
  - Output: To Merge GitHub Results  
  - Edge Cases: Conflicts if repo changed externally, API rate limits  

- **Create GitHub File**  
  - Type: GitHub node  
  - Role: Creates a new file in GitHub repo with workflow JSON content  
  - Config: Same as Update GitHub File but operation is create  
  - Credentials: GitHub OAuth2  
  - Input: From File Exists? (false)  
  - Output: To Merge GitHub Results  
  - Edge Cases: Repo write permission issues, naming conflicts  

- **Merge GitHub Results**  
  - Type: Merge node  
  - Role: Merges update and create outputs to continue processing  
  - Input: From both Update and Create GitHub File nodes  
  - Output: To Mark Complete node  
  - Edge Cases: None critical  

---

#### 2.6 Finalization and Notifications

**Overview:**  
After all workflows have been processed, this block compiles a summary of the backup operation and sends notifications to Telegram and Discord channels.

**Nodes Involved:**  
- Mark Complete  
- Final Summary  
- Notify: Telegram  
- Notify: Discord  

**Node Details:**  
- **Mark Complete**  
  - Type: Set node  
  - Role: Adds flags and metadata marking the backup as complete for each workflow  
  - Assigns:  
    - `backup_complete`=true  
    - `workflow_name` and `workflow_id` from current workflow  
    - `gdrive_backup`=true  
    - `github_backup`=true  
  - Input: From Merge GitHub Results  
  - Output: Loops back to Loop Over Workflows (for next iteration)  
  - Edge Cases: None significant  

- **Final Summary**  
  - Type: Set node (execute once)  
  - Role: Compiles summary information after all workflows processed  
  - Assigns:  
    - `total_workflows` = number of workflows processed  
    - `backup_timestamp` = timestamp from Config  
    - `gdrive_folder` = created Google Drive folder name  
    - `status` = "Backup completed successfully"  
  - Input: From Loop Over Workflows (completion output)  
  - Output: To Notify nodes  
  - Edge Cases: Could run prematurely if batch processing incomplete  

- **Notify: Telegram**  
  - Type: Telegram node  
  - Role: Sends formatted HTML message with summary details to Telegram chat  
  - Config:  
    - Chat ID from Config  
    - Message contains backup status, workflow count, timestamp, drive folder, repo name  
    - Parse mode HTML, attribution false  
  - Credentials: Telegram Bot API token  
  - Input: From Final Summary  
  - Edge Cases: Invalid chat ID or token, message delivery failure  

- **Notify: Discord**  
  - Type: Discord node  
  - Role: Sends message to Discord channel summarizing backup  
  - Config:  
    - Guild and channel IDs hardcoded in node  
    - Message includes status, total workflows, timestamp, drive folder, repo name  
  - Credentials: Discord Bot API token  
  - Input: From Final Summary  
  - Edge Cases: Bot permissions, channel visibility, API rate limits  

---

#### 2.7 Scheduling

**Overview:**  
Enables automation by triggering the entire workflow every 12 hours without manual intervention.

**Nodes Involved:**  
- Schedule Every 12 Hours (also part of block 2.1)  

**Node Details:**  
- Already described in 2.1 Trigger block  

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                             | Input Node(s)              | Output Node(s)                     | Sticky Note                                                                                         |
|----------------------|---------------------|--------------------------------------------|----------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Manual Trigger       | Trigger             | Manual start of workflow                    | None                       | Config                           | See Sticky Note: "🚀 n8n Workflow Backup System" (overview, requirements, features)               |
| Schedule Every 12 Hours | Schedule Trigger  | Scheduled start every 12 hours              | None                       | Config                           | See Sticky Note: "🚀 n8n Workflow Backup System" (overview, requirements, features)               |
| Config               | Set                 | Define config variables (repo info, folder, timestamps, chat IDs) | Manual Trigger, Schedule Every 12 Hours | Create GDrive Folder              | See Sticky Note: "⚙️ Configuration" (how to edit config, process overview, backup structure)       |
| Create GDrive Folder | Google Drive        | Create timestamped backup folder in Drive   | Config                     | Get All Workflows                |                                                                                                   |
| Get All Workflows    | n8n API             | Fetch all workflows from n8n instance       | Create GDrive Folder        | Loop Over Workflows              |                                                                                                   |
| Loop Over Workflows  | SplitInBatches      | Process workflows one by one                 | Get All Workflows           | Final Summary, Convert to JSON File | See Sticky Note: "⚙️ Configuration" (workflow process details)                                     |
| Convert to JSON File | ConvertToFile       | Convert workflow JSON to formatted file     | Loop Over Workflows         | Upload to Google Drive, Prepare GitHub Data |                                                                                                   |
| Upload to Google Drive | Google Drive       | Upload JSON file to Google Drive folder      | Convert to JSON File        | Merge GitHub Results             |                                                                                                   |
| Prepare GitHub Data  | Code                | Sanitize name, prepare GitHub file path & content | Convert to JSON File        | Check if File Exists             |                                                                                                   |
| Check if File Exists | GitHub              | Check if workflow file exists in GitHub repo | Prepare GitHub Data         | File Exists?                    |                                                                                                   |
| File Exists?         | If                  | Determine if file exists (branching)         | Check if File Exists        | Update GitHub File (true), Create GitHub File (false) |                                                                                                   |
| Update GitHub File   | GitHub              | Update existing workflow file in GitHub      | File Exists? (true)         | Merge GitHub Results             |                                                                                                   |
| Create GitHub File   | GitHub              | Create new workflow file in GitHub            | File Exists? (false)        | Merge GitHub Results             |                                                                                                   |
| Merge GitHub Results | Merge               | Merge create/update results for workflow loop | Update GitHub File, Create GitHub File | Mark Complete                  |                                                                                                   |
| Mark Complete        | Set                 | Mark backup status flags for current workflow | Merge GitHub Results        | Loop Over Workflows              |                                                                                                   |
| Final Summary        | Set (execute once)  | Compile overall backup summary                 | Loop Over Workflows (completion) | Notify: Telegram, Notify: Discord |                                                                                                   |
| Notify: Telegram     | Telegram             | Send backup summary notification               | Final Summary              | None                           |                                                                                                   |
| Notify: Discord      | Discord              | Send backup summary notification               | Final Summary              | None                           |                                                                                                   |
| Sticky Note          | Sticky Note          | Overview, requirements, features              | None                       | None                           | "🚀 n8n Workflow Backup System - Requirements, Resource URLs, Features, etc."                    |
| Sticky Note1         | Sticky Note          | Configuration instructions and workflow process | None                       | None                           | "⚙️ Configuration - How to edit config, Process steps, Backup structures"                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node, no parameters.  
   - Add a **Schedule Trigger** node with interval set to every 12 hours.

2. **Add Config Node**  
   - Add a **Set** node named "Config".  
   - Define variables:  
     - `repo_owner` (string): your GitHub username (e.g., "khmuhtadin")  
     - `repo_name` (string): your GitHub repo name (e.g., "n8n-backup")  
     - `repo_path` (string): folder inside repo to store backups (e.g., "n8n-backup/")  
     - `gdrive_folder_id` (string): Google Drive folder ID for backups  
     - `backup_timestamp` (string): expression `={{ $now.format('yyyy-MM-dd_HH-mm-ss') }}`  
     - `backup_folder_name` (string): expression `=wfBackup - {{ $now.format('yyyy-MM-dd_HH-mm-ss') }}`  
     - `telegram_chat_id` (number): your Telegram chat ID (optional)  
   - Connect both trigger nodes to this node.

3. **Create Google Drive Folder Node**  
   - Add a Google Drive node "Create GDrive Folder" with operation to create a folder.  
   - Configure folder name as `={{ $json.backup_folder_name }}` and parent folder ID as `={{ $json.gdrive_folder_id }}`.  
   - Use Google Drive OAuth2 credentials.  
   - Connect Config node to this node.

4. **Fetch All Workflows**  
   - Add an n8n node "Get All Workflows" with no filters.  
   - Use n8n API credentials with read access.  
   - Connect "Create GDrive Folder" to this node.

5. **Split Workflows for Processing**  
   - Add a "SplitInBatches" node "Loop Over Workflows".  
   - Default batch size (1).  
   - Connect "Get All Workflows" to this node.

6. **Convert Workflow to JSON File**  
   - Add "ConvertToFile" node "Convert to JSON File".  
   - Operation: toJson, format enabled.  
   - Filename: `={{ $json.name }}.json`.  
   - Connect "Loop Over Workflows" to this node.

7. **Upload JSON to Google Drive**  
   - Add Google Drive node "Upload to Google Drive".  
   - Filename: `={{ $('Loop Over Workflows').item.json.name }}.json`.  
   - Folder ID: `={{ $('Create GDrive Folder').item.json.id }}`.  
   - Use Google Drive OAuth2 credentials.  
   - Connect "Convert to JSON File" to this node.

8. **Prepare GitHub Data**  
   - Add a Code node "Prepare GitHub Data".  
   - Paste the provided JavaScript code that sanitizes workflow names, builds GitHub paths, and orders JSON keys.  
   - Connect "Convert to JSON File" to this node.

9. **Check if GitHub File Exists**  
   - Add GitHub node "Check if File Exists".  
   - Operation: get file.  
   - Owner: `={{ $('Config').item.json.repo_owner }}`.  
   - File path: `={{ $('Prepare GitHub Data').item.json.github_path }}`.  
   - Repository: `={{ $('Config').item.json.repo_name }}`.  
   - Auth: OAuth2 with GitHub OAuth token.  
   - Enable "Continue On Fail" to handle file not found.  
   - Connect "Prepare GitHub Data" to this node.

10. **If Node to Branch by File Existence**  
    - Add "If" node "File Exists?"  
    - Condition: check if `$json.content` exists (file found).  
    - Connect "Check if File Exists" to this node.

11. **Update GitHub File**  
    - Add GitHub node "Update GitHub File".  
    - Operation: edit file.  
    - Owner, repo, path as in step 9.  
    - File content: `={{ $('Prepare GitHub Data').item.json.workflow_json }}`.  
    - Commit message: `=Update: {{ $('Loop Over Workflows').item.json.name }} - {{ $('Config').item.json.backup_timestamp }}`  
    - Auth: OAuth2.  
    - Connect "File Exists?" true output to this node.

12. **Create GitHub File**  
    - Add GitHub node "Create GitHub File".  
    - Operation: create file.  
    - Owner, repo, path as above.  
    - File content and commit message similar to update.  
    - Auth: OAuth2.  
    - Connect "File Exists?" false output to this node.

13. **Merge GitHub Results**  
    - Add a "Merge" node "Merge GitHub Results".  
    - Connect outputs from both "Update GitHub File" and "Create GitHub File" to this node.

14. **Mark Backup Complete**  
    - Add a "Set" node "Mark Complete".  
    - Assign:  
      - `backup_complete`=true  
      - `workflow_name` and `workflow_id` from current workflow  
      - `gdrive_backup`=true  
      - `github_backup`=true  
    - Connect "Merge GitHub Results" to this node.

15. **Loop Back to Process Next Workflow**  
    - Connect "Mark Complete" node output back to "Loop Over Workflows" node input (to process next batch).

16. **Final Summary Node**  
    - Add a "Set" node "Final Summary".  
    - Execute once only.  
    - Assign:  
      - `total_workflows` = `={{ $('Get All Workflows').all().length }}`  
      - `backup_timestamp` = from Config node  
      - `gdrive_folder` = from Create GDrive Folder node  
      - `status` = "Backup completed successfully"  
    - Connect "Loop Over Workflows" second output (completion) to this node.

17. **Notification Nodes**  
    - Add Telegram node "Notify: Telegram".  
      - Chat ID: from Config node  
      - Message: formatted HTML with backup details  
      - Use Telegram Bot API credentials.  
    - Add Discord node "Notify: Discord".  
      - Guild and Channel IDs configured.  
      - Message includes same summary details as Telegram.  
      - Use Discord Bot API credentials.  
    - Connect "Final Summary" to both notification nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| 🚀 n8n Workflow Backup System: features dual backup (Google Drive + GitHub), sanitized filenames, error handling, optional notifications, runs twice daily. | See sticky note node "Sticky Note" in workflow JSON |
| ⚙️ Configuration instructions: edit Config node for GitHub repo info, Google Drive folder ID, Telegram chat ID etc. Backup structure explained. | See sticky note node "Sticky Note1" in workflow JSON |
| n8n API docs: [https://docs.n8n.io/api](https://docs.n8n.io/api) | For n8n API node credential setup and usage |
| Google Drive API console: [https://console.cloud.google.com/](https://console.cloud.google.com/) | For creating OAuth2 credentials for Google Drive |
| GitHub OAuth Apps: [https://github.com/settings/developers](https://github.com/settings/developers) | To create GitHub OAuth tokens with repo access |
| Telegram Bot setup via BotFather: [https://t.me/botfather](https://t.me/botfather) | For Telegram notification bot token |
| Discord Developer Portal: [https://discord.com/developers/applications](https://discord.com/developers/applications) | For creating Discord bot and obtaining tokens |

---

This comprehensive analysis and reproduction guide enables understanding, modification, and full rebuilding of the Automated Workflow Backup System using n8n, Google Drive, GitHub, and messaging alerts.