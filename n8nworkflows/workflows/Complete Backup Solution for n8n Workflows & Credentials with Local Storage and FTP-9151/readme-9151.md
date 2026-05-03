Complete Backup Solution for n8n Workflows & Credentials with Local Storage and FTP

https://n8nworkflows.xyz/workflows/complete-backup-solution-for-n8n-workflows---credentials-with-local-storage-and-ftp-9151


# Complete Backup Solution for n8n Workflows & Credentials with Local Storage and FTP

---

# Complete Backup Solution for n8n Workflows & Credentials with Local Storage and FTP

---

### 1. Workflow Overview

This workflow automates a comprehensive backup process for n8n workflows and credentials. It targets users who want reliable, scheduled backups of their n8n environment including workflows and credentials, with local filesystem storage and optional FTP upload for offsite storage. The workflow provides detailed logs, error handling, and notifications for backup monitoring.

The workflow consists of the following logical blocks:

- **1.1 Initialization**  
  Sets up time variables, environment and workflow-specific configuration, and prepares directory paths and filenames for backups and logs.

- **1.2 Workflow Backup Preparation**  
  Fetches all workflows, cleans their filenames for OS compatibility, converts them to JSON files, and writes them to disk.

- **1.3 Credentials Backup Preparation**  
  Exports credentials using n8n CLI, lists exported credential files, reads each file, and prepares them for FTP upload.

- **1.4 FTP Uploads**  
  Uploads workflow and credential backup files to configured FTP server locations, optionally disabled in this version for manual activation.

- **1.5 Aggregation and Summary Logging**  
  Aggregates backup results, processes FTP upload outcomes with detailed logs, generates backup summary statistics, writes backup logs, writes email logs, and sends notification emails with backup status.

- **1.6 Error Handling**  
  Stops workflow execution with descriptive error messages on critical failures such as filename cleaning or summary generation errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization

**Overview:**  
Initializes all variables related to time, paths, workflow configuration, and custom backup settings. Outputs structured data used globally throughout the workflow.

**Nodes Involved:**  
- Daily Backup (Schedule Trigger)  
- Init (Code)

**Node Details:**

- **Daily Backup**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 4:00 AM server time (configurable)  
  - Inputs: None  
  - Outputs: Init node  
  - Edge Cases: No major issues expected; time zone depends on server environment or overridden manually.

- **Init**  
  - Type: Code  
  - Role: Sets local timezone-aware timestamps, backup folder paths, filenames for logs and reports, and user-defined backup configurations such as FTP folder and credential backup folder.  
  - Configuration:  
    - Uses environment variables or default fallbacks for admin email, project directories, backup folders, FTP folder names, FTP server name (for logging only), and credential backup folder name.  
    - Defines naming conventions, max retries, timeout, and flags for including credentials or compression (future enhancement).  
  - Key variables: `timeData`, `workflowConfig`, `customConfig`  
  - Inputs: Trigger from Daily Backup  
  - Outputs: Creates structured JSON with all configuration and time data for downstream nodes  
  - Edge Cases: Wrong environment configurations or missing folders may cause downstream write errors.

---

#### 2.2 Workflow Backup Preparation

**Overview:**  
Fetches all active workflows from n8n, cleans workflow filenames for safe cross-platform storage, converts workflows to JSON files, and writes each workflow file locally.

**Nodes Involved:**  
- Create Date Folder (Execute Command)  
- Fetch Workflows (n8n node)  
- Clean Filename (Code)  
- Convert to File (Convert to File)  
- Write Each Workflow To Disk (Read/Write File)

**Node Details:**

- **Create Date Folder**  
  - Type: Execute Command  
  - Role: Creates the date-named backup folder inside the configured backup folder (e.g., `/files/n8n-backups/2024-06-15`) using `mkdir -p`.  
  - Inputs: Init output  
  - Outputs: Fetch Workflows & Export Credentials  
  - Edge Cases: Folder creation fails if permissions or parent folder missing.

- **Fetch Workflows**  
  - Type: n8n-nodes-base.n8n  
  - Role: Retrieves all workflows from the n8n instance without filters.  
  - Inputs: Create Date Folder output  
  - Outputs: Clean Filename  
  - Edge Cases: If API or node fails, no workflows fetched.

- **Clean Filename**  
  - Type: Code  
  - Role: Sanitizes workflow names removing invalid characters for filesystems, limits length, and provides fallback names.  
  - Input from Fetch Workflows (uses ‘name’ field)  
  - Outputs: Convert to File on success; stops with error on failure  
  - Error Handling: Configured to continue on error output for failures.  
  - Edge Cases: Missing workflow names or unexpected characters may cause errors.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts each workflow JSON into a file binary content named after the cleaned filename.  
  - Input: Clean Filename output  
  - Outputs: Upload Workflows To FTP (disabled), Write Each Workflow To Disk  
  - Edge Cases: Conversion failure if input data malformed.

- **Write Each Workflow To Disk**  
  - Type: Read/Write File  
  - Role: Writes each workflow JSON file to the local backup folder with date prefix and cleaned filename.  
  - Input: Convert to File output  
  - Outputs: Aggregate node for further processing  
  - Edge Cases: Disk write permission or space issues.

---

#### 2.3 Credentials Backup Preparation

**Overview:**  
Exports credentials via n8n CLI, lists exported credential files, reads each credential file for further processing.

**Nodes Involved:**  
- Export Credentials (Execute Command)  
- List Credential Files (Execute Command)  
- Output Credential Items (Code)  
- Read/Write Files from Disk (Read/Write File)

**Node Details:**

- **Export Credentials**  
  - Type: Execute Command  
  - Role: Runs `n8n export:credentials --backup` CLI command to export all credentials to the backup folder under credentials subfolder.  
  - Input: Create Date Folder output  
  - Outputs: List Credential Files  
  - Edge Cases: CLI command failure, permission issues, or export errors.

- **List Credential Files**  
  - Type: Execute Command  
  - Role: Lists all JSON files in the credentials backup folder using `ls` command, with fallback message if none found.  
  - Input: Export Credentials output  
  - Outputs: Output Credential Items  
  - Edge Cases: No files found or command failure.

- **Output Credential Items**  
  - Type: Code  
  - Role: Parses file list from command output, creates an item per credential file with metadata including local path and FTP target path.  
  - Input: List Credential Files output  
  - Outputs: Read/Write Files from Disk  
  - Edge Cases: Empty or malformed file list, export result errors.

- **Read/Write Files from Disk**  
  - Type: Read/Write File  
  - Role: Reads each credential file content from disk to prepare for FTP upload.  
  - Input: Output Credential Items  
  - Outputs: Upload Credentials To FTP (disabled)  
  - Edge Cases: File read permission issues or missing files.

---

#### 2.4 FTP Uploads

**Overview:**  
Uploads workflow and credential backup files to FTP server paths configured in the workflow. FTP nodes are disabled by default but structured for activation.

**Nodes Involved:**  
- Upload Workflows To FTP (FTP) [disabled]  
- Upload Credentials To FTP (FTP) [disabled]  
- FTP Logger (workflows) (Code) [disabled]  
- FTP Logger (credentials) (Code) [disabled]  
- Merge (Merge data from FTP loggers and summary)

**Node Details:**

- **Upload Workflows To FTP**  
  - Type: FTP  
  - Role: Uploads each workflow JSON file to FTP path based on date and cleaned filename.  
  - Inputs: Convert to File (for each workflow)  
  - Outputs: FTP Logger (workflows)  
  - Disabled by default - requires FTP credential configuration.  
  - Edge Cases: FTP connection errors, authentication failures, timeouts.

- **Upload Credentials To FTP**  
  - Type: FTP  
  - Role: Uploads each credential JSON file to FTP path under credentials folder with date prefix.  
  - Inputs: Read/Write Files from Disk output (credentials)  
  - Outputs: FTP Logger (credentials)  
  - Disabled by default.  
  - Edge Cases: Same as above.

- **FTP Logger (workflows)**  
  - Type: Code  
  - Role: Processes FTP upload results for workflow files, logs successes and failures, detects connection errors, and prepares summary for merging.  
  - Inputs: Upload Workflows To FTP output  
  - Outputs: Merge node  
  - Edge Cases: Partial uploads, connection timeouts.

- **FTP Logger (credentials)**  
  - Type: Code  
  - Role: Same as above but for credential files.  
  - Inputs: Upload Credentials To FTP output  
  - Outputs: Merge node  
  - Edge Cases: Same as above.

- **Merge**  
  - Type: Merge  
  - Role: Aggregates results from FTP loggers and workflow aggregation node to prepare data for final summary.  
  - Inputs: FTP Logger (workflows), FTP Logger (credentials), Aggregate (workflows)  
  - Outputs: Backup Summary node

---

#### 2.5 Aggregation and Summary Logging

**Overview:**  
Aggregates all workflow files, merges FTP upload logs, generates a comprehensive backup summary including statistics and error reports. Writes logs to disk and sends notification email with logs.

**Nodes Involved:**  
- Aggregate (Aggregate all workflow files into one structure)  
- Backup Summary (Code)  
- Write Backup Log (Read/Write File)  
- Write Email Log (Read/Write File)  
- Send email (Email Send)

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all workflow JSON files into a single array with binary data for backup summary processing.  
  - Inputs: Write Each Workflow To Disk outputs  
  - Outputs: Merge node

- **Backup Summary**  
  - Type: Code  
  - Role:  
    - Analyzes all backup data including workflows, credentials, FTP upload results.  
    - Calculates statistics: total workflows, successful/failed writes, file sizes, FTP upload success, errors.  
    - Generates console log lines and email log content.  
    - Converts logs to binary for writing files.  
    - Defines workflow completion status.  
  - Inputs: Merge node output  
  - Outputs: Write Backup Log and Write Email Log nodes (on success), or stops with error on failure.  
  - Edge Cases: Possible errors in aggregating data or missing expected inputs.

- **Write Backup Log**  
  - Type: Read/Write File  
  - Role: Writes detailed backup summary JSON log file to disk.  
  - Inputs: Backup Summary output  
  - Outputs: Send email node  
  - Edge Cases: Disk write failures.

- **Write Email Log**  
  - Type: Read/Write File  
  - Role: Writes plain text email log file to disk.  
  - Inputs: Backup Summary output  
  - Outputs: No further node except Send email (concurrent with Write Backup Log)  
  - Edge Cases: Disk write failures.

- **Send email**  
  - Type: Email Send  
  - Role: Sends notification email to admin email with backup status and attaches workflow JSON files.  
  - Inputs: Write Backup Log output  
  - Configuration:  
    - To: Admin email from environment variable or default  
    - From: admin@example.com  
    - Subject: Backup success notification with workflow name  
    - Body: Includes email log text with backup summary  
  - Edge Cases: SMTP configuration errors, email delivery failures.

---

#### 2.6 Error Handling

**Overview:**  
Stops workflow execution upon critical errors during filename cleaning or backup summary generation, providing meaningful error messages.

**Nodes Involved:**  
- ERROR: Clean Filename (Stop and Error)  
- ERROR: Backup Summary (Stop and Error)

**Node Details:**

- **ERROR: Clean Filename**  
  - Type: Stop and Error  
  - Role: Stops workflow if filename cleaning fails, outputs error message from code node.  
  - Inputs: Clean Filename error output  
  - Edge Cases: Ensures no partial or corrupt workflow files proceed.

- **ERROR: Backup Summary**  
  - Type: Stop and Error  
  - Role: Stops workflow if backup summary code node fails, outputs error message.  
  - Inputs: Backup Summary error output  
  - Edge Cases: Prevents incomplete or misleading backup logs.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                                              | Input Node(s)                  | Output Node(s)                                 | Sticky Note                                                                                              |
|-------------------------|-----------------------|--------------------------------------------------------------|-------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Daily Backup            | Schedule Trigger      | Triggers workflow daily at 4:00 AM                            | —                             | Init                                          | ## Initialisation                                                                                       |
| Init                    | Code                  | Initializes variables, paths, config, and timestamps          | Daily Backup                  | Create Date Folder                             | ## Initialisation                                                                                       |
| Create Date Folder       | Execute Command       | Creates backup folder with date prefix                         | Init                         | Fetch Workflows, Export Credentials            | ## Initialisation                                                                                       |
| Fetch Workflows          | n8n                   | Fetches all workflows from n8n instance                        | Create Date Folder            | Clean Filename                                | ## Workflows’ backup & FTP upload                                                                       |
| Clean Filename           | Code                  | Cleans workflow filenames for cross-platform compatibility    | Fetch Workflows               | Convert to File, ERROR: Clean Filename         | ## Workflows’ backup & FTP upload                                                                       |
| Convert to File          | Convert to File       | Converts workflows JSON to file binary data                    | Clean Filename               | Upload Workflows To FTP (disabled), Write Each Workflow To Disk | ## Workflows’ backup & FTP upload                                                                       |
| Write Each Workflow To Disk | Read/Write File    | Writes each workflow JSON file to local backup folder         | Convert to File              | Aggregate                                     | ## Workflows’ backup & FTP upload                                                                       |
| Aggregate               | Aggregate             | Aggregates all workflow files into single data structure      | Write Each Workflow To Disk  | Merge                                         | ## Workflows’ backup & FTP upload                                                                       |
| Export Credentials       | Execute Command       | Exports credentials via n8n CLI                                | Create Date Folder           | List Credential Files                         | ## Credentials’ backup & FTP upload                                                                     |
| List Credential Files    | Execute Command       | Lists all JSON credential files in backup folder              | Export Credentials           | Output Credential Items                        | ## Credentials’ backup & FTP upload                                                                     |
| Output Credential Items  | Code                  | Parses file list, prepares credential files with metadata     | List Credential Files        | Read/Write Files from Disk                     | ## Credentials’ backup & FTP upload                                                                     |
| Read/Write Files from Disk | Read/Write File     | Reads each credential JSON file                                | Output Credential Items      | Upload Credentials To FTP (disabled)          | ## Credentials’ backup & FTP upload                                                                     |
| Upload Workflows To FTP  | FTP                   | Uploads workflow backup files to FTP server                    | Convert to File              | FTP Logger (workflows) (disabled)             | ## Workflows’ backup & FTP upload                                                                       |
| Upload Credentials To FTP | FTP                  | Uploads credential backup files to FTP server                  | Read/Write Files from Disk   | FTP Logger (credentials) (disabled)           | ## Credentials’ backup & FTP upload                                                                     |
| FTP Logger (workflows)   | Code                  | Logs workflow FTP upload results, summarizes successes/failures | Upload Workflows To FTP     | Merge                                         | ## Workflows’ backup & FTP upload                                                                       |
| FTP Logger (credentials) | Code                  | Logs credentials FTP upload results                            | Upload Credentials To FTP    | Merge                                         | ## Credentials’ backup & FTP upload                                                                     |
| Merge                   | Merge                 | Merges FTP logs and aggregated workflows for final summary    | FTP Logger (workflows), FTP Logger (credentials), Aggregate | Backup Summary                              | ## Finalisation                                                                                        |
| Backup Summary           | Code                  | Generates backup summary, statistics, logs; prepares email content | Merge                        | Write Backup Log, Write Email Log, ERROR: Backup Summary | ## Finalisation                                                                                        |
| Write Backup Log         | Read/Write File       | Writes detailed JSON backup log to disk                        | Backup Summary              | Send email                                    | ## Finalisation                                                                                        |
| Write Email Log          | Read/Write File       | Writes plain text email log to disk                            | Backup Summary              | —                                             | ## Finalisation                                                                                        |
| Send email               | Email Send            | Sends email notification with backup status and attachments   | Write Backup Log            | —                                             | ## Finalisation                                                                                        |
| ERROR: Clean Filename    | Stop and Error        | Stops if cleaning workflow filenames fails                     | Clean Filename (error)       | —                                             |                                                                                                        |
| ERROR: Backup Summary    | Stop and Error        | Stops if backup summary generation fails                       | Backup Summary (error)       | —                                             |                                                                                                        |
| Sticky Note              | Sticky Note           | Provides configuration and informational comments              | —                           | —                                             | ## Backup Worflows & Credentials to Drive - Configuration instructions                                |
| Sticky Note2             | Sticky Note           | Label for Initialization block                                 | —                           | —                                             | ## Initialisation                                                                                       |
| Sticky Note3             | Sticky Note           | Label for Credentials’ backup & FTP upload block              | —                           | —                                             | ## Credentials’ backup & FTP upload                                                                     |
| Sticky Note5             | Sticky Note           | Label for Workflows’ backup & FTP upload block                | —                           | —                                             | ## Workflows’ backup & FTP upload                                                                       |
| Sticky Note6             | Sticky Note           | Label for Finalisation block                                   | —                           | —                                             | ## Finalisation                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node** named "Daily Backup"  
   - Set to trigger daily at 4:00 AM (hour 4).  
   - Connect its output to the "Init" node.

2. **Create a Code Node** named "Init"  
   - Insert JavaScript code to:  
     - Define local timezone-aware timestamps and formatted date/time strings.  
     - Read environment variables or use defaults for admin email, project directories, backup folder paths, FTP folders, FTP server name, and credential backup folder.  
     - Prepare file names and paths for logs and reports.  
     - Set backup configuration parameters such as retries, timeout, file naming, and flags for credentials export and compression.  
   - Output a single JSON object with `timeData`, `workflowConfig`, and `customConfig`.  
   - Connect output to "Create Date Folder".

3. **Create an Execute Command Node** named "Create Date Folder"  
   - Command: `mkdir -p "{{ $json.customConfig.backupFolder }}/{{ $json.customConfig.datePrefix }}"`  
   - Connect output to both "Fetch Workflows" and "Export Credentials".

4. **Create an n8n Node** named "Fetch Workflows"  
   - Configure to fetch all workflows without filters.  
   - Connect output to "Clean Filename".

5. **Create a Code Node** named "Clean Filename"  
   - Implement JS code to sanitize workflow filenames from the "Fetch Workflows" node, replacing invalid characters, trimming length, and providing fallbacks.  
   - Configure node to:  
     - Continue on error output.  
     - On success, connect to "Convert to File".  
     - On error, connect to "ERROR: Clean Filename".

6. **Create a Convert to File Node** named "Convert to File"  
   - Mode: Each  
   - Operation: to JSON  
   - File name: `={{ $json.name }}`  
   - Binary Property Name: `data`  
   - Connect output to both "Upload Workflows To FTP" (disabled by default) and "Write Each Workflow To Disk".

7. **Create a Read/Write File Node** named "Write Each Workflow To Disk"  
   - Operation: Write  
   - File name: `={{ $('Init').first().json.customConfig.backupFolder }}/{{ $('Init').first().json.customConfig.datePrefix }}/{{ $('Clean Filename').all()[$itemIndex].json.cleanedFileName }}.json`  
   - Append: false  
   - Connect output to "Aggregate".

8. **Create an Aggregate Node** named "Aggregate"  
   - Aggregate all item data including binaries into a field named `workflows`.  
   - Connect output to "Merge" input 3.

9. **Create an Execute Command Node** named "Export Credentials"  
   - Command:  
     ```bash
     n8n export:credentials --backup --output={{ $('Init').item.json.customConfig.backupFolder }}/{{ $('Init').item.json.customConfig.datePrefix }}/{{ $('Init').item.json.customConfig.credentials }}
     ```  
   - Connect output to "List Credential Files".

10. **Create an Execute Command Node** named "List Credential Files"  
    - Command:  
      ```bash
      ls -1 "{{ $('Init').first().json.customConfig.backupFolder }}/{{ $('Init').first().json.customConfig.datePrefix }}/{{ $('Init').first().json.customConfig.credentials }}"/*.json 2>/dev/null || echo "No JSON files found"
      ```  
    - Connect output to "Output Credential Items".

11. **Create a Code Node** named "Output Credential Items"  
    - Parse the command output to create individual items per credential file, enrich with local and FTP paths.  
    - Configure to continue on error output.  
    - Connect output to "Read/Write Files from Disk".

12. **Create a Read/Write File Node** named "Read/Write Files from Disk"  
    - Operation: Read  
    - File selector: `={{ $json.localPath }}`  
    - Connect output to "Upload Credentials To FTP" (disabled by default).

13. **Create FTP Nodes:**  
    - "Upload Workflows To FTP" (disabled by default)  
      - Operation: upload  
      - Path: `={{ $('Init').first().json.customConfig.FTP_BACKUP_FOLDER }}/{{ $('Init').first().json.customConfig.datePrefix }}/{{ $('Clean Filename').all()[$itemIndex].json.cleanedFileName }}.json`  
      - Connect output to "FTP Logger (workflows)".  
    - "Upload Credentials To FTP" (disabled by default)  
      - Operation: upload  
      - Path: `={{ $('Output Credential Items').item.json.ftpPath }}`  
      - Connect output to "FTP Logger (credentials)".

14. **Create Code Nodes for FTP Logs:**  
    - "FTP Logger (workflows)"  
      - Processes FTP upload results for workflows, logs success/failure, detects connection errors.  
      - Connect output to "Merge" input 1.  
    - "FTP Logger (credentials)"  
      - Processes FTP upload results for credentials similarly.  
      - Connect output to "Merge" input 2.

15. **Create a Merge Node** named "Merge"  
    - Number of inputs: 3  
    - Merge mode: Merge by index or combine inputs  
    - Inputs from: FTP Logger (workflows), FTP Logger (credentials), Aggregate  
    - Output to "Backup Summary".

16. **Create a Code Node** named "Backup Summary"  
    - Processes all merged data: workflow files, credential export results, FTP upload summaries.  
    - Generates detailed summary statistics, console logs, email logs, and binary data for log files.  
    - Configure to continue on error output.  
    - On success, output to "Write Backup Log" and "Write Email Log".  
    - On error, output to "ERROR: Backup Summary".

17. **Create Read/Write File Nodes:**  
    - "Write Backup Log"  
      - Write operation with filename from summary data.  
      - Connect output to "Send email".  
    - "Write Email Log"  
      - Write operation for plain text email log file.  
      - No further connections.

18. **Create an Email Send Node** named "Send email"  
    - To: Admin email from environment variable or default.  
    - From: `admin <admin@example.com>`  
    - Subject: `n8n SUCCESS: {{ $workflow.name }}`  
    - Text: Includes workflow name and email log text.  
    - Attachments: binary workflow JSON files.  
    - Connect from "Write Backup Log".

19. **Create Stop and Error Nodes:**  
    - "ERROR: Clean Filename"  
      - Connected from Clean Filename error output.  
      - Stops workflow with error message.  
    - "ERROR: Backup Summary"  
      - Connected from Backup Summary error output.  
      - Stops workflow with error message.

20. **Add Sticky Notes** to document configuration sections and major blocks for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Configure environment variables for `N8N_ADMIN_EMAIL`, `N8N_PROJECTS_DIR`, `N8N_BACKUP_FOLDER`, and `N8N_FTP_BACKUP_FOLDER`    | Essential for correct folder paths and admin notifications                                     |
| FTP nodes are disabled by default; configure FTP credentials and enable nodes for actual FTP upload                           | Ensure FTP credentials are properly set in n8n credentials manager                             |
| The workflow respects local timezone settings and can be customized for any timezone by editing the Init node’s code          | See Init node for timezone examples and customization instructions                              |
| Backup filenames are cleaned to avoid filesystem issues across Windows/Linux/Unix systems                                     | Ensures portability and avoids errors                                                          |
| Detailed logs and email notifications provide actionable feedback on success/failure and error details                        | Helps monitor backup health and troubleshoot issues                                           |
| For security, credential exports are stored locally before upload; sensitive data handling is configured in Init node         | Avoids leaking credentials unnecessarily                                                      |
| Commands used (mkdir, ls, n8n CLI) assume the n8n server has shell access and permissions                                     | Confirm n8n server environment supports these commands                                        |
| This workflow uses n8n version features such as Code node v2, Email node v2.1, and error handling with continueErrorOutput    | Requires n8n version 0.154 or later for some features                                         |
| For more info on n8n FTP node usage and credentials export CLI: https://docs.n8n.io/nodes/n8n-nodes-base.ftp/ and https://docs.n8n.io/cli/commands/ | Official n8n documentation                                                                        |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---