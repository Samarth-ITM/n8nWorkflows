Automated Google Drive to FTP Transfer with JSON Logging & Reports

https://n8nworkflows.xyz/workflows/automated-google-drive-to-ftp-transfer-with-json-logging---reports-8072


# Automated Google Drive to FTP Transfer with JSON Logging & Reports

### 1. Workflow Overview

This workflow automates the transfer of files from Google Drive to an FTP server while maintaining detailed JSON logs and generating summary reports. Its key use cases include scheduled or on-demand batch file transfers with validation based on file size and type, error handling, and comprehensive status reporting via email. The workflow is structured into clear logical blocks:

- **1.1 Input Reception:** Workflow activation via scheduled trigger or webhook for manual initiation.
- **1.2 File Retrieval and Validation:** Searching Google Drive for files and filtering valid candidates according to size and extension criteria.
- **1.3 File Processing Loop:** Sequentially downloading each valid file and uploading it to the FTP server.
- **1.4 Transfer Logging and Error Handling:** Updating transfer notes with success or failure metadata on each file processed.
- **1.5 Continuation Check and Completion:** Determining if more files remain to process or proceeding to save logs.
- **1.6 Output Generation:** Saving JSON logs back to Google Drive and sending a detailed report email summarizing the transfer session.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow either on a fixed schedule (every 6 hours) or manually via a webhook POST request.
- **Nodes Involved:** `Schedule Trigger`, `Webhook Trigger`, `Sticky Note`

| Node Name       | Details                                                                                                                         |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Type: Schedule trigger node. Configured to run every 6 hours automatically. Outputs trigger signal to begin file retrieval.      |
| Webhook Trigger | Type: Webhook node. Listens for POST requests at `/webhook-transfer-status` to allow manual or external-triggered runs.           |
| Sticky Note     | Provides user instructions on how to trigger the workflow manually via the webhook endpoint.                                     |

**Inputs & Outputs:** Both triggers output to the `Get Drive Files` node.

**Edge Cases:**  
- Trigger failures are rare but could occur due to server downtime or webhook misconfiguration.

---

#### 1.2 File Retrieval and Validation

- **Overview:** Retrieves files from Google Drive and filters them based on size and allowed extensions, preparing a transfer notes object for logging.
- **Nodes Involved:** `Get Drive Files`, `Filter & Validate Files`

| Node Name          | Details                                                                                                                                                                                                                 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Get Drive Files     | Type: Google Drive node. Performs a search operation to list files in Google Drive. No explicit filters; retrieves all files for validation downstream.                                                                  |
| Filter & Validate Files | Type: Code node. Implements JavaScript logic to: <br> - Initialize transfer metadata and settings (max 50MB, allowed file extensions). <br> - Validate each file's extension and size. <br> - Segregate valid files and record skipped files with reasons in transfer notes. <br> - Outputs an object containing `transferNotes`, `validFiles`, and file counts. |

**Edge Cases:**  
- Large files or unsupported extensions are skipped and logged.  
- Parsing file sizes or missing metadata could cause errors if Drive API returns unexpected data.

---

#### 1.3 File Processing Loop

- **Overview:** Processes each valid file sequentially by downloading from Google Drive and uploading to FTP.
- **Nodes Involved:** `Process One by One`, `Download from Drive`, `Upload to FTP`

| Node Name         | Details                                                                                                                                                                    |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Process One by One | SplitInBatches node. Processes the list of valid files one by one to handle sequential file transfer.                                                                      |
| Download from Drive| Google Drive node. Downloads the current file using file ID from `validFiles` batch index.                                                                                  |
| Upload to FTP      | FTP node. Uploads the downloaded file to the FTP server under `/remote/directory/` preserving the original filename.                                                        |

**Edge Cases:**  
- Download failures due to file access permissions or network issues.  
- FTP upload failures due to authentication, connectivity, or path errors.  
- Batch processing stops on error but error handling updates logs accordingly.

---

#### 1.4 Transfer Logging and Error Handling

- **Overview:** Updates the transfer notes with success or failure details after each file processed.
- **Nodes Involved:** `Update Notes - Success`, `Update Notes - Error`

| Node Name           | Details                                                                                                                                                                                                                                                                                                                                                          |
|---------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Update Notes - Success | Code node updating the transfer notes on successful FTP upload: adds a record with timestamp, file details, status “success”, and FTP path. Increments successful transfer counter and updates last updated timestamp.                                                                                                                                          |
| Update Notes - Error   | Code node triggered on errors during download or upload: adds a record with error message and status “failed”, increments failure counter, and updates last updated timestamp. Utilizes the last error input from the previous node.                                                                                                                                 |

**Edge Cases:**  
- Errors during transfer propagate here for logging.  
- Errors in code execution could cause incomplete logs if expressions fail.

---

#### 1.5 Continuation Check and Completion

- **Overview:** Checks if more files remain to process; loops back if yes or proceeds to save logs if done.
- **Nodes Involved:** `Check if More Files`

| Node Name           | Details                                                                                                                                                                         |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Check if More Files  | If node evaluating if current batch index is less than total valid files minus one. If true, loops back to `Process One by One`, else proceeds to `Save Notes JSON`.             |

**Edge Cases:**  
- Incorrect batch index management could cause infinite loops or premature termination.

---

#### 1.6 Output Generation and Reporting

- **Overview:** Saves the final transfer notes JSON file to disk and uploads it to Google Drive; then sends a summary report email.
- **Nodes Involved:** `Save Notes JSON`, `Upload Notes to Drive`, `Create Final Report`, `Send Report Email`

| Node Name          | Details                                                                                                                                                                                                                                          |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Save Notes JSON    | Writes the accumulated transfer notes to a JSON file named with the current date, e.g., `transfer_notes_YYYY-MM-DD.json`.                                                                                                                        |
| Upload Notes to Drive | Uploads the saved JSON file back to Google Drive root folder with the same date-based filename.                                                                                                                                                  |
| Create Final Report | Code node creating a summary report object including total files, success/failure/skipped counts, calculated success rate, and arrays of successes and failures with timestamps and details.                                                     |
| Send Report Email  | Email node configured to send a report email titled “Google Drive to FTP File Transfer - Report” with the summary data. Uses an associated webhook ID for email sending (possibly SMTP or third-party email service).                              |

**Edge Cases:**  
- File write permissions or disk space issues could fail JSON saving.  
- Google Drive upload requires valid credentials and permissions.  
- Email sending may fail due to SMTP/authentication issues.  
- Timezone differences could affect timestamp accuracy in reports.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                    | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                      |
|---------------------|-----------------------|----------------------------------|-----------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note         | Sticky Note            | Provides webhook trigger info    |                       |                          | 🔗 **WEBHOOK TRIGGER** Manual trigger endpoint: POST to: `/webhook-transfer-status`. Use for on-demand transfers or external integrations. |
| Schedule Trigger    | Schedule Trigger       | Scheduled workflow starter       |                       | Get Drive Files          |                                                                                                 |
| Webhook Trigger     | Webhook                | Manual trigger endpoint          |                       | Get Drive Files          |                                                                                                 |
| Get Drive Files     | Google Drive           | Retrieve files from Drive        | Schedule Trigger, Webhook Trigger | Filter & Validate Files |                                                                                                 |
| Filter & Validate Files | Code                  | Filter valid files & initialize transfer notes | Get Drive Files         | Process One by One        |                                                                                                 |
| Process One by One  | SplitInBatches         | Sequential file processing loop  | Filter & Validate Files | Download from Drive, Create Final Report |                                                                                                 |
| Download from Drive | Google Drive           | Download current file            | Process One by One     | Upload to FTP            |                                                                                                 |
| Upload to FTP       | FTP                    | Upload file to FTP server        | Download from Drive    | Update Notes - Success   |                                                                                                 |
| Update Notes - Success | Code                  | Log successful transfer          | Upload to FTP          | Check if More Files      |                                                                                                 |
| Update Notes - Error | Code                  | Log failed transfer              | Error from Download or Upload nodes | Check if More Files      |                                                                                                 |
| Check if More Files | If                     | Loop control for remaining files | Update Notes - Success, Update Notes - Error | Process One by One, Save Notes JSON |                                                                                                 |
| Save Notes JSON     | Write Binary File      | Save transfer notes JSON locally | Check if More Files    | Upload Notes to Drive    |                                                                                                 |
| Upload Notes to Drive | Google Drive           | Upload JSON log to Drive         | Save Notes JSON        | Send Report Email        |                                                                                                 |
| Create Final Report | Code                   | Generate summary report JSON     | Process One by One     | Save Notes JSON          |                                                                                                 |
| Send Report Email   | Email Send             | Email summary report             | Upload Notes to Drive  |                          |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Nodes:**
   - Add a **Schedule Trigger** node, set interval to every 6 hours.
   - Add a **Webhook Trigger** node:
     - HTTP Method: POST
     - Path: `/webhook-transfer-status`
   - Connect both triggers to the next node (`Get Drive Files`).

2. **Retrieve Files from Google Drive:**
   - Add a **Google Drive** node named `Get Drive Files`.
   - Operation: Search (default to list all files).
   - Connect both triggers to this node.

3. **Filter and Validate Files:**
   - Add a **Code** node named `Filter & Validate Files`.
   - Paste the provided JavaScript code which:
     - Initializes transfer metadata and settings.
     - Filters files by allowed extensions (`.pdf`, `.doc`, `.docx`, `.txt`, `.jpg`, `.png`, `.zip`, `.xlsx`).
     - Filters files larger than 50MB.
     - Records skipped files with reasons.
   - Connect `Get Drive Files` output to this node.

4. **Split Processing into Batches:**
   - Add a **SplitInBatches** node named `Process One by One`.
   - No special parameters needed.
   - Connect `Filter & Validate Files` output to this node.

5. **Download Files from Google Drive:**
   - Add a **Google Drive** node named `Download from Drive`.
   - Operation: Download.
   - File ID: Use expression `{{$json.validFiles[$json.batchIndex].id}}`.
   - Connect `Process One by One` output to this node.

6. **Upload Files to FTP:**
   - Add an **FTP** node named `Upload to FTP`.
   - Path: `/remote/directory/{{$json.validFiles[$json.batchIndex].name}}`.
   - Configure FTP credentials (host, username, password, port as needed).
   - Connect `Download from Drive` output to this node.

7. **Update Transfer Notes on Success:**
   - Add a **Code** node named `Update Notes - Success`.
   - Paste provided JS code that appends success metadata to transfer notes.
   - Connect `Upload to FTP` success output to this node.

8. **Update Transfer Notes on Error:**
   - Add a **Code** node named `Update Notes - Error`.
   - Paste provided JS code that appends failure metadata including error message.
   - Connect the error output from `Download from Drive` and `Upload to FTP` to this node.

9. **Check if More Files Remain:**
   - Add an **If** node named `Check if More Files`.
   - Condition: Expression `{{$json.batchIndex < ($json.validFiles.length - 1)}}` equals `true`.
   - Connect `Update Notes - Success` and `Update Notes - Error` to this node.

10. **Loop or Save Logs:**
    - If true, connect back to `Process One by One`.
    - If false, proceed to `Save Notes JSON`.

11. **Save Transfer Notes JSON Locally:**
    - Add a **Write Binary File** node named `Save Notes JSON`.
    - File name: `transfer_notes_{{ new Date().toISOString().split('T')[0] }}.json`.
    - Connect false output of `Check if More Files` to this node.

12. **Upload JSON Log to Google Drive:**
    - Add a **Google Drive** node named `Upload Notes to Drive`.
    - Operation: Upload file with the same name as saved.
    - Folder: Root or specify as needed.
    - Connect `Save Notes JSON` to this node.

13. **Create Final Report:**
    - Add a **Code** node named `Create Final Report`.
    - Paste provided JS code that computes summary statistics and prepares arrays of successes and failures.
    - Connect `Process One by One` main output (success path) to this node.

14. **Send Report Email:**
    - Add an **Email Send** node named `Send Report Email`.
    - Configure SMTP or email credentials.
    - Subject: `Google Drive to FTP File Transfer - Report`.
    - Include JSON data from final report node in the email body or as attachment as desired.
    - Connect `Upload Notes to Drive` output to this node.

15. **Credential Setup:**
    - Configure Google Drive credentials with sufficient scopes to read and upload files.
    - Configure FTP credentials with write access to target directory.
    - Configure email credentials for sending reports.

16. **Test Triggers:**
    - Test both scheduled and webhook triggers to confirm end-to-end functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Manual trigger endpoint available at POST `/webhook-transfer-status` for on-demand or external system integration.                                          | Sticky Note node content in workflow                                                               |
| Workflow uses batch processing to avoid memory overload and handle large numbers of files sequentially.                                                     | Best practice in n8n for large datasets                                                            |
| Allowed file extensions and max file size are configurable in the `Filter & Validate Files` code node. Adjust as needed for different use cases.           | Customizable validation settings in code node                                                      |
| Google Drive API permissions must allow both file reading and uploading.                                                                                     | Credential setup requirement                                                                        |
| FTP credentials must be tested separately to ensure connectivity and permissions before running workflow.                                                   | Credential setup requirement                                                                        |
| Email node uses webhook ID for sending emails; verify email server credentials and ensure email quota to avoid delivery failures.                           | Email configuration best practices                                                                 |
| Workflow logs comprehensive metadata to JSON files saved both locally and uploaded to Google Drive for audit and historic tracking purposes.               | Workflow design principle                                                                           |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.