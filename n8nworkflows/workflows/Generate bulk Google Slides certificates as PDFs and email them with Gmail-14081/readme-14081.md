Generate bulk Google Slides certificates as PDFs and email them with Gmail

https://n8nworkflows.xyz/workflows/generate-bulk-google-slides-certificates-as-pdfs-and-email-them-with-gmail-14081


# Generate bulk Google Slides certificates as PDFs and email them with Gmail

# 1. Workflow Overview

This workflow bulk-generates personalized certificate PDFs from a Google Slides template, emails each certificate through Gmail, stores the PDF in Google Drive, marks the recipient row as processed in Google Sheets, deletes the temporary Google Slides copy, and loops until all unsent rows are handled.

Typical use cases include:
- Course completion certificates
- Event or webinar participation certificates
- Training program certificates
- Any bulk personalized document generation where a Google Slides template is suitable

The workflow is organized into the following logical blocks:

## 1.1 Trigger and Data Intake
The workflow starts manually and reads recipient records from a Google Sheet. It filters for rows that have not yet been marked as sent.

## 1.2 Batch Loop Control
The returned rows are processed one at a time using a batch loop. A wait step is used between iterations to reduce the chance of hitting Google API limits.

## 1.3 Template Duplication and Personalization
For each recipient, the workflow copies a Google Slides certificate template and replaces placeholders such as recipient name, date, and certificate number.

## 1.4 PDF Export and Email Delivery
The personalized slide deck is exported as a PDF and emailed as an attachment to the recipient via Gmail.

## 1.5 Archival, Status Update, and Cleanup
The PDF is saved to Google Drive, the source row in Google Sheets is marked as processed, and the temporary Google Slides file is deleted.

## 1.6 Loop Synchronization
Parallel branches are merged so the workflow only continues to the wait-and-next-item step after both emailing and cleanup-related tasks are complete.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Data Intake

### Overview
This block manually starts the workflow and retrieves the list of recipients from Google Sheets. Only rows without a value in the `Sent` column are intended to be processed.

### Nodes Involved
- When clicking 'Test workflow'
- Read File

### Node Details

#### When clicking 'Test workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for test or operator-initiated runs.
- **Configuration choices:** No parameters are configured; execution starts when the user clicks the test button.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none
  - Output: `Read File`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None in normal use; this node is only available for manual execution and is not a scheduled or webhook-based trigger.
- **Sub-workflow reference:** None.

#### Read File
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a Google Sheets document.
- **Configuration choices:**  
  - Connected to the spreadsheet named **Certifications**
  - Uses sheet/tab **Foglio1** (`gid=0`)
  - Uses `filtersUI` with `lookupColumn: "Sent"` to target rows based on the `Sent` column
- **Key expressions or variables used:** None directly in parameters, but downstream nodes use fields from each returned row:
  - `N.`
  - `First Name`
  - `Last Name`
  - `Email`
  - `Sent`
  - `row_number`
- **Input and output connections:**  
  - Input: `When clicking 'Test workflow'`
  - Output: `Process One at a Time`
- **Version-specific requirements:** Type version `4.5`; requires valid Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - OAuth authorization failure
  - Spreadsheet or sheet not found
  - Incorrect filter behavior if `Sent` values are inconsistent
  - Missing required columns or renamed headers
  - Empty result set, in which case no items proceed to the batch loop
- **Sub-workflow reference:** None.

---

## 2.2 Batch Loop Control

### Overview
This block ensures recipients are processed sequentially rather than all at once. It also acts as the loop anchor for processing the next row after the wait step.

### Nodes Involved
- Process One at a Time
- Wait 10s

### Node Details

#### Process One at a Time
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through input items one batch at a time.
- **Configuration choices:**  
  - Processes items in batches
  - `reset: false`, so the node continues through the dataset across loop iterations
  - In this design, it effectively handles one recipient per cycle
- **Key expressions or variables used:** Downstream nodes refer back to this node’s current item using expressions such as:
  - `$('Process One at a Time').item.json['First Name']`
  - `$('Process One at a Time').item.json['Last Name']`
  - `$('Process One at a Time').item.json['Email']`
  - `$('Process One at a Time').item.json['N.']`
  - `$('Process One at a Time').item.json['row_number']`
- **Input and output connections:**  
  - Input 1: `Read File`
  - Input 2 / loop-back: `Wait 10s`
  - Output 0: no connection shown for completion branch
  - Output 1: `Copy Slides Template` for current item processing
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - If no rows are returned, the loop body never runs
  - If row data is malformed, downstream expressions can fail or produce incorrect content
  - Users unfamiliar with Split in Batches may misread output indexing; this workflow uses the iterative branch
- **Sub-workflow reference:** None.

#### Wait 10s
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution between loop iterations.
- **Configuration choices:**  
  - Wait duration: `10` seconds
  - Webhook ID exists internally for resumed executions
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Merge`
  - Output: `Process One at a Time`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Long runs can take significant total time for large recipient lists
  - Resumption behavior depends on n8n execution persistence settings
  - If the instance restarts and wait/resume storage is not configured correctly, resumed execution may fail
- **Sub-workflow reference:** None.

---

## 2.3 Template Duplication and Personalization

### Overview
This block creates a new Google Slides file from a template and replaces placeholders with recipient-specific values. It is the core document generation stage.

### Nodes Involved
- Copy Slides Template
- Replace Template Placeholders

### Node Details

#### Copy Slides Template
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls the Google Drive API to copy an existing Google Slides template file.
- **Configuration choices:**  
  - HTTP method: `POST`
  - URL: `https://www.googleapis.com/drive/v3/files/xxx/copy`
  - Auth: Google Drive OAuth2 predefined credential
  - Sends JSON body with dynamic file name:
    - `Certificate_` + recipient certificate number
- **Key expressions or variables used:**  
  - `{{ JSON.stringify({ name: 'Certificate_' + $json['N.'] }) }}`
  - Uses current batch item’s `N.` field
- **Input and output connections:**  
  - Input: `Process One at a Time`
  - Output: `Replace Template Placeholders`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - The `xxx` template file ID must be manually replaced
  - OAuth scope must allow file copy on Google Drive
  - Invalid template file ID returns 404 or permission errors
  - If `N.` is empty, the generated file name may be incomplete
- **Sub-workflow reference:** None.

#### Replace Template Placeholders
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls the Google Slides API `batchUpdate` endpoint to replace text placeholders in the copied presentation.
- **Configuration choices:**  
  - HTTP method: `POST`
  - URL uses copied presentation ID:
    - `https://slides.googleapis.com/v1/presentations/{{ $json.id }}:batchUpdate`
  - Auth: Google Slides OAuth2 predefined credential
  - Sends a JSON body with three `replaceAllText` requests
- **Key expressions or variables used:**  
  - Presentation ID from previous node: `{{ $json.id }}`
  - Name placeholder:
    - `{{ $('Process One at a Time').item.json['First Name'] }} {{ $('Process One at a Time').item.json['Last Name'] }}`
  - Date placeholder:
    - `{{ $now.format('dd/LL/yyyy') }}`
  - Certificate number placeholder:
    - `#{{ $('Process One at a Time').item.json['N.'] }}`
  - Expected placeholders in template:
    - `[Name]`
    - `[Date]`
    - `[N]`
- **Input and output connections:**  
  - Input: `Copy Slides Template`
  - Output: `Export to PDF`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Template must contain placeholders exactly as expected
  - Missing `First Name`, `Last Name`, or `N.` values lead to incomplete personalization
  - Date formatting depends on n8n expression support for `$now.format(...)`
  - Permission mismatch between Drive and Slides credentials can block update calls
- **Sub-workflow reference:** None.

---

## 2.4 PDF Export and Email Delivery

### Overview
This block converts the personalized Google Slides file into a PDF binary and sends it to the recipient as an email attachment through Gmail.

### Nodes Involved
- Export to PDF
- Send a message

### Node Details

#### Export to PDF
- **Type and technical role:** `n8n-nodes-base.httpRequest`; exports the copied Google Slides file to PDF via the Google Drive export endpoint.
- **Configuration choices:**  
  - URL:
    - `https://www.googleapis.com/drive/v3/files/{{ $('Copy Slides Template').item.json.id }}/export?mimeType=application/pdf`
  - Authentication: Google Slides OAuth2 predefined credential
  - Response format configured as **file**, so the output becomes binary data
- **Key expressions or variables used:**  
  - `{{ $('Copy Slides Template').item.json.id }}`
- **Input and output connections:**  
  - Input: `Replace Template Placeholders`
  - Output 1: `Save PDF to Drive`
  - Output 2: `Send a message`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - File export can fail if the copied file is not ready yet, though the linear flow typically avoids this
  - OAuth scope must support export/download
  - Binary output handling depends on workflow binary mode; this workflow uses `binaryMode: separate`
- **Sub-workflow reference:** None.

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an HTML email with the exported PDF attached.
- **Configuration choices:**  
  - Recipient:
    - `={{$('Process One at a Time').item.json['Email']}}`
  - Subject:
    - `Your n8n Certificate`
  - Message body:
    - Rich HTML congratulatory email
    - Includes recipient full name dynamically
    - Includes branding for n3w Italia and link to `https://n3w.it`
    - Includes CTA button to LinkedIn
  - Attachments:
    - Binary attachment enabled through `attachmentsUi.attachmentsBinary`
- **Key expressions or variables used:**  
  - Recipient email:
    - `{{$('Process One at a Time').item.json['Email']}}`
  - Recipient full name in body:
    - `{{$('Process One at a Time').item.json['First Name']}} {{$('Process One at a Time').item.json['Last Name']}}`
- **Input and output connections:**  
  - Input: `Export to PDF`
  - Output: `Merge`
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials with permission to send email.
- **Edge cases or potential failure types:**  
  - Invalid or blank email address
  - Gmail quota/rate limit issues
  - Attachment may fail if the binary property name is not what Gmail expects in a specific environment
  - HTML email may render differently across clients
- **Sub-workflow reference:** None.

---

## 2.5 Archival, Status Update, and Cleanup

### Overview
This block stores the generated PDF in Google Drive, marks the spreadsheet row as sent, and deletes the temporary Google Slides copy to avoid clutter.

### Nodes Involved
- Save PDF to Drive
- Mark as Processed
- Delete Temp Slide

### Node Details

#### Save PDF to Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the PDF binary to a target Google Drive folder.
- **Configuration choices:**  
  - File name:
    - `certificate_{{ $('Process One at a Time').item.json['N.'] }}.pdf`
  - Drive: `My Drive`
  - Folder: Google Drive folder named `n8n`
  - `alwaysOutputData: true` so the flow continues even if upload returns limited data
- **Key expressions or variables used:**  
  - `{{ $('Process One at a Time').item.json['N.'] }}`
- **Input and output connections:**  
  - Input: `Export to PDF`
  - Output: `Mark as Processed`
- **Version-specific requirements:** Type version `3`; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Upload failure if the binary input is missing
  - Folder permissions issue
  - Duplicate file names may create multiple files or collision behavior depending on Drive API handling
- **Sub-workflow reference:** None.

#### Mark as Processed
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates the source row to mark the item as already sent.
- **Configuration choices:**  
  - Operation: `update`
  - Spreadsheet: **Certifications**
  - Sheet: **Foglio1**
  - Matching column: `row_number`
  - Writes:
    - `Sent = x`
    - `row_number = current row_number`
  - Uses explicit schema mapping
- **Key expressions or variables used:**  
  - `={{ $('Process One at a Time').item.json.row_number }}`
- **Input and output connections:**  
  - Input: `Save PDF to Drive`
  - Output: `Delete Temp Slide`
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or potential failure types:**  
  - `row_number` must exist in input data
  - If rows are reordered externally between read and update, row-number-based updates may target the wrong row in some scenarios
  - Missing write permission on sheet
- **Sub-workflow reference:** None.

#### Delete Temp Slide
- **Type and technical role:** `n8n-nodes-base.httpRequest`; deletes the temporary copied Google Slides file using the Google Drive API.
- **Configuration choices:**  
  - HTTP method: `DELETE`
  - URL:
    - `https://www.googleapis.com/drive/v3/files/{{ $('Copy Slides Template').item.json.id }}`
  - Auth: Google Slides OAuth2 predefined credential
- **Key expressions or variables used:**  
  - `{{ $('Copy Slides Template').item.json.id }}`
- **Input and output connections:**  
  - Input: `Mark as Processed`
  - Output: `Merge`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Deletion may fail if credentials do not have delete permission
  - If earlier steps fail, the temporary slide may remain orphaned in Drive
  - Uses Google Drive deletion endpoint with Slides credential; this is valid only if scopes cover Drive file deletion
- **Sub-workflow reference:** None.

---

## 2.6 Loop Synchronization

### Overview
This block waits for both the email branch and the cleanup branch to complete before pausing and moving to the next recipient.

### Nodes Involved
- Merge

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`; synchronizes two branches.
- **Configuration choices:**  
  - Default merge behavior is used
  - One input comes from email delivery, the other from cleanup completion
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input 1: `Send a message`
  - Input 2: `Delete Temp Slide`
  - Output: `Wait 10s`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**  
  - If one branch fails, the merge is never reached and the loop stalls
  - Merge behavior should be validated when modifying the workflow, especially if item counts diverge between branches
- **Sub-workflow reference:** None.

---

## 2.7 Documentation / Sticky Notes

### Overview
These nodes do not execute business logic. They document setup requirements, workflow purpose, and operational steps directly on the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation for the template and PDF generation area.
- **Configuration choices:** Explains replacing `xxx` with the Google Slides template file ID and notes automatic PDF conversion.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation for the Google Sheets source setup.
- **Configuration choices:** Explains the spreadsheet clone and bulk certificate use case.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; high-level workflow description and setup checklist.
- **Configuration choices:** Documents end-to-end behavior, required placeholders, credentials, Drive folder setup, and Gmail attachment review.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual note for the email block.
- **Configuration choices:** States that certificates are automatically sent to recipients.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual note for storage and cleanup behavior.
- **Configuration choices:** Explains saving to Google Drive and deletion of temporary Slides files.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Test workflow' | Manual Trigger | Manual start of the workflow |  | Read File | ## Bulk certificate generator with Google Slides<br>### How it works<br>This workflow bulk-generates personalized certificates by reading recipient data from Google Sheets and processing only rows that have not been marked as sent. For each recipient, it creates a copy of a Google Slides certificate template, replaces placeholders such as name, date, and certificate number, exports the personalized slide deck as a PDF, saves it to Google Drive, emails it through Gmail, updates the source sheet to mark the row as processed, and deletes the temporary Slides file. A batch loop plus a 10-second wait keeps processing stable and helps avoid Google API rate limits.<br>### Setup steps<br>Set up a Google Sheet with recipient data and columns for certificate number, name, email, and sent status, then point the workflow’s Google Sheets nodes to that document. Create a Google Slides template with the exact placeholders `[Name]`, `[Date]`, and `[N]`, update the template file ID in the copy step, and choose the Google Drive folder where generated PDFs will be stored. After that, connect valid Google OAuth2 credentials for Drive, Slides, Sheets, and Gmail in n8n, then review the Gmail node content and confirm the PDF attachment is sourced from the export step. |
| Read File | Google Sheets | Read recipient rows from the spreadsheet | When clicking 'Test workflow' | Process One at a Time | ## STEP 1 - DB<br>Clone [this Sheet](https://docs.google.com/spreadsheets/d/1fHcfilCPpI4aAiJFQC3cPIAdJMmctMubz3kcOeai1yA/edit?usp=sharing). The workflow can generate and send personalized certificates for many recipients in bulk, making it ideal for courses, events, webinars, or training programs. |
| Process One at a Time | Split In Batches | Sequentially process recipients one by one | Read File, Wait 10s | Copy Slides Template |  |
| Copy Slides Template | HTTP Request | Copy the Google Slides certificate template | Process One at a Time | Replace Template Placeholders | ## STEP 2 - Template<br>Replace xxx with the Google Slides base template file id [like this](https://docs.google.com/presentation/d/1YWYsNfq4FeHP03MSYFz1cobVczBHWxwEW95BE4w0FHc/edit?slide=id.p#slide=id.p).<br>After personalization, the workflow automatically converts the generated Google Slides document into a **PDF file** |
| Replace Template Placeholders | HTTP Request | Replace placeholders in the copied presentation | Copy Slides Template | Export to PDF | ## STEP 2 - Template<br>Replace xxx with the Google Slides base template file id [like this](https://docs.google.com/presentation/d/1YWYsNfq4FeHP03MSYFz1cobVczBHWxwEW95BE4w0FHc/edit?slide=id.p#slide=id.p).<br>After personalization, the workflow automatically converts the generated Google Slides document into a **PDF file** |
| Export to PDF | HTTP Request | Export the personalized presentation as PDF binary | Replace Template Placeholders | Save PDF to Drive, Send a message | ## STEP 2 - Template<br>Replace xxx with the Google Slides base template file id [like this](https://docs.google.com/presentation/d/1YWYsNfq4FeHP03MSYFz1cobVczBHWxwEW95BE4w0FHc/edit?slide=id.p#slide=id.p).<br>After personalization, the workflow automatically converts the generated Google Slides document into a **PDF file** |
| Send a message | Gmail | Email the certificate PDF to the recipient | Export to PDF | Merge | ## STEP 3 - Email<br>Certificates are automatically sent to recipients |
| Save PDF to Drive | Google Drive | Upload the generated PDF to Google Drive | Export to PDF | Mark as Processed | ## STEP 4 - Save to Google Drive<br>Temporary Google Slides files created during the generation process are automatically deleted after use, keeping **Google Drive clean and organized**. |
| Mark as Processed | Google Sheets | Mark the spreadsheet row as sent | Save PDF to Drive | Delete Temp Slide | ## STEP 4 - Save to Google Drive<br>Temporary Google Slides files created during the generation process are automatically deleted after use, keeping **Google Drive clean and organized**. |
| Delete Temp Slide | HTTP Request | Delete the temporary copied Slides file | Mark as Processed | Merge | ## STEP 4 - Save to Google Drive<br>Temporary Google Slides files created during the generation process are automatically deleted after use, keeping **Google Drive clean and organized**. |
| Merge | Merge | Synchronize email and cleanup branches | Send a message, Delete Temp Slide | Wait 10s | ## STEP 4 - Save to Google Drive<br>Temporary Google Slides files created during the generation process are automatically deleted after use, keeping **Google Drive clean and organized**. |
| Wait 10s | Wait | Delay before requesting the next batch item | Merge | Process One at a Time | ## STEP 4 - Save to Google Drive<br>Temporary Google Slides files created during the generation process are automatically deleted after use, keeping **Google Drive clean and organized**. |
| Sticky Note | Sticky Note | Canvas documentation for template configuration |  |  | ## STEP 2 - Template<br>Replace xxx with the Google Slides base template file id [like this](https://docs.google.com/presentation/d/1YWYsNfq4FeHP03MSYFz1cobVczBHWxwEW95BE4w0FHc/edit?slide=id.p#slide=id.p).<br>After personalization, the workflow automatically converts the generated Google Slides document into a **PDF file** |
| Sticky Note1 | Sticky Note | Canvas documentation for spreadsheet setup |  |  | ## STEP 1 - DB<br>Clone [this Sheet](https://docs.google.com/spreadsheets/d/1fHcfilCPpI4aAiJFQC3cPIAdJMmctMubz3kcOeai1yA/edit?usp=sharing). The workflow can generate and send personalized certificates for many recipients in bulk, making it ideal for courses, events, webinars, or training programs. |
| Sticky Note2 | Sticky Note | Canvas overview and setup instructions |  |  | ## Bulk certificate generator with Google Slides<br>### How it works<br>This workflow bulk-generates personalized certificates by reading recipient data from Google Sheets and processing only rows that have not been marked as sent. For each recipient, it creates a copy of a Google Slides certificate template, replaces placeholders such as name, date, and certificate number, exports the personalized slide deck as a PDF, saves it to Google Drive, emails it through Gmail, updates the source sheet to mark the row as processed, and deletes the temporary Slides file. A batch loop plus a 10-second wait keeps processing stable and helps avoid Google API rate limits.<br>### Setup steps<br>Set up a Google Sheet with recipient data and columns for certificate number, name, email, and sent status, then point the workflow’s Google Sheets nodes to that document. Create a Google Slides template with the exact placeholders `[Name]`, `[Date]`, and `[N]`, update the template file ID in the copy step, and choose the Google Drive folder where generated PDFs will be stored. After that, connect valid Google OAuth2 credentials for Drive, Slides, Sheets, and Gmail in n8n, then review the Gmail node content and confirm the PDF attachment is sourced from the export step. |
| Sticky Note3 | Sticky Note | Canvas documentation for email delivery |  |  | ## STEP 3 - Email<br>Certificates are automatically sent to recipients |
| Sticky Note4 | Sticky Note | Canvas documentation for Drive storage and cleanup |  |  | ## STEP 4 - Save to Google Drive<br>Temporary Google Slides files created during the generation process are automatically deleted after use, keeping **Google Drive clean and organized**. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Bulk certificate generator with Google Slides`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Leave default configuration.
   - This will be the workflow entry point for testing.

3. **Prepare the Google Sheet**
   - Create a spreadsheet with columns at minimum:
     - `N.`
     - `First Name`
     - `Last Name`
     - `Email`
     - `Sent`
   - Make sure row headers are exactly spelled as above.
   - Keep `Sent` blank for rows that still need processing.
   - Ensure the Google Sheets node can expose `row_number`, since the update step depends on it.

4. **Add a Google Sheets node to read recipients**
   - Node name: `Read File`
   - Node type: **Google Sheets**
   - Credential: Google Sheets OAuth2
   - Select your spreadsheet document.
   - Select the target sheet/tab.
   - Configure it to read rows.
   - Add a filter based on the `Sent` column so only unprocessed rows are returned.
   - Connect:
     - `Manual Trigger -> Read File`

5. **Add a Split In Batches node**
   - Node name: `Process One at a Time`
   - Node type: **Split In Batches**
   - Keep default batch size behavior appropriate for one item at a time.
   - Set `reset` to `false`.
   - Connect:
     - `Read File -> Process One at a Time`

6. **Create or prepare the Google Slides template**
   - Create a Google Slides presentation to serve as the certificate template.
   - Put these exact placeholders somewhere in the slide content:
     - `[Name]`
     - `[Date]`
     - `[N]`
   - Copy the file ID from the template URL:
     - Example format: `https://docs.google.com/presentation/d/FILE_ID/edit`

7. **Add an HTTP Request node to copy the template**
   - Node name: `Copy Slides Template`
   - Node type: **HTTP Request**
   - Authentication: **Predefined Credential Type**
   - Credential type: **Google Drive OAuth2 API**
   - Method: `POST`
   - URL:
     - `https://www.googleapis.com/drive/v3/files/FILE_ID/copy`
   - Replace `FILE_ID` with your actual Google Slides template ID.
   - Enable JSON body.
   - Set body to:
     - Name the copied file dynamically as `Certificate_` plus the `N.` field.
   - Equivalent expression logic:
     - Use the current item’s certificate number from the sheet.
   - Connect:
     - `Process One at a Time -> Copy Slides Template`

8. **Add an HTTP Request node to replace placeholders**
   - Node name: `Replace Template Placeholders`
   - Node type: **HTTP Request**
   - Authentication: **Predefined Credential Type**
   - Credential type: **Google Slides OAuth2 API**
   - Method: `POST`
   - URL:
     - `https://slides.googleapis.com/v1/presentations/{{ $json.id }}:batchUpdate`
   - Send JSON body with three `replaceAllText` operations:
     - Replace `[Name]` with `First Name + Last Name`
     - Replace `[Date]` with current date
     - Replace `[N]` with `#` plus `N.`
   - Use expressions referencing the current batch item:
     - `$('Process One at a Time').item.json['First Name']`
     - `$('Process One at a Time').item.json['Last Name']`
     - `$('Process One at a Time').item.json['N.']`
   - For the date, use an n8n date expression such as:
     - `$now.format('dd/LL/yyyy')`
   - Connect:
     - `Copy Slides Template -> Replace Template Placeholders`

9. **Add an HTTP Request node to export the presentation to PDF**
   - Node name: `Export to PDF`
   - Node type: **HTTP Request**
   - Authentication: **Predefined Credential Type**
   - Credential type: **Google Slides OAuth2 API** or a Google credential with export access
   - Method: `GET`
   - URL:
     - `https://www.googleapis.com/drive/v3/files/{{ $('Copy Slides Template').item.json.id }}/export?mimeType=application/pdf`
   - Set response format to **File**.
   - This must produce binary output for downstream attachment/upload use.
   - Connect:
     - `Replace Template Placeholders -> Export to PDF`

10. **Add a Google Drive node to save the PDF**
    - Node name: `Save PDF to Drive`
    - Node type: **Google Drive**
    - Credential: Google Drive OAuth2
    - Configure upload/create file behavior using incoming binary data.
    - Select:
      - Drive: `My Drive` or your target drive
      - Folder: destination folder for generated certificates
    - File name expression:
      - `certificate_{{ $('Process One at a Time').item.json['N.'] }}.pdf`
    - Enable `Always Output Data` if you want the workflow to continue even when the node returns minimal metadata.
    - Connect:
      - `Export to PDF -> Save PDF to Drive`

11. **Add a Gmail node to send the certificate**
    - Node name: `Send a message`
    - Node type: **Gmail**
    - Credential: Gmail OAuth2
    - To:
      - `{{$('Process One at a Time').item.json['Email']}}`
    - Subject:
      - `Your n8n Certificate`
    - Message:
      - Paste your HTML email body
      - Include dynamic recipient name fields
    - Enable binary attachments and choose the binary produced by `Export to PDF`.
    - Verify the binary property name in execution data if attachment mapping needs adjustment.
    - Connect:
      - `Export to PDF -> Send a message`

12. **Add a Google Sheets node to mark the row as sent**
    - Node name: `Mark as Processed`
    - Node type: **Google Sheets**
    - Credential: Google Sheets OAuth2
    - Operation: `Update`
    - Spreadsheet: same source spreadsheet
    - Sheet: same source tab
    - Match by column:
      - `row_number`
    - Set mapped values:
      - `Sent = x`
      - `row_number = {{ $('Process One at a Time').item.json.row_number }}`
    - If using manual field mapping, include your relevant schema fields.
    - Connect:
      - `Save PDF to Drive -> Mark as Processed`

13. **Add an HTTP Request node to delete the temporary slide**
    - Node name: `Delete Temp Slide`
    - Node type: **HTTP Request**
    - Authentication: **Predefined Credential Type**
    - Credential type: credential with permission to delete the copied Google Slides file
    - Method: `DELETE`
    - URL:
      - `https://www.googleapis.com/drive/v3/files/{{ $('Copy Slides Template').item.json.id }}`
    - Connect:
      - `Mark as Processed -> Delete Temp Slide`

14. **Add a Merge node**
    - Node name: `Merge`
    - Node type: **Merge**
    - Leave default behavior unless you specifically need a different merge mode.
    - Connect:
      - `Send a message -> Merge` as input 1
      - `Delete Temp Slide -> Merge` as input 2

15. **Add a Wait node**
    - Node name: `Wait 10s`
    - Node type: **Wait**
    - Set amount to `10` seconds.
    - Connect:
      - `Merge -> Wait 10s`

16. **Close the loop back to Split In Batches**
    - Connect:
      - `Wait 10s -> Process One at a Time`
   - This allows the next sheet row to be processed after the current one finishes.

17. **Configure credentials**
    - You need working OAuth2 credentials for:
      - **Google Sheets**
      - **Google Drive**
      - **Google Slides**
      - **Gmail**
    - Make sure the authenticated Google account has access to:
      - The source spreadsheet
      - The Slides template
      - The Drive destination folder
      - Gmail send permission
    - Ensure OAuth scopes cover:
      - Sheets read/write
      - Drive copy/export/delete/upload
      - Slides update
      - Gmail send

18. **Optional but recommended: add sticky notes**
    - Add notes for:
      - Spreadsheet setup
      - Template placeholder requirements
      - Email step
      - Drive save and cleanup step

19. **Test with one sample row**
    - Put a single test row in the sheet with:
      - valid `N.`
      - valid `First Name`
      - valid `Last Name`
      - valid `Email`
      - blank `Sent`
    - Run the workflow manually.
    - Confirm:
      - a Slides copy is created
      - placeholders are replaced
      - the PDF is exported
      - the email is sent
      - the PDF is stored in Drive
      - the row gets `Sent = x`
      - the temporary slide is deleted

20. **Validate attachment behavior**
    - If the Gmail attachment does not appear:
      - inspect the binary output of `Export to PDF`
      - confirm the Gmail node is referencing the correct binary property
      - verify binary mode compatibility in your n8n instance

21. **Scale cautiously**
    - For larger recipient lists:
      - keep or increase the wait time
      - monitor Google API quotas
      - monitor Gmail sending limits
      - consider adding error handling branches for failed recipients

### Sub-workflow setup
This workflow does **not** use sub-workflows or Execute Workflow nodes. No separate child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Clone the example Google Sheet used as the recipient database. | https://docs.google.com/spreadsheets/d/1fHcfilCPpI4aAiJFQC3cPIAdJMmctMubz3kcOeai1yA/edit?usp=sharing |
| Example format for locating the Google Slides template file ID. | https://docs.google.com/presentation/d/1YWYsNfq4FeHP03MSYFz1cobVczBHWxwEW95BE4w0FHc/edit?slide=id.p#slide=id.p |
| The template must contain the exact placeholders `[Name]`, `[Date]`, and `[N]`. | Google Slides certificate template |
| Generated certificates are emailed from Gmail with a branded HTML message. | Gmail node content |
| Generated PDFs are stored in Google Drive, and temporary Slides copies are deleted afterward. | Drive archival and cleanup behavior |
| Branding referenced in the email footer: n3w Italia / n3w.it | https://n3w.it |
| The email CTA links recipients to LinkedIn for sharing their achievement. | https://www.linkedin.com |