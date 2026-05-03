Automatic Gmail Invoice PDF Upload to Google Drive with Spreadsheet Logging

https://n8nworkflows.xyz/workflows/automatic-gmail-invoice-pdf-upload-to-google-drive-with-spreadsheet-logging-9610


# Automatic Gmail Invoice PDF Upload to Google Drive with Spreadsheet Logging

### 1. Workflow Overview

This workflow automates the process of extracting invoice PDFs from incoming Gmail emails, uploading them to Google Drive, and logging relevant information in a Google Sheets spreadsheet for record-keeping. It is designed for users who regularly receive invoice emails and want to automate archival and tracking without manual intervention.

The workflow is divided into the following logical blocks:

- **1.1 Gmail Input Monitoring**: Watches for new unread emails in Gmail's personal category that contain PDF attachments.
- **1.2 Attachment Validation**: Filters emails to ensure they contain PDF attachments before processing.
- **1.3 Google Drive Upload**: Uploads the extracted PDF invoice attachments to a specified Google Drive folder.
- **1.4 Google Sheets Logging**: Logs metadata about the uploaded files and emails into a Google Sheets spreadsheet for tracking.
- **1.5 Email Status Update**: Marks the processed email as read to prevent duplicate processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Gmail Input Monitoring

- **Overview:**  
  This block uses a Gmail Trigger node to monitor the user's inbox for new unread emails categorized under "Personal". It downloads any attachments present to enable further processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Sticky Note (Gmail Trigger Explanation)

- **Node Details:**  

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail integration  
    - Configuration:  
      - Monitors emails labeled as "CATEGORY_PERSONAL"  
      - Filters for unread emails only  
      - Checks every minute (polling)  
      - Downloads all attachments automatically  
    - Expressions/Variables: None significant beyond standard polling settings  
    - Input/Output: Trigger node, no input; outputs matched emails with attachments  
    - Failure Modes:  
      - Authentication errors if Gmail credentials expire or are misconfigured  
      - API rate limits or network timeouts  
      - Emails without attachments are still passed on (filtered later)  
    - Sticky Note Content: Explains monitoring and attachment download setup  
    - Sub-workflow: None  

#### 2.2 Attachment Validation

- **Overview:**  
  This block filters incoming emails to process only those which actually contain PDF attachments, ensuring that irrelevant emails are ignored.

- **Nodes Involved:**  
  - Has PDF Attachments? (Filter node)  
  - If (Conditional node)  
  - Sticky Note (Filter Check Explanation)

- **Node Details:**  

  - **Has PDF Attachments?**  
    - Type: Filter node  
    - Configuration:  
      - Checks that the email has at least one binary attachment  
      - Ensures the attachment is a PDF (intended, but filtering on file type not explicitly visible in parameters)  
    - Expressions:  
      - Uses expression `={{ Object.keys($binary ?? {}).length }}` to check attachment presence  
    - Input: Gmail Trigger output  
    - Output: Passes only emails with attachments  
    - Failure Modes:  
      - Emails with unsupported attachment types might pass if not filtered by file type  
      - Expression evaluation failure if `$binary` is undefined or malformed  
  - **If**  
    - Type: Conditional node  
    - Configuration:  
      - Checks again if there are binary attachments (`$binary` exists and length > 0)  
    - Input: Output of Filter node  
    - Output: Routes emails with attachments to Google Drive Upload node  
    - Failure Modes: Misconfiguration or expression errors  
  - Sticky Note Content: Describes the purpose of attachment filtering  

#### 2.3 Google Drive Upload

- **Overview:**  
  This block uploads the invoice PDF attachments to Google Drive, naming files according to their original filenames or by timestamp if unavailable.

- **Nodes Involved:**  
  - Upload to Google Drive  
  - Sticky Note (Google Drive Upload Explanation)

- **Node Details:**  

  - **Upload to Google Drive**  
    - Type: Google Drive node  
    - Configuration:  
      - Uploads a file to Google Drive root folder by default (configurable)  
      - Filename is set via expression: `={{ $json.filename || 'invoice_' + $now.toFormat('yyyyMMdd_HHmmss') + '.pdf' }}`  
      - Input data field: `attachment_0` (first attachment)  
    - Input: Output from "If" node (emails with attachments)  
    - Output: Uploaded file metadata  
    - Failure Modes:  
      - Authentication errors on Google Drive credentials  
      - API limits or quota exceeded  
      - Missing or malformed attachment data  
    - Sticky Note Content: Instructions for credential setup and folder configuration  

#### 2.4 Google Sheets Logging

- **Overview:**  
  This block logs detailed metadata about the uploaded invoice files into a specified Google Sheets spreadsheet, facilitating tracking and indexing.

- **Nodes Involved:**  
  - Log to Google Sheets  
  - Sticky Note (Google Sheets Logging Explanation)

- **Node Details:**  

  - **Log to Google Sheets**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append or update rows in a spreadsheet  
      - Spreadsheet ID and sheet name must be configured externally  
      - Columns mapped automatically from Google Drive file metadata fields such as name, id, mimeType, createdTime, etc.  
    - Input: Output from Google Drive Upload node  
    - Output: Confirmation of logging operation  
    - Failure Modes:  
      - Authentication errors with Google Sheets credentials  
      - Invalid spreadsheet ID or sheet name  
      - Schema mismatch if sheet headers don't align with mapped fields  
    - Sticky Note Content: Notes on spreadsheet setup and column headers  

#### 2.5 Email Status Update

- **Overview:**  
  After successful upload and logging, this block marks the original email as read in Gmail to avoid reprocessing.

- **Nodes Involved:**  
  - Mark Email as Read (Gmail node)  
  - Sticky Note (Mark as Processed Explanation)

- **Node Details:**  

  - **Mark Email as Read**  
    - Type: Gmail node (non-trigger)  
    - Configuration:  
      - Operation: Mark as read  
      - Uses message ID from the Gmail Trigger node's output to identify the email  
    - Input: Output from Google Sheets Logging node  
    - Output: Confirmation of marking email as read  
    - Failure Modes:  
      - Authentication issues with Gmail credentials  
      - If message ID is missing or invalid  
    - Sticky Note Content: Explains the importance of marking emails processed  

---

### 3. Summary Table

| Node Name              | Node Type              | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                       |
|------------------------|------------------------|------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger          | Gmail Trigger           | Monitor Gmail inbox for new emails | None                   | Has PDF Attachments?    | 📧 Gmail Trigger: Monitors inbox every minute for unread emails with attachments. Requires Gmail credentials.                    |
| Has PDF Attachments?   | Filter                  | Filter emails with PDF attachments | Gmail Trigger          | If                     | 🔍 Filter Check: Ensures only emails with attachments are processed.                                                              |
| If                     | Conditional             | Check for presence of attachments  | Has PDF Attachments?   | Upload to Google Drive  |                                                                                                                                    |
| Upload to Google Drive | Google Drive            | Upload PDF attachments to Drive    | If                     | Log to Google Sheets    | 📁 Google Drive Upload: Uploads PDFs to Drive with original filename or timestamp. Requires Google Drive credentials.            |
| Log to Google Sheets   | Google Sheets           | Log file metadata in spreadsheet   | Upload to Google Drive  | Mark Email as Read      | 📊 Google Sheets Logging: Logs invoice details. Configure spreadsheet ID, sheet name, and column headers. Requires Sheets creds. |
| Mark Email as Read     | Gmail                   | Mark processed email as read       | Log to Google Sheets    | None                   | ✅ Mark as Processed: Marks email as read to prevent reprocessing. Uses same Gmail credentials as trigger.                      |
| Sticky Note            | Sticky Note             | Documentation/Comments             | N/A                    | N/A                    | See individual notes for each block.                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Credentials: Connect Gmail OAuth2 credentials  
   - Parameters:  
     - Label IDs: Select "CATEGORY_PERSONAL"  
     - Read Status: "Unread"  
     - Polling interval: Every 1 minute  
     - Options: Enable "Download Attachments"  
   - Position node at start of workflow

2. **Add Filter Node "Has PDF Attachments?":**  
   - Type: Filter  
   - Condition:  
     - Check if attachments exist: Use expression `Object.keys($binary ?? {}).length > 0`  
     - (Optional) Add check if attachment file extension is `.pdf` (not explicitly in original but recommended)  
   - Connect Gmail Trigger output to this node's input

3. **Add Conditional Node "If":**  
   - Type: If  
   - Condition:  
     - Check that `$binary` exists and has keys length > 0  
   - Connect Filter node output to If node input

4. **Add Google Drive Node "Upload to Google Drive":**  
   - Type: Google Drive  
   - Credentials: Connect Google Drive OAuth2 credentials  
   - Parameters:  
     - Name: Use expression `{{$json.filename || 'invoice_' + $now.toFormat('yyyyMMdd_HHmmss') + '.pdf'}}`  
     - Folder ID: Default to "root" or specify folder ID  
     - Input field name: `attachment_0` (first attachment)  
   - Connect "If" node output (true path) to this node

5. **Add Google Sheets Node "Log to Google Sheets":**  
   - Type: Google Sheets  
   - Credentials: Connect Google Sheets OAuth2 credentials  
   - Parameters:  
     - Operation: Append or update  
     - Spreadsheet ID: Paste your spreadsheet ID here  
     - Sheet name: Enter your sheet name (e.g., "Sheet1")  
     - Columns: Map columns automatically or manually to file metadata fields such as name, id, createdTime, webViewLink, etc.  
   - Connect Google Drive node output to this node

6. **Add Gmail Node "Mark Email as Read":**  
   - Type: Gmail (non-trigger)  
   - Credentials: Use same Gmail credentials as trigger node  
   - Parameters:  
     - Operation: Mark as read  
     - Message ID: Use expression `{{$node["Gmail Trigger"].json["id"]}}` to get email ID  
   - Connect Google Sheets node output to this node

7. **Add Sticky Notes (optional):**  
   - Add sticky notes to document each block with instructions and configuration tips, matching the notes described in the overview.

8. **Configure Workflow Settings:**  
   - Set timezone to your preference (e.g., America/New_York)  
   - Enable saving of manual executions and execution progress for debugging  
   - Configure error handling workflows if desired

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup instructions include connecting OAuth2 credentials for Gmail, Google Drive, and Google Sheets, and specifying spreadsheet IDs. | See Sticky Note5 in the workflow for detailed step-by-step setup instructions.                   |
| Ensure your Google Sheets document has headers matching the logged columns: Date, Sender, Subject, Filename, Drive_Link, File_ID, Email_ID | This ensures proper data mapping and avoids schema mismatches in the Google Sheets node.        |
| Optional: Modify the Google Drive folder ID in the upload node to organize invoices into a specific folder instead of root.             | Recommended for better file management and to avoid clutter in your root drive.                  |
| This workflow is inactive by default; enable it after configuring credentials and parameters.                                            | Prevents accidental runs before setup is complete.                                              |
| For testing, send an email to your Gmail account with the word "invoice" in the subject and attach a PDF invoice file.                  | Validates end-to-end functionality before deployment.                                           |
| Workflow timezone is set to America/New_York; adjust in workflow settings if different timezone is desired.                             | Important for correct timestamping in file names and spreadsheet logs.                           |

---

This document fully describes the workflow's structure, node configurations, dependencies, and operational logic. It enables reproduction, modification, and troubleshooting by advanced users or automation agents.