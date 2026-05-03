Extract data from Gmail attachments to Google Sheets using DocuPipe

https://n8nworkflows.xyz/workflows/extract-data-from-gmail-attachments-to-google-sheets-using-docupipe-14329


# Extract data from Gmail attachments to Google Sheets using DocuPipe

# 1. Workflow Overview

This workflow automates document extraction from Gmail attachments and stores the extracted structured data in Google Sheets using DocuPipe.

It is designed for teams that receive files such as invoices, receipts, or contracts by email and want to avoid manual data entry. The workflow is split into two independent but related execution paths:

- **Scenario 1 - Upload:** detect incoming Gmail messages with attachments, mark them as being processed, send the first attachment to DocuPipe for extraction, and save a backup copy in Google Drive.
- **Scenario 2 - Process & Save:** receive DocuPipe’s completion event through a webhook trigger, fetch the extraction result, flatten it into a spreadsheet-friendly structure, enrich it with metadata, and append it to Google Sheets.

## 1.1 Input Reception and Preprocessing
This block monitors Gmail for new emails with attachments and labels matching emails as “Processing” before extraction starts.

## 1.2 DocuPipe Upload and Backup
This block uploads the binary attachment to DocuPipe using a selected extraction schema and stores a backup copy in Google Drive.

## 1.3 Webhook-Based Extraction Completion
This block listens for DocuPipe completion events and starts the second scenario when extraction succeeds.

## 1.4 Result Retrieval and Transformation
This block retrieves the extracted data from DocuPipe, flattens nested structures into JSON strings, and prepares a row-compatible payload.

## 1.5 Spreadsheet Enrichment and Persistence
This block adds metadata such as the document name and extraction timestamp, then appends the final row to Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Preprocessing

### Overview
This block starts the first scenario by polling Gmail every minute for emails that contain attachments. Once a message is found, it is labeled as “DocuPipe - Processing” so users can distinguish items already sent for extraction.

### Nodes Involved
- New Email with Attachment
- Label as Processing

### Node Details

#### New Email with Attachment
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`; polling trigger for Gmail messages.
- **Configuration choices:**
  - Filters only emails with attachments.
  - Downloads attachments automatically.
  - Polling interval is every minute.
- **Key expressions or variables used:**
  - No custom expression in the node itself.
  - Provides message metadata and binary attachment data downstream.
- **Input and output connections:**
  - No input; this is an entry point.
  - Outputs to **Label as Processing**.
- **Version-specific requirements:**
  - Uses node type version `1`.
- **Edge cases or potential failure types:**
  - Gmail OAuth2 credential expiration or missing authorization.
  - Polling delays depending on Gmail/API timing.
  - Emails with multiple attachments are only partially handled later because the upload node explicitly uses `attachment_0`.
  - If attachments are not downloaded correctly, downstream binary-dependent nodes will fail.
- **Sub-workflow reference:** None.

#### Label as Processing
- **Type and technical role:** `n8n-nodes-base.gmail`; updates the Gmail message by adding a label.
- **Configuration choices:**
  - Resource: message.
  - Operation: add labels.
  - Uses the current Gmail message ID.
  - Adds the label `DocuPipe - Processing`.
- **Key expressions or variables used:**
  - `={{ $json.id }}` for the Gmail message ID.
- **Input and output connections:**
  - Input from **New Email with Attachment**.
  - Output to **Upload Attachment & Extract Data**.
- **Version-specific requirements:**
  - Uses node type version `2.1`.
  - The specified label should already exist in Gmail, unless changed to an existing custom label.
- **Edge cases or potential failure types:**
  - Failure if the label does not exist in Gmail.
  - Gmail permission issues for modifying messages.
  - If the incoming item lacks `id`, the expression will fail.
- **Sub-workflow reference:** None.

---

## 2.2 DocuPipe Upload and Backup

### Overview
This block sends the attachment to DocuPipe for asynchronous extraction and then uploads a backup copy of the same file to Google Drive. The extraction is not completed inline; the actual results are handled later by the webhook-based second scenario.

### Nodes Involved
- Upload Attachment & Extract Data
- Save Attachment to Drive

### Node Details

#### Upload Attachment & Extract Data
- **Type and technical role:** `n8n-nodes-docupipe.docuPipe`; DocuPipe community node for creating an extraction job from a binary file.
- **Configuration choices:**
  - Resource: extraction.
  - Operation: upload and extract.
  - Input mode: binary.
  - Binary property name: `attachment_0`.
  - Requires selection of a DocuPipe extraction schema.
- **Key expressions or variables used:**
  - Uses binary input property `attachment_0`.
- **Input and output connections:**
  - Input from **Label as Processing**.
  - Output to **Save Attachment to Drive**.
- **Version-specific requirements:**
  - Uses node type version `1`.
  - Requires the **DocuPipe community node** to be installed on the n8n instance.
  - Self-hosted n8n is required for community nodes in this setup.
- **Edge cases or potential failure types:**
  - Missing or invalid DocuPipe API key.
  - Schema not selected or unavailable.
  - Failure if the email attachment is not present under `attachment_0`.
  - If the email has multiple attachments, only the first attachment is processed unless the workflow is expanded.
  - File type or size restrictions may apply on the DocuPipe side.
- **Sub-workflow reference:** None.

#### Save Attachment to Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the processed attachment as a backup file to Google Drive.
- **Configuration choices:**
  - Operation: upload.
  - File name is set from the Gmail subject line.
  - Upload destination is a selected Google Drive folder.
- **Key expressions or variables used:**
  - `={{ $('New Email with Attachment').item.json.subject }}`
- **Input and output connections:**
  - Input from **Upload Attachment & Extract Data**.
  - No downstream node in this scenario.
- **Version-specific requirements:**
  - Uses node type version `3`.
- **Edge cases or potential failure types:**
  - Google Drive OAuth2 authentication issues.
  - Folder not selected or inaccessible.
  - The file name may be non-unique if multiple emails share the same subject.
  - Depending on incoming binary handling, this node may fail if no valid binary payload is available for upload.
  - Subject may be empty or contain characters that lead to undesirable file names.
- **Sub-workflow reference:** None.

---

## 2.3 Webhook-Based Extraction Completion

### Overview
This block starts the second scenario. It listens for DocuPipe’s success event and receives the reference needed to fetch completed extraction results.

### Nodes Involved
- Extraction Complete

### Node Details

#### Extraction Complete
- **Type and technical role:** `n8n-nodes-docupipe.docuPipeTrigger`; webhook-style DocuPipe trigger for successful extraction completion events.
- **Configuration choices:**
  - Event subscribed: `standardization.processed.success`.
  - Uses a generated webhook ID inside n8n.
- **Key expressions or variables used:**
  - No custom expression configured.
  - Emits a payload containing `standardizationId`, which is used later.
- **Input and output connections:**
  - No input; this is a second entry point.
  - Outputs to **Get Extracted Data**.
- **Version-specific requirements:**
  - Uses node type version `1`.
  - Requires DocuPipe credentials and a reachable webhook endpoint from DocuPipe to n8n.
- **Edge cases or potential failure types:**
  - Webhook not reachable because n8n is not publicly accessible.
  - Wrong environment URL in n8n webhook configuration.
  - Missing `standardizationId` in webhook payload.
  - Event mismatch if DocuPipe sends a different event than configured.
- **Sub-workflow reference:** None.

---

## 2.4 Result Retrieval and Transformation

### Overview
This block fetches the finished extraction result from DocuPipe and converts it into a single flat object suitable for insertion into a spreadsheet. Arrays and nested objects are stringified to preserve them in one row.

### Nodes Involved
- Get Extracted Data
- Process Extraction Result

### Node Details

#### Get Extracted Data
- **Type and technical role:** `n8n-nodes-docupipe.docuPipe`; retrieves extraction results from DocuPipe using the completion identifier.
- **Configuration choices:**
  - Resource: extraction.
  - Operation: get result.
  - Standardization ID comes from the trigger payload.
- **Key expressions or variables used:**
  - `={{ $json.standardizationId }}`
- **Input and output connections:**
  - Input from **Extraction Complete**.
  - Output to **Process Extraction Result**.
- **Version-specific requirements:**
  - Uses node type version `1`.
  - Requires the DocuPipe community node.
- **Edge cases or potential failure types:**
  - Invalid or missing `standardizationId`.
  - API authentication or rate-limit issues.
  - Result not yet available despite webhook firing, depending on timing or backend delays.
- **Sub-workflow reference:** None.

#### Process Extraction Result
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript transformation node.
- **Configuration choices:**
  - Reads `data.result`.
  - Iterates over all top-level fields.
  - Converts arrays and objects to JSON strings.
  - Leaves primitive values unchanged.
  - Returns a single item containing the flattened row object.
- **Key expressions or variables used:**
  - Uses `$input.first().json`
  - Reads `data.result || {}`
- **Input and output connections:**
  - Input from **Get Extracted Data**.
  - Output to **Add Metadata**.
- **Version-specific requirements:**
  - Uses node type version `2`.
- **Edge cases or potential failure types:**
  - If `result` is missing, the node returns an empty object.
  - Nested data becomes JSON strings, which is acceptable for spreadsheets but may reduce direct filterability in Sheets.
  - Unexpected data types may still serialize, but downstream sheet headers may not match.
- **Sub-workflow reference:** None.

---

## 2.5 Spreadsheet Enrichment and Persistence

### Overview
This block adds metadata fields to the flattened extraction output and writes the final item as a new row in Google Sheets. It assumes the spreadsheet already contains matching headers for the extraction fields and metadata fields.

### Nodes Involved
- Add Metadata
- Append Row to Spreadsheet

### Node Details

#### Add Metadata
- **Type and technical role:** `n8n-nodes-base.set`; augments the transformed extraction result with extra fields.
- **Configuration choices:**
  - Keeps all incoming fields.
  - Adds:
    - `Document Name`
    - `Extracted At`
- **Key expressions or variables used:**
  - `={{ $('Get Extracted Data').item.json.documentName }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**
  - Input from **Process Extraction Result**.
  - Output to **Append Row to Spreadsheet**.
- **Version-specific requirements:**
  - Uses node type version `3.4`.
- **Edge cases or potential failure types:**
  - If `documentName` is missing from DocuPipe output, `Document Name` may be empty.
  - Spreadsheet headers must include these exact column names if auto-mapping is expected to work fully.
- **Sub-workflow reference:** None.

#### Append Row to Spreadsheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends the final extraction data as a new row in a chosen sheet.
- **Configuration choices:**
  - Uses auto-map input data to columns.
  - Spreadsheet selected by document ID.
  - Target sheet selected by sheet ID.
- **Key expressions or variables used:**
  - No custom expression in the node itself.
- **Input and output connections:**
  - Input from **Add Metadata**.
  - No downstream output.
- **Version-specific requirements:**
  - Uses node type version `4.5`.
- **Edge cases or potential failure types:**
  - Google Sheets OAuth2 authorization issues.
  - Sheet or document not selected.
  - Auto-mapping fails silently for fields that do not match existing column headers.
  - Additional fields may be ignored if the sheet lacks matching headers.
  - Data types are inserted as strings where appropriate.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/visual annotation |  |  | ## Gmail to DocuPipe to Google Sheets<br>Automatically extract structured data from Gmail attachments using [DocuPipe](https://docupipe.ai) AI and save results to Google Sheets.<br>### Who is this for?<br>Teams that receive documents via email (invoices, receipts, contracts) and want structured data automatically extracted and added to a spreadsheet - without manual data entry.<br>### How it works<br>This template contains two connected flows:<br>**Scenario 1 - Upload:** Watches Gmail for new emails with attachments, labels them as "Processing", uploads the attachment to DocuPipe for extraction, and saves a backup copy to Google Drive.<br>**Scenario 2 - Process & Save:** When DocuPipe finishes extracting, the webhook fires, results are fetched, processed into a flat row format, enriched with metadata, and appended to your Google Sheet.<br>### How to set up<br>1. Connect your **Gmail** account<br>2. Create a Gmail label called **"DocuPipe - Processing"** (or customize the label name)<br>3. Sign up at [docupipe.ai](https://docupipe.ai), then get your **DocuPipe API key** at [app.docupipe.ai/settings/general](https://app.docupipe.ai/settings/general)<br>4. Select an extraction **schema** in the Upload node<br>5. Connect your **Google Drive** account and select a backup folder<br>6. Connect your **Google Sheets** account and select your spreadsheet<br>7. Ensure your sheet's column headers match the schema field names<br>8. Activate this workflow<br>### Requirements<br>- A [DocuPipe](https://docupipe.ai) account with an API key<br>- A Gmail account<br>- A Google Drive folder for backups<br>- A Google Sheets spreadsheet<br>- An extraction schema configured in DocuPipe<br>**Note:** Requires the [DocuPipe community node](https://www.npmjs.com/package/n8n-nodes-docupipe). Install via Settings > Community Nodes. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/visual annotation |  |  | ## Scenario 1 - Upload Gmail Attachments to DocuPipe<br>New emails with attachments are detected via Gmail, labeled as "Processing", and the attachment is uploaded to DocuPipe for AI-powered extraction. A backup copy of the attachment is also saved to Google Drive. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/visual annotation |  |  | ## Scenario 2 - Process Extraction Results & Save to Google Sheets<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into a flat row-friendly format, enriched with metadata (document name, timestamp), and appended as a new row in your spreadsheet. |
| New Email with Attachment | n8n-nodes-base.gmailTrigger | Poll Gmail for messages with attachments and download them |  | Label as Processing | ## Scenario 1 - Upload Gmail Attachments to DocuPipe<br>New emails with attachments are detected via Gmail, labeled as "Processing", and the attachment is uploaded to DocuPipe for AI-powered extraction. A backup copy of the attachment is also saved to Google Drive. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Label as Processing | n8n-nodes-base.gmail | Add a Gmail label to mark the message as in processing | New Email with Attachment | Upload Attachment & Extract Data | ## Scenario 1 - Upload Gmail Attachments to DocuPipe<br>New emails with attachments are detected via Gmail, labeled as "Processing", and the attachment is uploaded to DocuPipe for AI-powered extraction. A backup copy of the attachment is also saved to Google Drive. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Upload Attachment & Extract Data | n8n-nodes-docupipe.docuPipe | Upload the first email attachment to DocuPipe and start extraction | Label as Processing | Save Attachment to Drive | ## Scenario 1 - Upload Gmail Attachments to DocuPipe<br>New emails with attachments are detected via Gmail, labeled as "Processing", and the attachment is uploaded to DocuPipe for AI-powered extraction. A backup copy of the attachment is also saved to Google Drive. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Save Attachment to Drive | n8n-nodes-base.googleDrive | Save a backup copy of the attachment to Google Drive | Upload Attachment & Extract Data |  | ## Scenario 1 - Upload Gmail Attachments to DocuPipe<br>New emails with attachments are detected via Gmail, labeled as "Processing", and the attachment is uploaded to DocuPipe for AI-powered extraction. A backup copy of the attachment is also saved to Google Drive. The extraction runs asynchronously - results arrive via webhook in Scenario 2. |
| Extraction Complete | n8n-nodes-docupipe.docuPipeTrigger | Receive DocuPipe completion webhook events |  | Get Extracted Data | ## Scenario 2 - Process Extraction Results & Save to Google Sheets<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into a flat row-friendly format, enriched with metadata (document name, timestamp), and appended as a new row in your spreadsheet. |
| Get Extracted Data | n8n-nodes-docupipe.docuPipe | Fetch the extraction result from DocuPipe | Extraction Complete | Process Extraction Result | ## Scenario 2 - Process Extraction Results & Save to Google Sheets<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into a flat row-friendly format, enriched with metadata (document name, timestamp), and appended as a new row in your spreadsheet. |
| Process Extraction Result | n8n-nodes-base.code | Flatten extraction output into a spreadsheet-friendly object | Get Extracted Data | Add Metadata | ## Scenario 2 - Process Extraction Results & Save to Google Sheets<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into a flat row-friendly format, enriched with metadata (document name, timestamp), and appended as a new row in your spreadsheet. |
| Add Metadata | n8n-nodes-base.set | Add document name and extraction timestamp | Process Extraction Result | Append Row to Spreadsheet | ## Scenario 2 - Process Extraction Results & Save to Google Sheets<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into a flat row-friendly format, enriched with metadata (document name, timestamp), and appended as a new row in your spreadsheet. |
| Append Row to Spreadsheet | n8n-nodes-base.googleSheets | Append extracted data to Google Sheets | Add Metadata |  | ## Scenario 2 - Process Extraction Results & Save to Google Sheets<br>When DocuPipe completes extraction, the webhook triggers this flow. The extracted data is fetched, processed into a flat row-friendly format, enriched with metadata (document name, timestamp), and appended as a new row in your spreadsheet. |

---

# 4. Reproducing the Workflow from Scratch

1. **Prepare prerequisites**
   - Use a self-hosted n8n instance.
   - Install the DocuPipe community node from **Settings > Community Nodes**:
     - Package: `n8n-nodes-docupipe`
   - Create or confirm:
     - A Gmail account
     - A Gmail label named **DocuPipe - Processing**
     - A DocuPipe account and API key
     - A Google Drive folder for backups
     - A Google Sheets spreadsheet with header columns matching DocuPipe schema field names
   - Also add columns such as:
     - `Document Name`
     - `Extracted At`

2. **Create a new workflow**
   - Give it the title: **Extract data from Gmail attachments to Google Sheets using DocuPipe**.

3. **Add optional documentation sticky notes**
   - Add one general sticky note summarizing the workflow purpose and setup requirements.
   - Add one sticky note above the Gmail/DocuPipe upload flow labeled **Scenario 1 - Upload Gmail Attachments to DocuPipe**.
   - Add one sticky note above the webhook/result-processing flow labeled **Scenario 2 - Process Extraction Results & Save to Google Sheets**.

4. **Create the first entry node: Gmail Trigger**
   - Add node: **Gmail Trigger**
   - Name it: **New Email with Attachment**
   - Configure:
     - Filter: **Has Attachments = true**
     - Options: **Download Attachments = true**
     - Poll time: **Every minute**
   - Connect Gmail OAuth2 credentials.

5. **Add Gmail message labeling**
   - Add node: **Gmail**
   - Name it: **Label as Processing**
   - Configure:
     - Resource: **Message**
     - Operation: **Add Labels**
     - Message ID: `={{ $json.id }}`
     - Label IDs / label name: `DocuPipe - Processing`
   - Use the same Gmail OAuth2 credentials.
   - Connect **New Email with Attachment -> Label as Processing**.

6. **Add DocuPipe upload/extraction**
   - Add node: **DocuPipe**
   - Name it: **Upload Attachment & Extract Data**
   - Configure:
     - Resource: **Extraction**
     - Operation: **Upload and Extract**
     - Input mode: **Binary**
     - Binary property name: `attachment_0`
     - Schema: choose your DocuPipe extraction schema
   - Connect DocuPipe API credentials.
   - Connect **Label as Processing -> Upload Attachment & Extract Data**.

7. **Add Google Drive backup upload**
   - Add node: **Google Drive**
   - Name it: **Save Attachment to Drive**
   - Configure:
     - Operation: **Upload**
     - Destination folder: choose your backup folder
     - File name: `={{ $('New Email with Attachment').item.json.subject }}`
   - Connect Google Drive OAuth2 credentials.
   - Connect **Upload Attachment & Extract Data -> Save Attachment to Drive**.

8. **Create the second entry node: DocuPipe Trigger**
   - Add node: **DocuPipe Trigger**
   - Name it: **Extraction Complete**
   - Configure:
     - Event: `standardization.processed.success`
   - Connect DocuPipe API credentials.
   - Ensure your n8n instance is reachable by DocuPipe so webhook events can be delivered.

9. **Add DocuPipe result retrieval**
   - Add node: **DocuPipe**
   - Name it: **Get Extracted Data**
   - Configure:
     - Resource: **Extraction**
     - Operation: **Get Result**
     - Standardization ID: `={{ $json.standardizationId }}`
   - Connect **Extraction Complete -> Get Extracted Data**.

10. **Add a Code node to flatten the result**
    - Add node: **Code**
    - Name it: **Process Extraction Result**
    - Paste this logic:
      ```javascript
      const data = $input.first().json;
      const result = data.result || {};
      const output = {};

      for (const [key, value] of Object.entries(result)) {
        if (Array.isArray(value)) {
          output[key] = JSON.stringify(value);
        } else if (typeof value === 'object' && value !== null) {
          output[key] = JSON.stringify(value);
        } else {
          output[key] = value;
        }
      }

      return [{ json: output }];
      ```
    - Connect **Get Extracted Data -> Process Extraction Result**.

11. **Add metadata fields**
    - Add node: **Set**
    - Name it: **Add Metadata**
    - Configure it to **include all input fields**.
    - Add assignments:
      - `Document Name` = `={{ $('Get Extracted Data').item.json.documentName }}`
      - `Extracted At` = `={{ $now.toISO() }}`
    - Connect **Process Extraction Result -> Add Metadata**.

12. **Append the result to Google Sheets**
    - Add node: **Google Sheets**
    - Name it: **Append Row to Spreadsheet**
    - Configure:
      - Select your spreadsheet document
      - Select the target sheet/tab
      - Column mapping mode: **Auto-map input data**
    - Connect Google Sheets OAuth2 credentials.
    - Connect **Add Metadata -> Append Row to Spreadsheet**.

13. **Verify sheet headers**
    - Make sure your Google Sheet has columns matching:
      - Every expected DocuPipe extraction field
      - `Document Name`
      - `Extracted At`
   - Auto-mapping depends on exact or compatible column naming.

14. **Test Scenario 1**
    - Send an email with an attachment to the connected Gmail account.
    - Confirm:
      - The Gmail trigger detects it
      - The message gets labeled
      - The file uploads to DocuPipe
      - A backup copy is stored in Google Drive

15. **Test Scenario 2**
    - Wait for DocuPipe to complete extraction.
    - Confirm:
      - The webhook trigger receives the event
      - `Get Extracted Data` returns a result
      - The Code node outputs a flattened object
      - Metadata is added
      - A row appears in Google Sheets

16. **Activate the workflow**
    - Once both scenarios work correctly, activate the workflow.

## Credential configuration summary
- **Gmail OAuth2**
  - Required by:
    - New Email with Attachment
    - Label as Processing
- **DocuPipe API**
  - Required by:
    - Upload Attachment & Extract Data
    - Extraction Complete
    - Get Extracted Data
- **Google Drive OAuth2**
  - Required by:
    - Save Attachment to Drive
- **Google Sheets OAuth2**
  - Required by:
    - Append Row to Spreadsheet

## Rebuild constraints and behavioral notes
- The workflow has **two entry points**:
  - Gmail polling trigger
  - DocuPipe completion trigger
- The DocuPipe extraction is **asynchronous**.
- The upload path does **not** directly continue into Google Sheets; it ends after backup upload.
- The process-and-save path begins only when DocuPipe sends its completion event.
- Only the first Gmail attachment is processed because the binary property is hardcoded to `attachment_0`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DocuPipe website | https://docupipe.ai |
| DocuPipe API key settings page | https://app.docupipe.ai/settings/general |
| DocuPipe community node package for n8n | https://www.npmjs.com/package/n8n-nodes-docupipe |
| Gmail label expected by default | `DocuPipe - Processing` |
| Important deployment requirement | Self-hosted n8n is required for community nodes in this setup |
| Spreadsheet preparation note | Sheet headers should match DocuPipe schema field names, plus `Document Name` and `Extracted At` |
| Processing limitation | The current workflow processes only the first attachment from each email via `attachment_0` |