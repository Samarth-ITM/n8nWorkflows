Generate client contracts automatically using Google Docs, Drive, and GoHighLevel

https://n8nworkflows.xyz/workflows/generate-client-contracts-automatically-using-google-docs--drive--and-gohighlevel-14364


# Generate client contracts automatically using Google Docs, Drive, and GoHighLevel

## 1. Workflow Overview

This workflow automatically generates a client contract from submitted data, using a Google Docs template as the source document, Google Drive for file lifecycle management, and GoHighLevel for final PDF upload.

Primary use case:
- A form, CRM, or external system sends a POST request containing contract details.
- The workflow creates a copy of a master contract template.
- It replaces placeholder/example text in the copied Google Doc with actual client-specific values.
- It exports the finished document as a PDF.
- It uploads the PDF to GoHighLevel.
- It deletes the temporary Google Docs copy from Drive.

### 1.1 Input Reception
The workflow starts from a webhook that receives the full contract payload as an HTTP POST request.

### 1.2 Template Copy Creation
A master Google Docs contract template is duplicated in Google Drive and renamed dynamically using the client name and date.

### 1.3 Contract Population
The copied Google Doc is updated through a large set of text replacement actions. These replacements map incoming webhook fields to contract placeholders/example text embedded in the template.

### 1.4 PDF Export and CRM Upload
The populated contract is downloaded as a PDF from Google Drive and then uploaded to GoHighLevel using its media upload API.

### 1.5 Cleanup
After a successful upload, the temporary Google Drive copy is deleted to avoid leaving draft files behind.

### 1.6 Documentation and Security Notes
Several sticky notes describe the workflow purpose, setup steps, security expectations, and functional areas.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block is the entry point of the workflow. It receives the contract data payload via HTTP POST and exposes the request body to downstream nodes through expressions.

**Nodes Involved:**  
- Receive Contract Request

### Node Details

#### Receive Contract Request
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Trigger node that starts the workflow when an HTTP POST request hits the configured webhook path.
- **Configuration choices:**
  - HTTP method: `POST`
  - Webhook path: `dc09097f-d5ac-4352-b249-7f03e71876d6`
  - No special options configured
- **Key expressions or variables used:**
  - Downstream nodes access payload values via:
    - `$('Receive Contract Request').item.json.body.full_name`
    - `$('Receive Contract Request').item.json.body.date`
    - `$('Receive Contract Request').item.json.body.scope_of_works[...]`
    - `$('Receive Contract Request').item.json.body.pc_items[...]`
    - `$('Receive Contract Request').item.json.body.subtotal`
    - `$('Receive Contract Request').item.json.body.gst`
    - `$('Receive Contract Request').item.json.body.total`
    - `$('Receive Contract Request').item.json.body.fee_full_name`
    - `$('Receive Contract Request').item.json.body.fee_date`
- **Input and output connections:**
  - Input: none
  - Output: `Copy Master Template`
- **Version-specific requirements:**
  - Type version `2.1`
  - Standard n8n webhook behavior applies; production URL differs from test URL.
- **Edge cases or potential failure types:**
  - Missing or malformed JSON body
  - Wrong HTTP method
  - Downstream expression failures if required keys are absent
  - Array index errors if `scope_of_works` or `pc_items` are shorter than expected
- **Sub-workflow reference:** none

---

## 2.2 Template Copy Creation

**Overview:**  
This block duplicates the master Google Docs template in Drive. It preserves the original template and creates a working document named from incoming client data.

**Nodes Involved:**  
- Copy Master Template

### Node Details

#### Copy Master Template
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Copies an existing file in Google Drive.
- **Configuration choices:**
  - Operation: `copy`
  - Source file ID: manually configured as `YOUR_MASTER_TEMPLATE_FILE_ID`
  - New file name built dynamically:
    - `Contract_<full_name>_<date>`
- **Key expressions or variables used:**
  - File name:
    - `={{ 'Contract_' + $('Receive Contract Request').item.json.body.full_name + '_' + $('Receive Contract Request').item.json.body.date }}`
- **Input and output connections:**
  - Input: `Receive Contract Request`
  - Output: `Populate Contract with Client Data`
- **Version-specific requirements:**
  - Type version `3`
  - Requires Google Drive OAuth2 credentials with permission to read/copy the source file
- **Edge cases or potential failure types:**
  - Invalid or missing Google Drive credentials
  - `YOUR_MASTER_TEMPLATE_FILE_ID` not replaced
  - File not found or insufficient permissions
  - Invalid file naming characters from payload data
  - If `full_name` or `date` is missing, the created file name may be blank or malformed
- **Sub-workflow reference:** none

---

## 2.3 Contract Population

**Overview:**  
This block performs all client-specific content injection. It opens the copied Google Doc and executes a series of replace-all actions to swap known template text with incoming payload values.

**Nodes Involved:**  
- Populate Contract with Client Data

### Node Details

#### Populate Contract with Client Data
- **Type and technical role:** `n8n-nodes-base.googleDocs`  
  Updates a Google Doc by performing bulk text replacements.
- **Configuration choices:**
  - Operation: `update`
  - Target document URL/value:
    - `={{ $json.id }}`
    - This refers to the file created by `Copy Master Template`
  - The node defines a long list of `replaceAll` actions
  - The workflow uses literal example text already present in the template as replacement anchors
- **Key expressions or variables used:**
  - Document reference:
    - `={{ $json.id }}`
  - Replacements from webhook body:
    - `full_name`
    - `date`
    - `scope_of_works[0..8].heading`
    - `scope_of_works[0..8].sub_heading`
    - `pc_items[0..14].item`
    - `pc_items[0..14].value`
    - `fee_full_name`
    - `fee_date`
    - `subtotal`
    - `gst`
    - `total`
- **Input and output connections:**
  - Input: `Copy Master Template`
  - Output: `Download Contract as PDF`
- **Version-specific requirements:**
  - Type version `2`
  - Requires Google Docs OAuth2 credentials with permission to edit the copied document
- **Edge cases or potential failure types:**
  - If the document ID is invalid or inaccessible, update fails
  - If any incoming arrays are too short, expressions like `scope_of_works[8]` or `pc_items[14]` can fail
  - If the source template text no longer exactly matches the configured find-text strings, replacement will silently miss content
  - Duplicate find-text values may cause unintended multiple replacements
  - Inconsistent date formatting between template text and incoming payload may cause mismatched expectations
  - Null values may replace content with empty strings
- **Sub-workflow reference:** none

**Important structural observation:**  
This workflow is tightly coupled to a specific template shape:
- exactly 9 scope-of-work entries are expected (`0` through `8`)
- exactly 15 PC items are expected (`0` through `14`)

If the payload structure changes, this node must be updated manually.

---

## 2.4 PDF Export and CRM Upload

**Overview:**  
This block turns the completed contract document into a PDF binary file and sends it to GoHighLevel through an authenticated multipart upload request.

**Nodes Involved:**  
- Download Contract as PDF
- Upload PDF to GoHighLevel

### Node Details

#### Download Contract as PDF
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads the copied Google Docs file, typically exporting it in binary form suitable for later upload.
- **Configuration choices:**
  - Operation: `download`
  - File ID taken from the copied template result:
    - `={{ $('Copy Master Template').item.json.id }}`
- **Key expressions or variables used:**
  - File ID:
    - `={{ $('Copy Master Template').item.json.id }}`
- **Input and output connections:**
  - Input: `Populate Contract with Client Data`
  - Output: `Upload PDF to GoHighLevel`
- **Version-specific requirements:**
  - Type version `3`
  - Requires Drive access to the copied file
- **Edge cases or potential failure types:**
  - Download/export may fail if the copied file is missing
  - Permission errors
  - If the Google Drive node does not return binary under the expected field name, the HTTP upload will fail
  - Large files may increase execution time
- **Sub-workflow reference:** none

#### Upload PDF to GoHighLevel
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the generated PDF to the GoHighLevel media upload API as multipart form-data.
- **Configuration choices:**
  - URL: `https://services.leadconnectorhq.com/medias/upload-file`
  - Method: `POST`
  - Content type: `multipart-form-data`
  - Sends headers explicitly
  - Request body includes one multipart field:
    - `file` mapped from binary property `data`
  - Custom headers:
    - `Accept: application/json`
    - `Version: 2021-07-28`
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Channel: API`
    - `source: WEB_USER`
    - `origin: https://app.gohighlevel.com`
    - `referer: https://app.gohighlevel.com/`
- **Key expressions or variables used:**
  - No dynamic expression in headers/body except binary attachment reference
  - Binary input field expected: `data`
- **Input and output connections:**
  - Input: `Download Contract as PDF`
  - Output: `Delete Temp Copy from Drive`
- **Version-specific requirements:**
  - Type version `4.3`
  - GoHighLevel API behavior may depend on account permissions and API version
- **Edge cases or potential failure types:**
  - Placeholder token not replaced
  - Expired or invalid token
  - Binary property mismatch if file is not in `data`
  - API schema changes or required headers changing over time
  - HTTP 401/403 authentication failures
  - HTTP 400 if multipart request is malformed
  - Upload succeeds but returned media ID is not captured for further association to a CRM object
- **Sub-workflow reference:** none

**Important integration note:**  
The workflow uploads the file to GoHighLevel media storage, but it does not attach the uploaded media to a specific contact, opportunity, or conversation in the current version.

---

## 2.5 Cleanup

**Overview:**  
This block removes the temporary Google Drive copy once the upload has completed. It ensures the process is ephemeral and does not accumulate generated document drafts.

**Nodes Involved:**  
- Delete Temp Copy from Drive

### Node Details

#### Delete Temp Copy from Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Deletes the copied contract file from Google Drive.
- **Configuration choices:**
  - Operation: `deleteFile`
  - File ID:
    - `={{ $('Copy Master Template').item.json.id }}`
- **Key expressions or variables used:**
  - File ID:
    - `={{ $('Copy Master Template').item.json.id }}`
- **Input and output connections:**
  - Input: `Upload PDF to GoHighLevel`
  - Output: none
- **Version-specific requirements:**
  - Type version `3`
  - Requires delete permission on the copied file
- **Edge cases or potential failure types:**
  - If upload fails, this node will not run
  - If the file was already deleted or moved, deletion fails
  - Permission issues may leave orphaned files
- **Sub-workflow reference:** none

---

## 2.6 Documentation and Security Notes

**Overview:**  
These nodes are non-executable annotations that describe the workflow purpose, setup requirements, execution flow, and credential handling practices.

**Nodes Involved:**  
- Sticky Note: Overview
- Sticky Note: Webhook
- Sticky Note: Template Population
- Sticky Note: Export & Upload
- Sticky Note: Cleanup
- Sticky Note: Security

### Node Details

#### Sticky Note: Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for overall workflow explanation and setup.
- **Configuration choices:**
  - Large note containing process summary and setup checklist
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none; documentation only
- **Sub-workflow reference:** none

#### Sticky Note: Webhook
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** documents the webhook trigger purpose
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note: Template Population
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains copy and replace behavior
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note: Export & Upload
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains PDF export and GHL upload behavior
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note: Cleanup
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** explains deletion of temp file
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note: Security
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** credential safety reminder
- **Input and output connections:** none
- **Version-specific requirements:** type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note: Overview | stickyNote | Visual documentation for workflow purpose and setup |  |  | ## 📄 Automated Contract Generator<br>### How it works<br>When a webhook receives client data (name, date, scope of works, PC items, and financials), the workflow copies a master Google Docs contract template, populates it with the submitted details using find-and-replace, exports it as a PDF, uploads it to GoHighLevel, and then deletes the temporary copy from Drive — leaving no clutter behind.<br>This removes the manual work of copying, editing, and emailing contracts one by one. Every contract is generated in seconds, consistently formatted, and automatically delivered to your CRM.<br>### Setup steps<br>1. **Google Drive** — Connect your Google Drive OAuth2 credential. Update the `Copy Master Template` node with your own master template file ID.<br>2. **Google Docs** — Connect your Google Docs OAuth2 credential to the `Populate Contract with Client Data` node.<br>3. **GoHighLevel** — Replace the placeholder Bearer token in `Upload PDF to GoHighLevel` with your own GHL API key.<br>4. **Webhook** — Copy the webhook URL and add it to whatever form or CRM triggers contract generation.<br>5. **Test** — Submit a test payload matching the expected body structure (full_name, date, scope_of_works[], pc_items[], subtotal, gst, total) and verify the PDF arrives in GHL. |
| Sticky Note: Webhook | stickyNote | Visual note for entry point |  |  | ## 🔗 Webhook Trigger<br>Receives a POST request with all client contract details — name, date, scope of works, PC items, and pricing. This is the entry point; nothing runs until this fires. |
| Sticky Note: Template Population | stickyNote | Visual note for copy and replace block |  |  | ## 📋 Template Copy & Population<br>Clones the master Google Docs contract template and replaces all placeholder text with the client's actual data — name, date, scope of works headings and descriptions, PC item names and values, and totals. The original template is never modified. |
| Sticky Note: Export & Upload | stickyNote | Visual note for PDF export and CRM upload block |  |  | ## 📤 Export & Upload to CRM<br>Downloads the completed contract as a PDF from Google Drive, then uploads it directly to GoHighLevel via the media API. The PDF is ready to attach to a contact record or send to the client. |
| Sticky Note: Cleanup | stickyNote | Visual note for cleanup block |  |  | ## 🗑️ Cleanup<br>Deletes the temporary Google Drive copy after the PDF has been successfully uploaded. Keeps your Drive tidy with no leftover contract drafts. |
| Sticky Note: Security | stickyNote | Visual security guidance |  |  | ## 🔐 Credentials & Security<br>Use OAuth2 for Google Drive and Google Docs. Replace the GoHighLevel Bearer token with a credential stored in n8n — never paste live API keys into node fields for shared templates. |
| Receive Contract Request | webhook | Entry point receiving POST payload |  | Copy Master Template | ## 🔗 Webhook Trigger<br>Receives a POST request with all client contract details — name, date, scope of works, PC items, and pricing. This is the entry point; nothing runs until this fires. |
| Copy Master Template | googleDrive | Duplicate the master Google Docs template | Receive Contract Request | Populate Contract with Client Data | ## 📋 Template Copy & Population<br>Clones the master Google Docs contract template and replaces all placeholder text with the client's actual data — name, date, scope of works headings and descriptions, PC item names and values, and totals. The original template is never modified. |
| Populate Contract with Client Data | googleDocs | Replace template text with client payload data | Copy Master Template | Download Contract as PDF | ## 📋 Template Copy & Population<br>Clones the master Google Docs contract template and replaces all placeholder text with the client's actual data — name, date, scope of works headings and descriptions, PC item names and values, and totals. The original template is never modified. |
| Download Contract as PDF | googleDrive | Export/download the generated document as binary PDF | Populate Contract with Client Data | Upload PDF to GoHighLevel | ## 📤 Export & Upload to CRM<br>Downloads the completed contract as a PDF from Google Drive, then uploads it directly to GoHighLevel via the media API. The PDF is ready to attach to a contact record or send to the client. |
| Upload PDF to GoHighLevel | httpRequest | Upload PDF to GoHighLevel media API | Download Contract as PDF | Delete Temp Copy from Drive | ## 📤 Export & Upload to CRM<br>Downloads the completed contract as a PDF from Google Drive, then uploads it directly to GoHighLevel via the media API. The PDF is ready to attach to a contact record or send to the client.<br>## 🔐 Credentials & Security<br>Use OAuth2 for Google Drive and Google Docs. Replace the GoHighLevel Bearer token with a credential stored in n8n — never paste live API keys into node fields for shared templates. |
| Delete Temp Copy from Drive | googleDrive | Delete temporary contract copy after upload | Upload PDF to GoHighLevel |  | ## 🗑️ Cleanup<br>Deletes the temporary Google Drive copy after the PDF has been successfully uploaded. Keeps your Drive tidy with no leftover contract drafts. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Generate Client Contracts Automatically using Google Docs, Drive, and GoHighLevel`.

2. **Add the webhook trigger**
   - Create a **Webhook** node.
   - Rename it to `Receive Contract Request`.
   - Set:
     - HTTP Method: `POST`
     - Path: `dc09097f-d5ac-4352-b249-7f03e71876d6` or your own path
   - Keep default response behavior unless your integration requires a custom response.

3. **Prepare the expected incoming payload**
   - Ensure the sender posts a JSON body containing at least:
     - `full_name`
     - `date`
     - `fee_full_name`
     - `fee_date`
     - `subtotal`
     - `gst`
     - `total`
     - `scope_of_works` as an array with at least 9 entries
     - `pc_items` as an array with at least 15 entries
   - Each `scope_of_works` item should contain:
     - `heading`
     - `sub_heading`
   - Each `pc_items` item should contain:
     - `item`
     - `value`

4. **Add the Google Drive copy node**
   - Create a **Google Drive** node.
   - Rename it to `Copy Master Template`.
   - Connect `Receive Contract Request` → `Copy Master Template`.
   - Set operation to **Copy**.
   - In the source file field, provide the Google Drive file ID of your master Google Docs template.
   - Set the destination name expression to:
     - `{{ 'Contract_' + $('Receive Contract Request').item.json.body.full_name + '_' + $('Receive Contract Request').item.json.body.date }}`
   - Attach a **Google Drive OAuth2** credential with access to the template file.

5. **Create or prepare the master Google Docs template**
   - Build a Google Doc containing the exact placeholder/example text you want to replace.
   - In this workflow version, replacements rely on exact literal text matches, not generic tokens like `{{name}}`.
   - That means the example values in your template must exactly match the `text` entries configured in the Google Docs update node.
   - Recommended improvement: use unique placeholders such as `{{FULL_NAME}}`, `{{SCOPE_1_HEADING}}`, etc., if rebuilding for maintainability.

6. **Add the Google Docs update node**
   - Create a **Google Docs** node.
   - Rename it to `Populate Contract with Client Data`.
   - Connect `Copy Master Template` → `Populate Contract with Client Data`.
   - Set operation to **Update**.
   - Set the document reference field to:
     - `{{ $json.id }}`
   - Add a series of **Replace All** actions.

7. **Configure name/date replacements**
   - Add replace actions mapping template text to webhook values:
     - Replace `Michelle Humphries` with `{{ $('Receive Contract Request').item.json.body.full_name }}`
     - Replace `22ⁿᵈ of December 2025` with `{{ $('Receive Contract Request').item.json.body.date }}`
     - Replace `Michelle Humphries` with `{{ $('Receive Contract Request').item.json.body.fee_full_name }}`
     - Replace `22nd of December 2025` with `{{ $('Receive Contract Request').item.json.body.fee_date }}`
   - Note: using the same original text multiple times can cause ambiguity if both instances are identical in the document.

8. **Configure scope of works replacements**
   - Add 18 replace actions:
     - 9 for `scope_of_works[n].heading`
     - 9 for `scope_of_works[n].sub_heading`
   - For each action:
     - `text` = exact example text currently present in the Google Docs template
     - `replaceText` = matching expression, for example:
       - `{{ $('Receive Contract Request').item.json.body.scope_of_works[0].heading }}`
       - `{{ $('Receive Contract Request').item.json.body.scope_of_works[0].sub_heading }}`
   - Repeat through index `8`.

9. **Configure PC item replacements**
   - Add 30 replace actions:
     - 15 for `pc_items[n].item`
     - 15 for `pc_items[n].value`
   - Example mappings:
     - item: `{{ $('Receive Contract Request').item.json.body.pc_items[0].item }}`
     - value: `{{ $('Receive Contract Request').item.json.body.pc_items[0].value }}`
   - Repeat through index `14`.

10. **Configure financial total replacements**
    - Add replace actions for:
      - `58,808.61` → `{{ $('Receive Contract Request').item.json.body.subtotal }}`
      - `5,880.86` → `{{ $('Receive Contract Request').item.json.body.gst }}`
      - `64,689.47` → `{{ $('Receive Contract Request').item.json.body.total }}`

11. **Attach Google Docs credentials**
    - Connect a **Google Docs OAuth2** credential to the node.
    - Ensure the authorized account can edit the copied document.

12. **Add the PDF download/export node**
    - Create another **Google Drive** node.
    - Rename it to `Download Contract as PDF`.
    - Connect `Populate Contract with Client Data` → `Download Contract as PDF`.
    - Set operation to **Download**.
    - Set File ID to:
      - `{{ $('Copy Master Template').item.json.id }}`
    - Use the same Google Drive OAuth2 credential or another one with access to the copied file.

13. **Verify binary output**
    - Confirm the Google Drive download node outputs the file as binary.
    - The next node expects the binary property to be named `data`.
    - If your n8n version or node configuration uses a different binary property name, adapt the HTTP Request node accordingly.

14. **Add the GoHighLevel upload node**
    - Create an **HTTP Request** node.
    - Rename it to `Upload PDF to GoHighLevel`.
    - Connect `Download Contract as PDF` → `Upload PDF to GoHighLevel`.
    - Set:
      - Method: `POST`
      - URL: `https://services.leadconnectorhq.com/medias/upload-file`
      - Content Type: `Multipart Form-Data`
      - Send Body: enabled
      - Send Headers: enabled

15. **Configure the multipart body**
    - Add one body parameter:
      - Name: `file`
      - Parameter type: `Form Binary Data`
      - Input data field name: `data`

16. **Configure GoHighLevel headers**
    - Add these headers:
      - `Accept` = `application/json`
      - `Version` = `2021-07-28`
      - `Authorization` = `Bearer YOUR_TOKEN_HERE`
      - `Channel` = `API`
      - `source` = `WEB_USER`
      - `origin` = `https://app.gohighlevel.com`
      - `referer` = `https://app.gohighlevel.com/`
   - Recommended improvement:
     - Store the token in n8n credentials, an environment variable, or a preconfigured auth header pattern instead of hardcoding it.

17. **Add the cleanup node**
    - Create a **Google Drive** node.
    - Rename it to `Delete Temp Copy from Drive`.
    - Connect `Upload PDF to GoHighLevel` → `Delete Temp Copy from Drive`.
    - Set operation to **Delete File**.
    - Set File ID to:
      - `{{ $('Copy Master Template').item.json.id }}`

18. **Add optional sticky notes**
    - Add sticky notes for:
      - workflow overview
      - webhook trigger
      - template copy and population
      - export/upload
      - cleanup
      - credential security

19. **Configure credentials**
    - **Google Drive OAuth2**
      - Must allow copying, downloading, and deleting files.
    - **Google Docs OAuth2**
      - Must allow editing the copied Google Doc.
    - **GoHighLevel**
      - Prefer secure token storage outside the node field.

20. **Test with a complete payload**
   - Use the webhook test URL and submit a JSON POST request.
   - Include all required arrays and indexes.
   - Confirm:
     - template copy is created
     - replacements occur
     - PDF is downloaded
     - upload to GoHighLevel succeeds
     - copied file is deleted

21. **Activate the workflow**
   - Switch from test URL usage to production webhook URL.
   - Update the calling system to use the production endpoint.

### Expected payload shape

A compatible payload should resemble this structure conceptually:

- `body.full_name`
- `body.date`
- `body.fee_full_name`
- `body.fee_date`
- `body.subtotal`
- `body.gst`
- `body.total`
- `body.scope_of_works[0..8].heading`
- `body.scope_of_works[0..8].sub_heading`
- `body.pc_items[0..14].item`
- `body.pc_items[0..14].value`

### Rebuild cautions
- The Google Docs replace mechanism is dependent on exact source text matches.
- The workflow assumes fixed-length arrays.
- No validation node exists before replacement.
- No explicit webhook response node is configured.
- No error branch exists for failed upload or failed cleanup.

### Suggested hardening if rebuilding
- Add a **Set** or **Code** node to validate required payload fields.
- Add an **IF** node to check array lengths.
- Replace literal template text with unique placeholders.
- Add **Error Trigger** or failure branches.
- Add a final webhook response confirming success/failure.
- Capture the GoHighLevel upload response and store the uploaded media reference if needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated Contract Generator overview: receives client data, copies a master Google Docs contract template, populates it, exports it as PDF, uploads it to GoHighLevel, and deletes the temporary Drive copy. | Workflow documentation note |
| Setup reminder: connect Google Drive OAuth2, Google Docs OAuth2, replace the GoHighLevel Bearer token, configure the webhook URL, and test with a matching payload. | Workflow documentation note |
| Security reminder: use OAuth2 for Google Drive and Google Docs; do not leave live GoHighLevel API keys directly in shared node fields. | Workflow documentation note |

### Additional implementation note
There are no sub-workflows in this workflow, and there is only one entry point: the `Receive Contract Request` webhook.