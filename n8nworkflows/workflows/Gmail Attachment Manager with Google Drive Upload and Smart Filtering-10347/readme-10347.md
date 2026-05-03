Gmail Attachment Manager with Google Drive Upload and Smart Filtering

https://n8nworkflows.xyz/workflows/gmail-attachment-manager-with-google-drive-upload-and-smart-filtering-10347


# Gmail Attachment Manager with Google Drive Upload and Smart Filtering

### 1. Workflow Overview

This workflow automates the processing of attachments received via Gmail. Its primary goal is to filter incoming emails with attachments based on sender/recipient and attachment file types, process the attachments differently depending on their MIME type (e.g., decompress ZIP files), upload the processed files to Google Drive, and notify a Slack channel about the processed files. It also marks processed emails as read and archives them.

**Logical blocks:**

- **1.1 Trigger and Email Reception:** Detect incoming Gmail messages with attachments.
- **1.2 Sender/Receiver Filtering:** Filter emails based on specific sender or recipient addresses.
- **1.3 Attachment Extraction and Looping:** Extract attachments and loop through each file.
- **1.4 Attachment Type Filtering:** Filter attachments based on MIME types.
- **1.5 File Type-Specific Processing:** Process files differently based on their type (e.g., unzip archives).
- **1.6 File Upload:** Upload processed files to Google Drive or optionally post to a webhook.
- **1.7 Email Archival:** Mark emails as read and archive them in Gmail.
- **1.8 Slack Notification:** Send a Slack message summarizing the processed files.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Email Reception

- **Overview:** Watches the Gmail inbox for any new incoming emails that contain attachments, downloading the attachments automatically.
- **Nodes Involved:** `Trigger on incoming email with attachment`
- **Node Details:**

  - **Name:** Trigger on incoming email with attachment  
  - **Type:** Gmail Trigger  
  - **Role:** Watches Gmail mailbox for new emails with attachments, triggering workflow upon detection.  
  - **Configuration:**  
    - Uses Gmail API with polling every minute.  
    - Filter query: `has:attachment -label:DRAFT` to only fetch emails with attachments excluding drafts.  
    - Downloads attachments automatically.  
  - **Input:** None (trigger node)  
  - **Output:** Email data including attachments (binary data)  
  - **Edge Cases/Potential Failures:**  
    - Gmail API quota limits or authentication failure.  
    - Attachment download failures due to file size or network.  
  - **Version:** 1.3

#### 1.2 Sender/Receiver Filtering

- **Overview:** Filters emails based on specific sender or recipient addresses to process only relevant emails.
- **Nodes Involved:** `Filter based on sender or receiver`
- **Node Details:**

  - **Name:** Filter based on sender or receiver  
  - **Type:** Filter  
  - **Role:** Checks if the email's sender or recipient matches configured email addresses.  
  - **Configuration:**  
    - Filters using OR condition: matches if sender is `sender@domain.tld` or recipient is `recipient@domain.tld`.  
  - **Input:** Email data from trigger node  
  - **Output:** Passes matching emails forward; others are filtered out.  
  - **Edge Cases:**  
    - Emails with multiple senders or recipients not handled (only first address checked).  
    - Case sensitivity enabled; mismatches possible if email case varies.  
  - **Version:** 2.2

#### 1.3 Attachment Extraction and Looping

- **Overview:** Extracts binary attachment data from the email JSON and loops through each attachment individually.
- **Nodes Involved:** `Split Out`, `Loop attachments`
- **Node Details:**

  - **Split Out:**  
    - **Type:** Split Out  
    - **Role:** Extracts the `$binary` field (attachments) to separate items for processing.  
    - **Input:** Filtered email JSON with attachments  
    - **Output:** Each attachment as an individual item  
    - **Edge Cases:** Missing or empty `$binary` field leads to no attachments to process.  
    - **Version:** 1

  - **Loop attachments:**  
    - **Type:** Split In Batches  
    - **Role:** Processes attachments one by one or in batches (default batch size 1).  
    - **Input:** Items from `Split Out`  
    - **Output:** Single attachment per iteration  
    - **Edge Cases:** Large batch sizes may cause memory issues; reset option disabled to ensure continuous processing.  
    - **Version:** 3

#### 1.4 Attachment Type Filtering

- **Overview:** Filters attachments to only process those with specific MIME types (ZIP archives, plain text, or any file if no MIME type).
- **Nodes Involved:** `Filter based on file type`
- **Node Details:**

  - **Name:** Filter based on file type  
  - **Type:** Filter  
  - **Role:** Allows only certain MIME types to continue processing:  
    - `application/zip`  
    - `text/plain`  
    - Or any file with a MIME type present (exists)  
  - **Input:** Single attachment item from loop  
  - **Output:** Passes filtered attachments to next node  
  - **Edge Cases:** If attachment has no MIME type or unexpected MIME type, it may be filtered out.  
  - **Version:** 2.2

#### 1.5 File Type-Specific Processing

- **Overview:** Processes attachments differently depending on their MIME type; decompresses ZIP files, uploads other files directly.
- **Nodes Involved:** `Treat different file types different`, `Decompress zip`, `Upload file to Google Drive`, `Loop attachments` (fallback)
- **Node Details:**

  - **Treat different file types different:**  
    - **Type:** Switch  
    - **Role:** Routes attachments based on MIME type.  
    - **Cases:**  
      - `zip`: routes to decompression node  
      - `no file`: routes back to loop (no attachments)  
      - `extra`: fallback for other file types, routes to Google Drive upload  
    - **Input:** Filtered attachments  
    - **Output:** Routes accordingly  
    - **Edge Cases:**  
      - If MIME type is missing or unexpected, files go to fallback (upload directly).  
    - **Version:** 3.3

  - **Decompress zip:**  
    - **Type:** Compression (Decompress)  
    - **Role:** Extracts files inside ZIP archives for further processing.  
    - **Configuration:**  
      - Output files prefixed with "file_"  
      - Operates on dynamic binary property (first binary key)  
    - **Input:** ZIP file from switch  
    - **Output:** Extracted files as binary data  
    - **Edge Cases:**  
      - Corrupt ZIP files may cause errors or empty output.  
    - **Version:** 1.1

  - **Upload file to Google Drive:**  
    - **Type:** Google Drive  
    - **Role:** Uploads files (extracted or direct) to Google Drive in specified folder.  
    - **Configuration:**  
      - Uses "My Drive" root folder as default destination  
      - File name taken from binary data's filename property  
      - Input binary data dynamically assigned  
    - **Input:** Files from decompression or directly from switch fallback  
    - **Output:** Confirmation of upload  
    - **Edge Cases:**  
      - Authentication failures with Google Drive credentials  
      - Folder permissions or quota exceeded errors  
    - **Version:** 3

  - **Loop attachments (fallback):**  
    - This path appears for the "no file" output key, looping again to continue processing if no attachments found.  
    - Ensures workflow continues gracefully if no files are detected.  

#### 1.6 File Upload Alternative: Post File to Webhook

- **Overview:** Alternative node to upload or send files to an external webhook endpoint instead of Google Drive.
- **Nodes Involved:** `Post file to webhook`
- **Node Details:**

  - **Type:** HTTP Request  
  - **Role:** Posts binary file data to an external webhook URL via POST method.  
  - **Configuration:**  
    - URL placeholder `https://n8n.com:8443/webhook/[unique-id]` (to be customized)  
    - Sends binary data as request body  
    - Uses dynamic binary property key for input data  
  - **Input:** Binary files  
  - **Output:** HTTP response from webhook  
  - **Edge Cases:**  
    - Network errors or invalid webhook URL cause failures  
    - Webhook endpoint must accept binary data  
  - **Version:** 4.2

#### 1.7 Email Archival

- **Overview:** After processing all attachments, the original email is marked as read and removed from the inbox (archived).
- **Nodes Involved:** `Mark read and archive email`
- **Node Details:**

  - **Type:** Gmail  
  - **Role:** Removes `UNREAD` and `INBOX` labels from the email to mark it read and archive it.  
  - **Configuration:**  
    - Operates on the message ID from the filtered email node  
    - Removes labels `UNREAD`, `INBOX`  
  - **Input:** Triggered after looping through all attachments  
  - **Output:** Confirmation of label removal  
  - **Edge Cases:**  
    - Message ID must be valid; if email is deleted or altered, operation may fail  
    - Requires valid Gmail credentials with label management permissions  
  - **Version:** 2.1

#### 1.8 Slack Notification

- **Overview:** Sends a Slack message summarizing how many files were processed and who the email sender was.
- **Nodes Involved:** `Send a message`
- **Node Details:**

  - **Type:** Slack  
  - **Role:** Posts notification text to a configured Slack channel.  
  - **Configuration:**  
    - Message text includes dynamic expressions for:  
      - Number of files processed (`Loop attachments` count)  
      - Email sender address  
  - **Input:** Triggered after email archival node  
  - **Output:** Slack message posted  
  - **Edge Cases:**  
    - Slack credentials must be valid and have permission to post to channel  
    - Expressions must resolve correctly or message will be malformed  
  - **Version:** 2.3

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                       | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                                        |
|-----------------------------------|---------------------|------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Trigger on incoming email with attachment | Gmail Trigger       | Watches Gmail inbox for emails with attachments | None                             | Filter based on sender or receiver | ## 1. Trigger Configure your credentials here                                                                                      |
| Filter based on sender or receiver| Filter              | Filters emails by sender or recipient address | Trigger on incoming email with attachment | Split Out                        | ## 3. Filter sender/receiver                                                                                                       |
| Split Out                         | Split Out           | Extracts attachments from email JSON | Filter based on sender or receiver | Loop attachments                 |                                                                                                                                   |
| Loop attachments                 | Split In Batches    | Loops through each attachment      | Split Out                        | Mark read and archive email, Filter based on file type |                                                                                                                                   |
| Filter based on file type         | Filter              | Filters attachments by MIME type   | Loop attachments                | Treat different file types different | ## 4. Filter based on file type                                                                                                   |
| Treat different file types different | Switch             | Routes files based on MIME type    | Filter based on file type        | Decompress zip, Loop attachments, Upload file to Google Drive | ## 5. Treat different file types separate                                                                                         |
| Decompress zip                   | Compression         | Decompresses ZIP files             | Treat different file types different | Upload file to Google Drive       |                                                                                                                                   |
| Upload file to Google Drive       | Google Drive        | Uploads files to Google Drive      | Decompress zip, Treat different file types different | Loop attachments                 | ## 6. Upload file somewhere Configure or replace this node with some other destination for your files.                            |
| Mark read and archive email       | Gmail               | Archives and marks email as read   | Loop attachments                | Send a message                   | ## 2. Archive Configure your credentials here                                                                                      |
| Send a message                   | Slack               | Sends Slack notification           | Mark read and archive email     | None                            | ## 7. Slack notification Configure your credentials here                                                                           |
| Post file to webhook              | HTTP Request        | Alternative upload method via webhook | None (alternative path)          | None                            | ## 8. Example This is a webhook example to replace the Google Drive node (6)                                                      |
| Sticky Note                      | Sticky Note         | Informational                      | None                            | None                            | Template created by Smultron Studio (https://smultronstudio.com/en) - feel free to reach out at hello@smultronstudio.com           |
| Sticky Note1                     | Sticky Note         | Informational                      | None                            | None                            |                                                                                                                                   |
| Sticky Note2                     | Sticky Note         | Slack notification info            | None                            | None                            | ## Notification out Notify in Slack that files has been processed                                                                 |
| Sticky Note3                     | Sticky Note         | Upload configuration info          | None                            | None                            | ## 6. Upload file somewhere Configure or replace this node with some other destination for your files.                            |
| Sticky Note4                     | Sticky Note         | File type filtering info           | None                            | None                            | ## 4. Filter based on file type If you expect one or more file types, you can specify that here.                                  |
| Sticky Note5                     | Sticky Note         | File type processing info          | None                            | None                            | ## 5. Treat different file types separate If you expect multiple file types that requires different processing, build those paths here (like decompressing zip files). If you filter to only process one specific file type or will process all files the same then you can remove this part. |
| Sticky Note6                     | Sticky Note         | Trigger configuration info         | None                            | None                            | ## 1. Trigger Configure your credentials here                                                                                      |
| Sticky Note7                     | Sticky Note         | Archive node configuration info    | None                            | None                            | ## 2. Archive Configure your credentials here                                                                                      |
| Sticky Note8                     | Sticky Note         | Sender/receiver filter info        | None                            | None                            | ## 3. Filter sender/receiver                                                                                                       |
| Sticky Note9                     | Sticky Note         | Slack notification configuration   | None                            | None                            | ## 7. Slack notification Configure your credentials here                                                                           |
| Sticky Note10                    | Sticky Note         | Webhook upload example             | None                            | None                            | ## 8. Example This is a webhook example to replace the Google Drive node (6)                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**
   - Type: Gmail Trigger  
   - Credentials: Configure Gmail OAuth2 credentials with access to read emails and attachments.  
   - Parameters:  
     - Poll every minute.  
     - Filter query: `has:attachment -label:DRAFT`.  
     - Enable attachment download.

2. **Create Filter Node for Sender/Receiver:**
   - Type: Filter  
   - Input: Connect from Gmail Trigger.  
   - Conditions:  
     - OR combinator.  
     - Check if `from.value[0].address` equals `sender@domain.tld`.  
     - OR check if `to.value[0].address` equals `recipient@domain.tld`.  
   - Case sensitive matching enabled.

3. **Create Split Out Node:**
   - Type: Split Out  
   - Input: Connect from sender/receiver filter node.  
   - Field to split out: `$binary` (attachment binary data).

4. **Create Split In Batches Node (Loop attachments):**
   - Type: Split In Batches  
   - Input: Connect from Split Out.  
   - Configure batch size to 1 (default).  
   - Disable reset option.

5. **Create Filter Node for File Type:**
   - Type: Filter  
   - Input: Connect from Loop attachments.  
   - Conditions: OR combinator, checking if the MIME type of the binary data matches:  
     - `application/zip`  
     - `text/plain`  
     - OR MIME type exists (any file).  

6. **Create Switch Node (Treat different file types differently):**
   - Type: Switch  
   - Input: Connect from File type filter.  
   - Rules:  
     - If MIME type equals `application/zip` → output "zip"  
     - If no files found (empty input) → output "no file"  
     - Fallback output: "extra" (all other files).

7. **Create Compression Node (Decompress zip):**
   - Type: Compression  
   - Input: Connect from Switch node's "zip" output.  
   - Operation: Decompress.  
   - Binary property name: Use first binary property key dynamically.  
   - Output prefix: `file_`.

8. **Create Google Drive Node (Upload file to Google Drive):**
   - Type: Google Drive  
   - Input: Connect from Decompress zip node and also from Switch node "extra" output.  
   - Credentials: Configure Google Drive OAuth2 credentials with upload permissions.  
   - Parameters:  
     - Drive: "My Drive"  
     - Folder: Root or specify desired folder ID.  
     - File name: Use binary filename property dynamically.  
     - Binary data input: Use first binary key dynamically.

9. **Connect Google Drive node output to Loop attachments node to continue processing remaining files.**

10. **Create Gmail Node (Mark read and archive email):**
    - Type: Gmail  
    - Input: Connect from Loop attachments node (after file processing).  
    - Credentials: Use Gmail OAuth2 credentials with label modification rights.  
    - Operation: Remove labels `UNREAD` and `INBOX` from the email using message ID from filtered email node.

11. **Create Slack Node (Send a message):**
    - Type: Slack  
    - Input: Connect from Gmail archival node.  
    - Credentials: Configure Slack OAuth2 credentials with permission to post messages.  
    - Message Text:  
      ```
      Did something with {{$('Loop attachments').all().length}} files received from {{ $('Trigger on incoming email with attachment').item.json.from.value[0].address }}
      ```

12. **Optional: Create HTTP Request node for webhook upload alternative:**
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: Your webhook URL  
      - Send binary data as request body with dynamic binary key.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Template created by Smultron Studio (https://smultronstudio.com/en) - feel free to reach out at hello@smultronstudio.com                                         | Template credits and support                                                                      |
| List of MIME types for filtering attachments: https://docs.cloud.google.com/appengine/docs/legacy/standard/php/mail/mail-with-headers-attachments               | MIME types reference for file filtering                                                          |
| Slack notification node requires configured Slack credentials with scope to post messages                                                                        | Slack API documentation: https://api.slack.com/messaging                                          |
| Google Drive upload requires OAuth2 credentials with drive.file or drive permissions                                                                              | Google Drive API documentation: https://developers.google.com/drive/api/v3/about-auth             |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.