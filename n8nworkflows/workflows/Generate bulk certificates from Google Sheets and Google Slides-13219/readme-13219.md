Generate bulk certificates from Google Sheets and Google Slides

https://n8nworkflows.xyz/workflows/generate-bulk-certificates-from-google-sheets-and-google-slides-13219


# Generate bulk certificates from Google Sheets and Google Slides

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate bulk certificates from Google Sheets and Google Slides  
**Workflow name (in JSON):** Bulk certificate generator from Google Sheets and Slides

This workflow generates personalized **PDF certificates in bulk** using:
- **Google Sheets** as the recipient/data source
- **Google Slides** as the certificate template engine (placeholders replaced per recipient)
- **Google Drive** as the storage destination for generated PDFs

It processes only rows where **Processed = FALSE**, generates a certificate per row, uploads the PDF to a Drive folder, updates the spreadsheet with the Drive link and sets **Processed = TRUE**, then deletes the temporary Slides copy. A **2-second wait** helps avoid Google API rate limits.

### Logical blocks
1. **1.1 Trigger & Data Input**
2. **1.2 Batch Loop Control (one recipient at a time)**
3. **1.3 Certificate Generation (copy template → replace placeholders → export PDF)**
4. **1.4 Save & Spreadsheet Update**
5. **1.5 Cleanup & Rate-Limit Control (delete temp file → wait → loop)**

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Data Input

**Overview:** Starts the workflow manually and fetches only recipients that have not been processed yet from Google Sheets.

**Nodes involved:**
- When clicking 'Test workflow'
- Read Unprocessed Recipients

#### Node: When clicking 'Test workflow'
- **Type / role:** Manual Trigger (starts workflow execution in n8n UI).
- **Configuration choices:** No parameters; standard manual trigger.
- **Inputs/Outputs:**  
  - Input: none  
  - Output → **Read Unprocessed Recipients**
- **Failure modes / edge cases:** None significant; only runs when manually executed.
- **Version notes:** typeVersion **1**.

#### Node: Read Unprocessed Recipients
- **Type / role:** Google Sheets node (read rows with filter).
- **Configuration choices (interpreted):**
  - Reads from a specific sheet tab (`sheetName: gid=0`).
  - Uses a **filter**: `Processed` column equals `FALSE` (string comparison).
  - Requires the spreadsheet ID set as `YOUR_GOOGLE_SHEET_ID`.
- **Key fields/variables:**
  - Filter: `lookupColumn = Processed`, `lookupValue = FALSE`.
- **Inputs/Outputs:**
  - Input ← Manual Trigger
  - Output → **Process One at a Time** (emits one item per matching row)
- **Failure modes / edge cases:**
  - Auth/permission errors (Sheets OAuth).
  - Filter mismatch if the sheet stores boolean TRUE/FALSE vs strings `"TRUE"/"FALSE"`—your sheet must be consistent.
  - Missing columns (Processed, CertificateID, etc.) will break later expressions/updates.
- **Version notes:** typeVersion **4.5**.

---

### 2.2 Batch Loop Control (one recipient at a time)

**Overview:** Ensures recipients are processed sequentially, which reduces concurrency and helps avoid Google API quota/rate-limit problems.

**Nodes involved:**
- Process One at a Time

#### Node: Process One at a Time
- **Type / role:** Split In Batches (loop controller).
- **Configuration choices (interpreted):**
  - `reset: false` so it continues batching until all input items are processed in the loop.
  - Default batch size is used (n8n commonly defaults to 1 for this node unless changed in UI); functionally it is used here to process **one row at a time**.
- **Inputs/Outputs:**
  - Input ← Read Unprocessed Recipients
  - Output (loop “current batch”) → **Copy Slides Template**
  - Output (when finished) → not connected (workflow ends when no items remain)
  - Receives loop-back input from **Wait 2s (Rate Limit)** to continue next item.
- **Key expressions used elsewhere referencing this node:**
  - `$('Process One at a Time').item.json.Name`
  - `$('Process One at a Time').item.json.Date`
  - `$('Process One at a Time').item.json['CertificateID']`
- **Failure modes / edge cases:**
  - If there are zero matching rows, downstream nodes never run.
  - If any item lacks required fields (Name/Date/CertificateID), later placeholder replacement or file naming may fail.
- **Version notes:** typeVersion **3**.

---

### 2.3 Certificate Generation (copy → replace → export)

**Overview:** Creates a per-recipient Slides file by copying a template, replaces placeholders with recipient data, then exports the presentation to a PDF file.

**Nodes involved:**
- Copy Slides Template
- Replace Template Placeholders
- Export to PDF

#### Node: Copy Slides Template
- **Type / role:** HTTP Request (Google Drive API `files.copy` to duplicate the Slides template).
- **Configuration choices (interpreted):**
  - **POST** to: `https://www.googleapis.com/drive/v3/files/YOUR_TEMPLATE_SLIDES_ID/copy`
  - Sends JSON body: `{ "name": "Certificate_" + CertificateID }`
  - Uses **predefined credential type**: `googleDriveOAuth2Api`
- **Key expressions/variables:**
  - Body name: `Certificate_{{ $json['CertificateID'] }}`
    - Note: here `$json` is the current item from SplitInBatches (the recipient row).
- **Inputs/Outputs:**
  - Input ← Process One at a Time
  - Output → Replace Template Placeholders
  - Output JSON typically includes the copied file’s `id` used later.
- **Failure modes / edge cases:**
  - 403/401 if Drive OAuth scopes/permissions insufficient.
  - 404 if `YOUR_TEMPLATE_SLIDES_ID` is wrong or not accessible.
  - If CertificateID is missing/empty, file name becomes `Certificate_`.
- **Version notes:** typeVersion **4.2** (HTTP Request node version).

#### Node: Replace Template Placeholders
- **Type / role:** HTTP Request (Google Slides API `presentations.batchUpdate`).
- **Configuration choices (interpreted):**
  - **POST** to: `https://slides.googleapis.com/v1/presentations/{{ copiedFileId }}:batchUpdate`
  - Replaces placeholders in the Slides copy:
    - `{{name}}` → recipient `Name`
    - `{{date}}` → recipient `Date`
    - `{{certificateid}}` → recipient `CertificateID`
  - Uses **predefined credential type**: `googleSlidesOAuth2Api`
- **Key expressions/variables:**
  - Presentation ID: `{{ $json.id }}` (from Copy Slides Template output)
  - Replacement text uses explicit node references:
    - `{{ $('Process One at a Time').item.json.Name }}`
    - `{{ $('Process One at a Time').item.json.Date }}`
    - `{{ $('Process One at a Time').item.json['CertificateID'] }}`
- **Inputs/Outputs:**
  - Input ← Copy Slides Template
  - Output → Export to PDF
- **Failure modes / edge cases:**
  - If placeholders in the template do not match exactly (`{{name}}`, etc.), replacements won’t occur (resulting PDFs keep placeholders).
  - If the Slides API credential lacks proper scopes, returns 403.
  - Text replacement is case-insensitive (`matchCase: false`), which helps minor casing differences but not different brace formats.
- **Version notes:** typeVersion **4.2**.

#### Node: Export to PDF
- **Type / role:** HTTP Request (Drive API export endpoint to get PDF binary).
- **Configuration choices (interpreted):**
  - GET to: `https://www.googleapis.com/drive/v3/files/{{ presentationId }}/export?mimeType=application/pdf`
  - Response format set to **file** (binary).
  - Auth uses predefined credentials (configured as `googleSlidesOAuth2Api` in this workflow).
- **Key expressions/variables:**
  - Uses `$('Copy Slides Template').item.json.id` to export the copied presentation.
- **Inputs/Outputs:**
  - Input ← Replace Template Placeholders
  - Output → Save PDF to Drive (binary PDF attached)
- **Failure modes / edge cases:**
  - Export may fail if file not fully updated/available yet (rare, but possible); rate limiting can also occur.
  - Incorrect credential type/scopes can break export.
- **Version notes:** typeVersion **4.2**.

---

### 2.4 Save & Spreadsheet Update

**Overview:** Uploads the generated PDF to a Drive folder, then updates the corresponding row in Google Sheets with the Drive link and sets Processed to TRUE.

**Nodes involved:**
- Save PDF to Drive
- Mark as Processed

#### Node: Save PDF to Drive
- **Type / role:** Google Drive node (upload file).
- **Configuration choices (interpreted):**
  - Uploads the binary PDF produced by “Export to PDF”.
  - File name: `CertificateID.pdf`
  - Drive: “My Drive”
  - Target folder: `YOUR_OUTPUT_FOLDER_ID`
  - `alwaysOutputData: true` ensures the node still outputs metadata even if some operations might not output by default.
- **Key expressions/variables:**
  - Name: `={{ $('Process One at a Time').item.json['CertificateID']}}.pdf`
- **Inputs/Outputs:**
  - Input ← Export to PDF (binary)
  - Output → Mark as Processed
  - Output includes `webViewLink` used in the sheet update.
- **Failure modes / edge cases:**
  - Missing binary data (if export failed) → upload fails.
  - Folder ID invalid or not shared with the authenticated user.
  - CertificateID missing could create `.pdf` filename collisions.
- **Version notes:** typeVersion **3**.

#### Node: Mark as Processed
- **Type / role:** Google Sheets node (update row by matching key).
- **Configuration choices (interpreted):**
  - Operation: **update**
  - Matching columns: `CertificateID`
  - Updates these fields:
    - `DriveLink` = Drive file `webViewLink`
    - `Processed` = `"TRUE"`
    - `CertificateID` is also set from the read item (ensures key is present)
  - Uses spreadsheet ID `YOUR_GOOGLE_SHEET_ID` and sheet tab `gid=0`.
- **Key expressions/variables:**
  - `DriveLink`: `{{ $('Save PDF to Drive').item.json.webViewLink }}`
  - `CertificateID`: `{{ $('Read Unprocessed Recipients').item.json['CertificateID'] }}`
- **Inputs/Outputs:**
  - Input ← Save PDF to Drive
  - Output → Delete Temporary Slide
- **Failure modes / edge cases:**
  - If `CertificateID` is not unique in the sheet, multiple rows could be updated unexpectedly (depends on Sheets node behavior).
  - If the row was edited concurrently (e.g., Processed changed), update may target an unexpected row if CertificateID duplicates exist.
  - If the sheet stores booleans, writing `"TRUE"` as string may be inconsistent; ensure consistent type usage.
- **Version notes:** typeVersion **4.5**.

---

### 2.5 Cleanup & Rate-Limit Control (delete temp → wait → loop)

**Overview:** Deletes the temporary copied Slides file and waits 2 seconds before the loop continues with the next recipient.

**Nodes involved:**
- Delete Temporary Slide
- Wait 2s (Rate Limit)

#### Node: Delete Temporary Slide
- **Type / role:** HTTP Request (Drive API delete file).
- **Configuration choices (interpreted):**
  - **DELETE** to: `https://www.googleapis.com/drive/v3/files/{{ copiedFileId }}`
  - Uses predefined credentials (`googleSlidesOAuth2Api` as configured).
- **Key expressions/variables:**
  - File ID: `{{ $('Copy Slides Template').item.json.id }}`
- **Inputs/Outputs:**
  - Input ← Mark as Processed
  - Output → Wait 2s (Rate Limit)
- **Failure modes / edge cases:**
  - If the copy ID is missing (upstream failure), deletion request fails.
  - Permissions issues: user must have rights to delete the copied file.
- **Version notes:** typeVersion **4.2**.

#### Node: Wait 2s (Rate Limit)
- **Type / role:** Wait (delay execution).
- **Configuration choices (interpreted):**
  - Wait amount: **2 seconds**
  - Used to reduce likelihood of hitting Google API rate limits.
- **Inputs/Outputs:**
  - Input ← Delete Temporary Slide
  - Output → Process One at a Time (loop continues)
- **Failure modes / edge cases:**
  - Adds latency; large batches will take longer.
  - If rate limits are still hit, you may need longer waits or exponential backoff logic.
- **Version notes:** typeVersion **1.1**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Test workflow' | Manual Trigger | Manual entry point | — | Read Unprocessed Recipients |  |
| Read Unprocessed Recipients | Google Sheets | Read unprocessed rows (Processed=FALSE) | When clicking 'Test workflow' | Process One at a Time | ### 📥 Data Input<br>Reads recipients from Google Sheets and processes them one at a time to avoid API rate limits. |
| Process One at a Time | Split In Batches | Sequential loop controller | Read Unprocessed Recipients; Wait 2s (Rate Limit) | Copy Slides Template | ### 📥 Data Input<br>Reads recipients from Google Sheets and processes them one at a time to avoid API rate limits. |
| Copy Slides Template | HTTP Request | Copy Slides template via Drive API | Process One at a Time | Replace Template Placeholders | ### 🎨 Certificate Generation<br>Copies the Slides template, replaces placeholders with recipient data, and exports as PDF. |
| Replace Template Placeholders | HTTP Request | Replace placeholders via Slides batchUpdate | Copy Slides Template | Export to PDF | ### 🎨 Certificate Generation<br>Copies the Slides template, replaces placeholders with recipient data, and exports as PDF. |
| Export to PDF | HTTP Request | Export Slides to PDF (binary) | Replace Template Placeholders | Save PDF to Drive | ### 🎨 Certificate Generation<br>Copies the Slides template, replaces placeholders with recipient data, and exports as PDF. |
| Save PDF to Drive | Google Drive | Upload generated PDF to Drive folder | Export to PDF | Mark as Processed | ### 💾 Save & Update<br>Saves the PDF to Drive and updates the spreadsheet with the file link. |
| Mark as Processed | Google Sheets | Update row: DriveLink + Processed=TRUE | Save PDF to Drive | Delete Temporary Slide | ### 💾 Save & Update<br>Saves the PDF to Drive and updates the spreadsheet with the file link. |
| Delete Temporary Slide | HTTP Request | Delete temp Slides copy from Drive | Mark as Processed | Wait 2s (Rate Limit) | ### 🧹 Cleanup & Loop<br>Deletes temporary files and waits 2 seconds before processing the next recipient. |
| Wait 2s (Rate Limit) | Wait | Delay to reduce rate-limit risk | Delete Temporary Slide | Process One at a Time | ### 🧹 Cleanup & Loop<br>Deletes temporary files and waits 2 seconds before processing the next recipient. |
| Sticky Note | Sticky Note | Documentation (how it works + setup steps) | — | — | ## How it works<br>This workflow generates personalized PDF certificates in bulk using Google Sheets as the data source and Google Slides as the template engine.<br><br>1. Reads recipient data from a Google Sheet (only rows where Processed = FALSE)<br>2. For each recipient, copies your Slides template and replaces placeholders with actual values<br>3. Exports the personalized slide as a PDF and saves it to Google Drive<br>4. Updates the spreadsheet with the Drive link and marks the row as processed<br>5. Cleans up by deleting the temporary slide copy<br><br>## Setup steps<br>1. **Google Sheet**: Create a sheet with columns: Name, Date, CertificateID, Processed, DriveLink. Set Processed to FALSE for new entries.<br>2. **Slides Template**: Create a template with placeholders: {{name}}, {{date}}, {{certificateid}}<br>3. **Credentials**: Connect Google Sheets, Drive, and Slides OAuth2 credentials<br>4. **Configure IDs**: Replace YOUR_GOOGLE_SHEET_ID, YOUR_TEMPLATE_SLIDES_ID, and YOUR_OUTPUT_FOLDER_ID in the nodes<br>5. **Test**: Run with one row first to verify everything works correctly |
| Sticky Note1 | Sticky Note | Documentation label (Data Input) | — | — | ### 📥 Data Input<br>Reads recipients from Google Sheets and processes them one at a time to avoid API rate limits. |
| Sticky Note2 | Sticky Note | Documentation label (Certificate Generation) | — | — | ### 🎨 Certificate Generation<br>Copies the Slides template, replaces placeholders with recipient data, and exports as PDF. |
| Sticky Note3 | Sticky Note | Documentation label (Save & Update) | — | — | ### 💾 Save & Update<br>Saves the PDF to Drive and updates the spreadsheet with the file link. |
| Sticky Note4 | Sticky Note | Documentation label (Cleanup & Loop) | — | — | ### 🧹 Cleanup & Loop<br>Deletes temporary files and waits 2 seconds before processing the next recipient. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Bulk certificate generator from Google Sheets and Slides*.

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: **When clicking 'Test workflow'**

3. **Add node: Google Sheets (read)**
   - Node type: **Google Sheets**
   - Name: **Read Unprocessed Recipients**
   - Credentials: Google Sheets OAuth2 (account with access to the sheet)
   - Document ID: set to your spreadsheet ID (replace `YOUR_GOOGLE_SHEET_ID`)
   - Sheet: select the tab (in JSON it is `gid=0`)
   - Add a filter:
     - Column: **Processed**
     - Value: **FALSE** (ensure it matches your sheet data type)
   - Connect: **Manual Trigger → Read Unprocessed Recipients**

4. **Add node: Split In Batches**
   - Node type: **Split In Batches**
   - Name: **Process One at a Time**
   - Options:
     - Reset: **false**
     - Batch size: **1** (set explicitly in UI to ensure one-at-a-time behavior)
   - Connect: **Read Unprocessed Recipients → Process One at a Time**

5. **Add node: HTTP Request (copy template)**
   - Node type: **HTTP Request**
   - Name: **Copy Slides Template**
   - Authentication: **Predefined Credential Type**
   - Credential type: **Google Drive OAuth2 API**
   - Method: **POST**
   - URL: `https://www.googleapis.com/drive/v3/files/YOUR_TEMPLATE_SLIDES_ID/copy`  
     Replace `YOUR_TEMPLATE_SLIDES_ID` with the Slides template file ID.
   - Body content type: **JSON**
   - Body (JSON) expression equivalent:
     - Set `name` to: `Certificate_{{$json["CertificateID"]}}`
   - Connect: **Process One at a Time (main output) → Copy Slides Template**

6. **Add node: HTTP Request (replace placeholders)**
   - Node type: **HTTP Request**
   - Name: **Replace Template Placeholders**
   - Authentication: **Predefined Credential Type**
   - Credential type: **Google Slides OAuth2 API**
   - Method: **POST**
   - URL: `https://slides.googleapis.com/v1/presentations/{{$json.id}}:batchUpdate`
     - Here `$json.id` is the copied Slides file id from the previous node.
   - Body: JSON with `requests[]` using `replaceAllText` for:
     - `{{name}}` → `{{$('Process One at a Time').item.json.Name}}`
     - `{{date}}` → `{{$('Process One at a Time').item.json.Date}}`
     - `{{certificateid}}` → `{{$('Process One at a Time').item.json.CertificateID}}`
   - Connect: **Copy Slides Template → Replace Template Placeholders**

7. **Add node: HTTP Request (export PDF)**
   - Node type: **HTTP Request**
   - Name: **Export to PDF**
   - Authentication: **Predefined Credential Type**
   - Credential type: (use one with Drive export access; workflow uses Google Slides OAuth2 API)
   - Method: **GET**
   - URL: `https://www.googleapis.com/drive/v3/files/{{$('Copy Slides Template').item.json.id}}/export?mimeType=application/pdf`
   - Response: set **Response Format = File** (binary)
   - Connect: **Replace Template Placeholders → Export to PDF**

8. **Add node: Google Drive (upload)**
   - Node type: **Google Drive**
   - Name: **Save PDF to Drive**
   - Credentials: Google Drive OAuth2 (same account)
   - Operation: upload (default “Upload” behavior)
   - File name: `{{$('Process One at a Time').item.json.CertificateID}}.pdf`
   - Drive: **My Drive**
   - Folder ID: replace `YOUR_OUTPUT_FOLDER_ID`
   - Ensure it uploads the **binary** from the previous HTTP Request node (Export to PDF).
   - Connect: **Export to PDF → Save PDF to Drive**

9. **Add node: Google Sheets (update)**
   - Node type: **Google Sheets**
   - Name: **Mark as Processed**
   - Credentials: Google Sheets OAuth2
   - Document ID: your spreadsheet ID
   - Sheet: `gid=0` (or your chosen tab)
   - Operation: **Update**
   - Matching column: **CertificateID**
   - Columns to update:
     - **DriveLink** = `{{$('Save PDF to Drive').item.json.webViewLink}}`
     - **Processed** = `TRUE` (use consistent type with your sheet)
   - Connect: **Save PDF to Drive → Mark as Processed**

10. **Add node: HTTP Request (delete temp slide)**
    - Node type: **HTTP Request**
    - Name: **Delete Temporary Slide**
    - Authentication: predefined (Drive-capable credential)
    - Method: **DELETE**
    - URL: `https://www.googleapis.com/drive/v3/files/{{$('Copy Slides Template').item.json.id}}`
    - Connect: **Mark as Processed → Delete Temporary Slide**

11. **Add node: Wait**
    - Node type: **Wait**
    - Name: **Wait 2s (Rate Limit)**
    - Amount: **2 seconds**
    - Connect: **Delete Temporary Slide → Wait 2s (Rate Limit)**

12. **Close the loop**
    - Connect: **Wait 2s (Rate Limit) → Process One at a Time**
    - Ensure you connect to the node input so it continues with the next batch item.

13. **Prepare external assets**
    - **Google Sheet columns:** `Name`, `Date`, `CertificateID`, `Processed`, `DriveLink`
    - Set `Processed` = `FALSE` for new rows.
    - **Slides template placeholders:** `{{name}}`, `{{date}}`, `{{certificateid}}` placed in text boxes.

14. **Credentials / scopes checklist**
    - Google Sheets OAuth2: read + update spreadsheet.
    - Google Drive OAuth2: copy file, export file, upload file, delete file.
    - Google Slides OAuth2: batchUpdate presentation text.
    - Ensure the authenticated account has access to:
      - the sheet
      - the Slides template
      - the output Drive folder

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow generates personalized PDF certificates in bulk using Google Sheets as the data source and Google Slides as the template engine. | Sticky Note (“How it works”) |
| Setup steps: create Sheet columns (Name, Date, CertificateID, Processed, DriveLink); create Slides placeholders ({{name}}, {{date}}, {{certificateid}}); connect OAuth2; replace IDs; test with one row first. | Sticky Note (“Setup steps”) |