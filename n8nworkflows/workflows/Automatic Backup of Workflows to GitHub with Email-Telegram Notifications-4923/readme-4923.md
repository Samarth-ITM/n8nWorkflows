Automatic Backup of Workflows to GitHub with Email/Telegram Notifications

https://n8nworkflows.xyz/workflows/automatic-backup-of-workflows-to-github-with-email-telegram-notifications-4923


# Automatic Backup of Workflows to GitHub with Email/Telegram Notifications

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows into a GitHub repository, providing version control and secure storage. It is designed to run on demand or on a daily schedule, creating a complete snapshot of workflows, uploading them as individual files to a GitHub repo, and sending notifications upon completion via Gmail and Telegram.  

The workflow is structured into the following logical blocks:

- **1.1 Backup Initialization**: Triggering the backup and loading configuration.
- **1.2 Workflow Retrieval and Preparation**: Fetching all workflows from n8n, cleaning and packaging them for upload.
- **1.3 GitHub Repository Management**: Checking if the target GitHub repo exists, creating it if necessary.
- **1.4 Uploading Backup Files**: Splitting workflows into files, uploading them and supporting manifest and README files.
- **1.5 Completion and Notifications**: Summarizing the backup process and sending notifications via email and Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Backup Initialization

- **Overview:**  
  This block listens for manual or scheduled triggers to start the backup process and initializes the configuration parameters.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`  
  - `Daily`  
  - `Backup_Config`  

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the backup workflow for testing or on-demand runs.  
    - Configuration: Default manual trigger without parameters.  
    - Inputs: None  
    - Outputs: Connects to `Backup_Config`.  
    - Edge Cases: User must manually trigger; no automatic error conditions.  

  - **Daily**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily.  
    - Configuration: Default daily schedule (time not specified in JSON).  
    - Inputs: None  
    - Outputs: Separate branch, not connected in this JSON snapshot but intended for scheduled backups.  
    - Edge Cases: Schedule misconfiguration could prevent automatic runs.  

  - **Backup_Config**  
    - Type: Code  
    - Role: Initializes and provides configuration data such as GitHub repo name, authentication tokens, and backup parameters.  
    - Configuration: Custom JavaScript code (not shown in JSON) setting variables and credentials.  
    - Inputs: From manual trigger.  
    - Outputs: Connects to `Get_Workflows`.  
    - Edge Cases: Misconfigured credentials or parameters can cause downstream failures.  

---

#### 2.2 Workflow Retrieval and Preparation

- **Overview:**  
  This block fetches all current workflows from the n8n instance, cleans the data to remove unnecessary info, and prepares a backup package.

- **Nodes Involved:**  
  - `Get_Workflows`  
  - `Clean_Workflows`  
  - `Create_Backup_Package`  

- **Node Details:**  

  - **Get_Workflows**  
    - Type: HTTP Request  
    - Role: Calls the n8n API endpoint to retrieve all workflows.  
    - Configuration: HTTP GET request to `/workflows` endpoint with appropriate authentication (likely via credentials).  
    - Inputs: From `Backup_Config`.  
    - Outputs: Connects to `Clean_Workflows`.  
    - Edge Cases: API errors, auth failures, network timeouts.  
    - Version: Uses n8n API v4.2 HTTP Request node.  

  - **Clean_Workflows**  
    - Type: Code  
    - Role: Cleans and filters the raw workflow data to remove sensitive or unnecessary fields before backup.  
    - Configuration: JavaScript code that processes items to keep only relevant data for backup.  
    - Inputs: From `Get_Workflows`.  
    - Outputs: Connects to `Create_Backup_Package`.  
    - Edge Cases: Possible expression or script errors if unexpected data structure changes occur.  

  - **Create_Backup_Package**  
    - Type: Code  
    - Role: Packages cleaned workflows into a single structure, possibly generating metadata or backup manifest data.  
    - Configuration: JavaScript code assembling final backup data.  
    - Inputs: From `Clean_Workflows`.  
    - Outputs: Connects to `Check_Repository_Exists`.  
    - Edge Cases: Logical errors in packaging, large data handling issues.  

---

#### 2.3 GitHub Repository Management

- **Overview:**  
  This block verifies if the GitHub repository for backups exists; if not, it creates the repository to prepare for file uploads.

- **Nodes Involved:**  
  - `Check_Repository_Exists`  
  - `If_Repo_Not_Exists`  
  - `Create_Repository`  

- **Node Details:**  

  - **Check_Repository_Exists**  
    - Type: HTTP Request  
    - Role: Checks GitHub API to see if the backup repository exists.  
    - Configuration: HTTP GET request to GitHub repository endpoint, using GitHub OAuth or token credentials.  
    - Inputs: From `Create_Backup_Package`.  
    - Outputs: Connects to `If_Repo_Not_Exists`.  
    - Edge Cases: GitHub API rate limits, auth errors, network issues.  
    - OnError: Configured to continue regular output to handle repo absence gracefully.  

  - **If_Repo_Not_Exists**  
    - Type: If Condition  
    - Role: Branches workflow based on whether repository exists.  
    - Configuration: Condition evaluates HTTP response status or content for repo existence.  
    - Inputs: From `Check_Repository_Exists`.  
    - Outputs:  
      - True (repo does not exist) → `Create_Repository`  
      - False (repo exists) → `Upload_Manifest` (skips repo creation)  
    - Edge Cases: Incorrect condition evaluation could cause redundant repo creation or skipped uploads.  

  - **Create_Repository**  
    - Type: HTTP Request  
    - Role: Creates a new GitHub repository for backups if missing.  
    - Configuration: HTTP POST request to GitHub API with repo creation parameters (name, visibility).  
    - Inputs: From `If_Repo_Not_Exists`.  
    - Outputs: Connects to `Upload_Manifest`.  
    - Edge Cases: Creation errors due to permissions or name conflicts.  

---

#### 2.4 Uploading Backup Files

- **Overview:**  
  This block prepares individual workflow files, uploads them to the GitHub repository, and uploads supporting files such as README and manifest.

- **Nodes Involved:**  
  - `Upload_Manifest`  
  - `Upload_README`  
  - `Split_Workflow_Files`  
  - `Loop_Workflow_Uploads`  
  - `Upload_Workflow_Files`  

- **Node Details:**  

  - **Upload_Manifest**  
    - Type: HTTP Request  
    - Role: Uploads a manifest file describing the backup contents to GitHub.  
    - Configuration: HTTP PUT/POST to GitHub contents API with manifest JSON file content.  
    - Inputs: From `Create_Repository` or `If_Repo_Not_Exists` false branch.  
    - Outputs: Connects to `Upload_README`.  
    - Edge Cases: API failure, file encoding issues.  

  - **Upload_README**  
    - Type: HTTP Request  
    - Role: Uploads a README file explaining the backup contents and instructions.  
    - Configuration: HTTP PUT/POST to GitHub contents API with README content.  
    - Inputs: From `Upload_Manifest`.  
    - Outputs: Connects to `Split_Workflow_Files`.  
    - Edge Cases: Similar to `Upload_Manifest`.  

  - **Split_Workflow_Files**  
    - Type: Code  
    - Role: Splits the backup package into individual workflow files for upload.  
    - Configuration: JavaScript code that creates an array of workflow files (JSON per workflow).  
    - Inputs: From `Upload_README`.  
    - Outputs: Connects to `Loop_Workflow_Uploads`.  
    - Edge Cases: Data processing errors, large payloads.  

  - **Loop_Workflow_Uploads**  
    - Type: Split In Batches  
    - Role: Iteratively processes each workflow file for upload to avoid API rate limits and large requests.  
    - Configuration: Batch size likely set to 1 for sequential uploads.  
    - Inputs: From `Split_Workflow_Files` and from `Upload_Workflow_Files` (loop).  
    - Outputs:  
      - Main output: Connects to `Backup_Complete_Summary` (after all uploads)  
      - Secondary output: Connects to `Upload_Workflow_Files` (to upload current batch).  
    - Edge Cases: Batch failures, partial uploads.  

  - **Upload_Workflow_Files**  
    - Type: HTTP Request  
    - Role: Uploads individual workflow JSON files to GitHub repo.  
    - Configuration: HTTP PUT/POST to GitHub contents API, per workflow JSON file.  
    - Inputs: From `Loop_Workflow_Uploads`.  
    - Outputs: Connects back to `Loop_Workflow_Uploads` to continue batch processing.  
    - Edge Cases: API rate limiting, authorization failures.  

---

#### 2.5 Completion and Notifications

- **Overview:**  
  After successful uploads, this block compiles a summary of the backup and sends notifications via Gmail and Telegram.

- **Nodes Involved:**  
  - `Backup_Complete_Summary`  
  - `Gmail - Notification`  
  - `Send_Backup_Notification`  

- **Node Details:**  

  - **Backup_Complete_Summary**  
    - Type: Code  
    - Role: Compiles a summary message including backup status, number of workflows backed up, timestamps, and any errors.  
    - Configuration: JavaScript code formatting the summary text.  
    - Inputs: From `Loop_Workflow_Uploads`.  
    - Outputs: Connects to `Gmail - Notification`.  
    - Edge Cases: Missing or incomplete data could cause malformed messages.  

  - **Gmail - Notification**  
    - Type: Gmail  
    - Role: Sends an email notification of backup completion.  
    - Configuration: Uses configured Gmail OAuth2 credentials; email content from summary node.  
    - Inputs: From `Backup_Complete_Summary`.  
    - Outputs: None (end of email notification chain).  
    - Edge Cases: Auth token expiration, email sending errors.  

  - **Send_Backup_Notification**  
    - Type: Telegram  
    - Role: Sends a Telegram message notification for backup completion.  
    - Configuration: Telegram bot credentials configured; message content not explicitly shown but likely mirrors email.  
    - Inputs: Independent, triggered separately or could be connected in extended logic.  
    - Outputs: None.  
    - Edge Cases: Bot token issues, Telegram API limits.  

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                       | Input Node(s)                    | Output Node(s)                 | Sticky Note                 |
|-------------------------|------------------------|------------------------------------|---------------------------------|-------------------------------|-----------------------------|
| When clicking ‘Test workflow’ | Manual Trigger         | Manual start of backup              | None                            | Backup_Config                  |                             |
| Daily                   | Schedule Trigger       | Daily automatic start               | None                            | None (not connected in JSON)  |                             |
| Backup_Config           | Code                   | Initialize configuration            | When clicking ‘Test workflow’   | Get_Workflows                 |                             |
| Get_Workflows           | HTTP Request           | Retrieve workflows from n8n         | Backup_Config                   | Clean_Workflows               |                             |
| Clean_Workflows         | Code                   | Clean and filter workflow data      | Get_Workflows                   | Create_Backup_Package         |                             |
| Create_Backup_Package   | Code                   | Package workflows for backup        | Clean_Workflows                 | Check_Repository_Exists       |                             |
| Check_Repository_Exists | HTTP Request           | Check if GitHub repo exists         | Create_Backup_Package           | If_Repo_Not_Exists            |                             |
| If_Repo_Not_Exists      | If Condition           | Branch on repo existence             | Check_Repository_Exists         | Create_Repository (true), Upload_Manifest (false) |                             |
| Create_Repository       | HTTP Request           | Create GitHub repo                  | If_Repo_Not_Exists (true)       | Upload_Manifest              |                             |
| Upload_Manifest         | HTTP Request           | Upload manifest file to GitHub      | Create_Repository / If_Repo_Not_Exists (false) | Upload_README               |                             |
| Upload_README           | HTTP Request           | Upload README file                  | Upload_Manifest                | Split_Workflow_Files          |                             |
| Split_Workflow_Files    | Code                   | Split backup package into files     | Upload_README                  | Loop_Workflow_Uploads         |                             |
| Loop_Workflow_Uploads   | Split In Batches       | Batch process file uploads          | Split_Workflow_Files, Upload_Workflow_Files | Backup_Complete_Summary (end), Upload_Workflow_Files (loop) |                             |
| Upload_Workflow_Files   | HTTP Request           | Upload individual workflow files    | Loop_Workflow_Uploads           | Loop_Workflow_Uploads          |                             |
| Backup_Complete_Summary | Code                   | Create backup summary message        | Loop_Workflow_Uploads           | Gmail - Notification          |                             |
| Gmail - Notification    | Gmail                  | Send email notification              | Backup_Complete_Summary         | None                         |                             |
| Send_Backup_Notification| Telegram               | Send Telegram notification           | None (triggered independently) | None                         |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger (no parameters)  

2. **Create Schedule Trigger Node**  
   - Name: `Daily`  
   - Type: Schedule Trigger  
   - Set schedule to daily at desired time  

3. **Create Code Node `Backup_Config`**  
   - Connect `When clicking ‘Test workflow’` to `Backup_Config`  
   - Include script to initialize:  
     - GitHub repo name, owner, and token  
     - n8n API URL and credentials  
     - Notification email and Telegram chat IDs/bot tokens  

4. **Create HTTP Request Node `Get_Workflows`**  
   - Connect `Backup_Config` to `Get_Workflows`  
   - Method: GET  
   - URL: `{{ $credentials.n8nApi.url }}/workflows` (or equivalent)  
   - Authentication: Use n8n credentials or API token  

5. **Create Code Node `Clean_Workflows`**  
   - Connect `Get_Workflows` to `Clean_Workflows`  
   - Script to remove unnecessary metadata from workflows, retain essential fields  

6. **Create Code Node `Create_Backup_Package`**  
   - Connect `Clean_Workflows` to `Create_Backup_Package`  
   - Script to assemble workflows into a backup data structure, add timestamp and metadata  

7. **Create HTTP Request Node `Check_Repository_Exists`**  
   - Connect `Create_Backup_Package` to `Check_Repository_Exists`  
   - Method: GET  
   - URL: `https://api.github.com/repos/{{ owner }}/{{ repo }}`  
   - Authentication: GitHub OAuth token  

8. **Create If Node `If_Repo_Not_Exists`**  
   - Connect `Check_Repository_Exists` to `If_Repo_Not_Exists`  
   - Condition: HTTP status code = 404 (repo does not exist)  

9. **Create HTTP Request Node `Create_Repository`**  
   - Connect true branch of `If_Repo_Not_Exists` to `Create_Repository`  
   - Method: POST  
   - URL: `https://api.github.com/user/repos`  
   - Body: JSON with repo name and settings  
   - Authentication: GitHub OAuth token  

10. **Create HTTP Request Node `Upload_Manifest`**  
    - Connect `Create_Repository` (true branch) and false branch of `If_Repo_Not_Exists` to `Upload_Manifest`  
    - Method: PUT  
    - URL: `https://api.github.com/repos/{{ owner }}/{{ repo }}/contents/manifest.json`  
    - Body: Base64-encoded manifest JSON  
    - Authentication: GitHub OAuth token  

11. **Create HTTP Request Node `Upload_README`**  
    - Connect `Upload_Manifest` to `Upload_README`  
    - Method: PUT  
    - URL: `https://api.github.com/repos/{{ owner }}/{{ repo }}/contents/README.md`  
    - Body: Base64-encoded README markdown content  
    - Authentication: GitHub OAuth token  

12. **Create Code Node `Split_Workflow_Files`**  
    - Connect `Upload_README` to `Split_Workflow_Files`  
    - Script to split backup package into array of individual workflow JSON files with filenames  

13. **Create SplitInBatches Node `Loop_Workflow_Uploads`**  
    - Connect `Split_Workflow_Files` to `Loop_Workflow_Uploads`  
    - Batch size: 1  

14. **Create HTTP Request Node `Upload_Workflow_Files`**  
    - Connect `Loop_Workflow_Uploads` secondary output to `Upload_Workflow_Files`  
    - Method: PUT  
    - URL: `https://api.github.com/repos/{{ owner }}/{{ repo }}/contents/{{workflowFileName}}`  
    - Body: Base64-encoded individual workflow JSON  
    - Authentication: GitHub OAuth token  
    - After upload, connect back to `Loop_Workflow_Uploads` main input to continue loop  

15. **Create Code Node `Backup_Complete_Summary`**  
    - Connect `Loop_Workflow_Uploads` main output (completion) to `Backup_Complete_Summary`  
    - Script to generate summary message with backup results  

16. **Create Gmail Node `Gmail - Notification`**  
    - Connect `Backup_Complete_Summary` to `Gmail - Notification`  
    - Configure with Gmail OAuth2 credentials  
    - Set email recipient and message body from summary  

17. **Create Telegram Node `Send_Backup_Notification`**  
    - Configure with Telegram bot credentials  
    - Prepare message similar to email notification (can be triggered separately or integrated in future)  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                   |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow tags include "tool agents" and "n8n management" indicating its role in infrastructure.    | n8n workflow metadata                             |
| The workflow is versioned as "n8n BackupButler v3" emphasizing iterative improvements.             | Workflow naming                                   |
| GitHub API usage requires OAuth tokens with repo creation and content write permissions.           | GitHub API docs: https://docs.github.com/en/rest |
| Notifications use Gmail OAuth2 and Telegram Bot API requiring credential setup in n8n.             | Gmail API: https://developers.google.com/gmail/api; Telegram Bot API: https://core.telegram.org/bots/api |
| Use of Split In Batches node is critical to avoid GitHub API rate limiting during file uploads.    | n8n docs on batching                              |
| Manual trigger node is useful for testing without waiting for scheduled runs.                       | n8n core nodes                                    |

---

**Disclaimer:** The provided documentation is based exclusively on an automated n8n workflow export. It respects all current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.