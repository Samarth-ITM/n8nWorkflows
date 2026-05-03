Automated FTP File Migration with Smart Cleanup and Email Notifications

https://n8nworkflows.xyz/workflows/automated-ftp-file-migration-with-smart-cleanup-and-email-notifications-8161


# Automated FTP File Migration with Smart Cleanup and Email Notifications

---

### 1. Workflow Overview

This workflow automates secure, scheduled file migration between two FTP servers, ensuring filtered file transfers with post-transfer cleanup and success notification emails. It is designed for daily execution, transferring only specific file types and sizes, with robust error handling, logging, and operational transparency.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 2 AM.
- **1.2 Source File Listing:** Connects to the source FTP server to list files recursively.
- **1.3 File Filtering:** Filters listed files by extension and optionally by other criteria.
- **1.4 File Download:** Downloads each filtered file from the source FTP server.
- **1.5 File Upload:** Uploads downloaded files to the destination FTP server.
- **1.6 Upload Verification:** Checks if the upload succeeded.
- **1.7 Source Cleanup:** Deletes the source file after successful upload.
- **1.8 Success Logging:** Records success details for each file migration.
- **1.9 Email Notification:** Sends a success notification email after each successful transfer.

Supplementary nodes provide documentation and guidance on configuration, troubleshooting, performance, security, and file processing strategies to support administrators and operators.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the workflow automatically every day at 2 AM.
- **Nodes Involved:**  
  - Daily Schedule (2 AM)

- **Node Details:**  
  - **Daily Schedule (2 AM)**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression set to `0 2 * * *` (daily at 2:00 AM server time, Europe/Warsaw timezone)  
    - Input: None (trigger node)  
    - Output: Initiates the workflow  
    - Failure Modes: Cron misconfiguration, time zone mismatch, server downtime  
    - Notes: Includes detailed reminders about testing schedules and timezone effects.

#### 1.2 Source File Listing

- **Overview:** Lists all files in the source FTP directory (and subdirectories) to identify candidates for migration.
- **Nodes Involved:**  
  - List Files - Source FTP

- **Node Details:**  
  - **List Files - Source FTP**  
    - Type: FTP node  
    - Operation: List files recursively in `/source/directory/`  
    - Credentials: Uses dedicated Source FTP credentials with host, port, username, password  
    - Output: Metadata array of files including name, size, date, permissions  
    - Failure Modes: Connection timeout, permission denied, path not found, authentication failure  
    - Notes: Strong recommendations for security best practices in credentials and connection.

#### 1.3 File Filtering

- **Overview:** Filters listed files to select only those matching specified extensions for transfer.
- **Nodes Involved:**  
  - Filter Files

- **Node Details:**  
  - **Filter Files**  
    - Type: Filter node  
    - Condition: Regex match on filename extension with pattern `\.(txt|csv|json|xml)$` (case-sensitive)  
    - Input: List of files from previous node  
    - Output: Filtered list of files matching criteria  
    - Failure Modes: Expression evaluation errors, regex misconfiguration, empty input sets  
    - Notes: Node can be removed to transfer all files; includes suggestions for alternative regexes and size/date filters.

#### 1.4 File Download

- **Overview:** Downloads each filtered file from the source FTP server into n8n temporary memory for processing.
- **Nodes Involved:**  
  - Download File

- **Node Details:**  
  - **Download File**  
    - Type: FTP node  
    - Operation: Download file  
    - Dynamic Path: Uses `{{ $json.name }}` to specify which file to download  
    - Binary Property: Stores file content in `file_data`  
    - Timeout: 300 seconds (5 minutes) default, adjustable for larger files  
    - Retry: Auto-retry 3 times on failure with exponential backoff  
    - Input: Filtered file metadata  
    - Output: File binary data plus metadata  
    - Failure Modes: Network interruptions, file locked, permission denied, timeout, memory overflow for large files  
    - Notes: Advises monitoring server RAM, file size limits, and retry strategy.

#### 1.5 File Upload

- **Overview:** Uploads the downloaded file binary to the destination FTP server, preserving filename and optionally directory structure.
- **Nodes Involved:**  
  - Upload to Destination

- **Node Details:**  
  - **Upload to Destination**  
    - Type: FTP node  
    - Operation: Upload file  
    - Dynamic Destination Path: `/destination/directory/{{ $json.name }}`  
    - Binary Property: Uses `file_data` from download node  
    - Options: Auto-create missing directories, preserve timestamps, prevent overwrites  
    - Credentials: Separate Destination FTP credentials (different from source)  
    - Input: File binary data + metadata  
    - Output: Upload result with success status  
    - Failure Modes: Disk full, permission denied, connection lost, filename conflicts  
    - Notes: Includes recommendations for security, performance optimizations, and verification strategies.

#### 1.6 Upload Verification

- **Overview:** Validates if the upload succeeded by checking the success flag from the upload node.
- **Nodes Involved:**  
  - Upload Success?

- **Node Details:**  
  - **Upload Success?**  
    - Type: Filter node  
    - Condition: Checks if `success` property equals `true`  
    - Input: Upload node output  
    - Output: True branch proceeds to source cleanup; false branch implicitly for error handling (not detailed here)  
    - Failure Modes: Incorrect property evaluation, missing success flag  
    - Notes: Validation can be extended with size or checksum comparisons.

#### 1.7 Source Cleanup

- **Overview:** Deletes the source file from the original FTP server after a successful upload to avoid duplicate transfers.
- **Nodes Involved:**  
  - Delete Source File

- **Node Details:**  
  - **Delete Source File**  
    - Type: FTP node  
    - Operation: Delete file  
    - Dynamic Path: `{{ $json.name }}` from prior nodes  
    - Input: Files validated as successfully uploaded  
    - Output: Confirmation of deletion  
    - Failure Modes: Permission denied, file locked, connection issues  
    - Notes: Advises disabling this node during testing, alternative strategies (move, rename, archive), and risk mitigation.

#### 1.8 Success Logging

- **Overview:** Logs details of the successful file migration for monitoring and auditing.
- **Nodes Involved:**  
  - Log Success

- **Node Details:**  
  - **Log Success**  
    - Type: Set node  
    - Assigns fields: timestamp, filename, status ("SUCCESS"), file size, operation type ("MIGRATE_AND_DELETE")  
    - Input: File metadata after successful deletion  
    - Output: Structured log data for downstream consumption  
    - Failure Modes: Expression evaluation errors  
    - Notes: Suggestions to enhance logging include database or file-based logging, external API calls, and enriched metadata.

#### 1.9 Email Notification

- **Overview:** Sends an email alert confirming successful file migration to designated recipients.
- **Nodes Involved:**  
  - Send Email Success Alert

- **Node Details:**  
  - **Send Email Success Alert**  
    - Type: Email Send node  
    - SMTP Credentials: Configured with secured email server credentials  
    - Email Settings:  
      - Subject: Includes timestamp `✅ FTP Migration Success - {{ $now.format('YYYY-MM-DD HH:mm') }}`  
      - Recipient: `admin@yourcompany.com`  
      - Sender: `noreply@yourcompany.com`  
    - Input: Log success data  
    - Output: None (final node)  
    - Failure Modes: SMTP authentication failure, network errors, invalid email addresses  
    - Notes: Allows batching or frequency adjustments; recommended for audit trail and operational transparency.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role               | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                           |
|-------------------------|---------------------|------------------------------|------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note         | Documentation Overview        | None                   | None                      | # 🚀 FTP FILE MIGRATION WORKFLOW... Comprehensive overview and quick start instructions.                                                |
| Performance             | Sticky Note         | Performance Optimization      | None                   | None                      | ## ⚡ PERFORMANCE OPTIMIZATION... Timeout and file size considerations.                                                                  |
| Security                | Sticky Note         | Security Best Practices       | None                   | None                      | ## 🔒 SECURITY BEST PRACTICES... Credential and connection security recommendations.                                                    |
| Troubleshooting         | Sticky Note         | Troubleshooting Guide         | None                   | None                      | ## 🔧 TROUBLESHOOTING GUIDE... Common errors and debug steps.                                                                           |
| File Processing         | Sticky Note         | File Processing Pipeline      | None                   | None                      | ## 📁 FILE PROCESSING PIPELINE... Filtering, order, and size limits for files.                                                          |
| Configuration           | Sticky Note         | Configuration Checklist       | None                   | None                      | ## ⚙️ CONFIGURATION CHECKLIST... Pre-deployment, testing, and post-deployment steps.                                                    |
| Daily Schedule (2 AM)   | Schedule Trigger    | Scheduled Trigger             | None                   | List Files - Source FTP    | SCHEDULE TRIGGER CONFIGURATION with detailed cron expression and timezone considerations.                                               |
| List Files - Source FTP | FTP                 | Source File Listing           | Daily Schedule (2 AM)  | Filter Files              | SOURCE FTP SERVER CONFIGURATION including credentials and connection notes.                                                             |
| Filter Files            | Filter              | File Filtering                | List Files - Source FTP | Download File             | FILE FILTERING & SELECTION with regex pattern and testing advice.                                                                        |
| Download File           | FTP                 | File Download                 | Filter Files           | Upload to Destination     | FILE DOWNLOAD OPERATION with retry, timeout, and memory considerations.                                                                  |
| Upload to Destination   | FTP                 | File Upload                   | Download File          | Upload Success?           | DESTINATION FTP UPLOAD details including credentials, options, and monitoring tips.                                                      |
| Upload Success?         | Filter              | Upload Verification           | Upload to Destination  | Delete Source File        | SUCCESS VALIDATION node checking upload success property.                                                                                |
| Delete Source File      | FTP                 | Source Cleanup                | Upload Success?        | Log Success               | SOURCE FILE CLEANUP operation with safety and alternative strategies.                                                                   |
| Log Success             | Set                 | Success Logging               | Delete Source File     | Send Email Success Alert  | SUCCESS LOGGING fields and output options.                                                                                              |
| Send Email Success Alert| Email Send          | Email Notification            | Log Success            | None                      | EMAIL SUCCESS NOTIFICATION with SMTP configuration and customizable content.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**:  
   - Name: `Daily Schedule (2 AM)`  
   - Set Cron Expression to `0 2 * * *` (daily at 2 AM)  
   - Configure timezone to Europe/Warsaw or as appropriate  
   - Save node.

3. **Add FTP node for source listing**:  
   - Name: `List Files - Source FTP`  
   - Operation: `List`  
   - Path: `/source/directory/`  
   - Credentials: Create new FTP credential named "Source FTP Server" with host, port (21 or 22), username, password  
   - Enable recursive listing  
   - Connect output of Schedule Trigger to this node's input.

4. **Add a Filter node**:  
   - Name: `Filter Files`  
   - Condition: Add regex condition on file name: `\.(txt|csv|json|xml)$` (case-sensitive)  
   - Connect output of `List Files - Source FTP` to Filter node.

5. **Add FTP node for file download**:  
   - Name: `Download File`  
   - Operation: `Download File`  
   - Path: Use expression `{{ $json.name }}` to dynamically download each filtered file  
   - Binary Property Name: `file_data`  
   - Use same "Source FTP Server" credentials  
   - Set timeout to 300 seconds (adjust as needed)  
   - Connect output of `Filter Files` to this node.

6. **Add FTP node for upload**:  
   - Name: `Upload to Destination`  
   - Operation: `Upload File`  
   - Path: `/destination/directory/{{ $json.name }}` (dynamic destination path preserving filename)  
   - Binary Property Name: `file_data` (from download node)  
   - Credentials: Create separate FTP credential "Destination FTP Server" with appropriate host, port, username, password  
   - Enable options: createDirectories = true, preserveTimestamp = true, overwriteExisting = false  
   - Connect output of `Download File` to this node.

7. **Add Filter node for upload success**:  
   - Name: `Upload Success?`  
   - Condition: Check if `{{ $json.success }}` equals `true`  
   - Connect output of `Upload to Destination` to this node.

8. **Add FTP node for deleting source file**:  
   - Name: `Delete Source File`  
   - Operation: `Delete`  
   - Path: Use expression `{{ $json.name }}` to specify file to delete  
   - Use "Source FTP Server" credentials  
   - Connect `true` output of `Upload Success?` node to this node.

9. **Add Set node for logging success**:  
   - Name: `Log Success`  
   - Assign fields:  
     - `timestamp` = `{{ $now.format('YYYY-MM-DD HH:mm:ss') }}`  
     - `filename` = `{{ $json.name }}`  
     - `status` = `"SUCCESS"`  
     - `file_size_bytes` = `{{ $json.size }}`  
     - `operation` = `"MIGRATE_AND_DELETE"`  
   - Connect output of `Delete Source File` to this node.

10. **Add Email Send node for success notification**:  
    - Name: `Send Email Success Alert`  
    - SMTP Credentials: Configure with your email SMTP server (host, port, username, password)  
    - From Email: `noreply@yourcompany.com`  
    - To Email: `admin@yourcompany.com` (or your recipient)  
    - Subject: `✅ FTP Migration Success - {{ $now.format('YYYY-MM-DD HH:mm') }}`  
    - Connect output of `Log Success` to this node.

11. **Save and activate the workflow** after thorough testing.

12. **Optional:** Add Sticky Note nodes at strategic points with key documentation, configuration checklists, troubleshooting, security, performance, and file processing guidelines for operator reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| ⚠️ **IMPORTANT**: Test thoroughly before production to avoid accidental data loss or service disruption.         | Workflow Overview Sticky Note                                         |
| Use SFTP (port 22) and SSH key authentication for better security and compliance.                                | Security Sticky Note                                                  |
| Adjust timeouts based on file sizes: large files (>100MB) need extended timeouts to prevent errors.               | Performance Sticky Note                                              |
| Regex filters can be customized to handle different file types, sizes, and date ranges as per business needs.    | File Processing Sticky Note                                          |
| Monitor execution logs and set up alerts to detect failures early.                                               | Troubleshooting Sticky Note                                          |
| Configure SMTP settings carefully to ensure email notifications are delivered reliably.                          | Configuration Sticky Note                                            |
| Consider batching email notifications in high-volume environments to prevent flooding.                          | Email Notification Node Notes                                        |
| Maintain separate credentials for source and destination FTP servers with least privilege and regular rotation.   | Security and FTP Nodes Notes                                         |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---