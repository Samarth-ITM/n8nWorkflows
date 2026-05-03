Archive HR and legal documents with UploadToURL, Google Drive and Airtable

https://n8nworkflows.xyz/workflows/archive-hr-and-legal-documents-with-uploadtourl--google-drive-and-airtable-13544


# Archive HR and legal documents with UploadToURL, Google Drive and Airtable

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Archive HR and legal documents with UploadToURL, Google Drive and Airtable

**Purpose:**  
This workflow ingests signed HR/legal documents (either as a remote URL or a binary upload), validates and normalizes employee metadata, prevents duplicates via Airtable, uploads the document to a stable CDN (UploadToURL) and to a structured Google Drive folder, updates/creates the employee record in Airtable, sends confirmation email(s), and returns a structured webhook response. It also provides dedicated conflict handling (409) and general error handling (400).

### 1.1 Input Reception & Validation
Receives POST requests and sanitizes/normalizes metadata, generates deterministic filenames, MIME types, and Drive folder paths.

### 1.2 Duplicate Prevention (Pre-Upload)
Checks Airtable first to avoid re-filing and wasted API/Drive operations; returns **409 Conflict** when a contract already exists.

### 1.3 Upload Chain (CDN + Google Drive)
Uploads the document to UploadToURL (remote or binary path), extracts the CDN URL, resolves/creates the correct Drive folder, and uploads to Drive.

### 1.4 Airtable Upsert (Update or Create)
Finds the employee record and updates it; if missing, creates it. Stores both Drive URL and CDN URL for redundancy.

### 1.5 Notification & Webhook Response
Sends confirmation via Gmail and returns a **201 Created** response containing an audit-friendly summary.

### 1.6 Error & Conflict Handling
Provides a specific duplicate response (409) and a generic error handler that can be connected to node error outputs, returning **400**.

---

## 2. Block-by-Block Analysis

### Block A — Input Reception & Validation

**Overview:**  
Accepts inbound contract filing requests via webhook, validates required fields, normalizes employee identifiers and names, derives file metadata (extension/MIME), and computes the Drive folder path and structured filename.

**Nodes involved:**  
- Webhook - Receive Contract  
- Validate & Enrich Payload  

#### Node: Webhook - Receive Contract
- **Type / role:** `Webhook` (n8n core) — entry point, receives external HTTP requests.
- **Key configuration:**
  - Method: `POST`
  - Path: `hr-contract-archive`
  - Response mode: `responseNode` (workflow must end with Respond to Webhook nodes)
  - CORS/Origins: `allowedOrigins: *`
- **Input/Output:**
  - Input: external HTTP request
  - Output → `Validate & Enrich Payload`
- **Version notes:** TypeVersion `2` (modern webhook node options).
- **Failure modes / edge cases:**
  - Payload not matching expected shape (e.g., missing `body` wrapper depending on sender) is handled in next Code node.
  - Large file uploads may hit n8n instance limits (body size / binary data constraints).

#### Node: Validate & Enrich Payload
- **Type / role:** `Code` — strict validation + canonical metadata generation.
- **Key configuration choices (interpreted):**
  - Reads payload from `$input.first().json.body` or `$input.first().json`.
  - Required guards:
    - `employeeName` must exist and not be empty.
    - Requires either `fileUrl` (remote) or `filename` (binary upload indicator).
  - Normalizes employee name:
    - Strips special chars except spaces, hyphen, apostrophe.
    - Collapses whitespace.
    - Title-cases each word.
  - Employee ID normalization:
    - Uppercases and strips invalid chars, fallback `EMP-{Date.now()}` if absent.
  - Contract type slug:
    - Uppercase, hyphenated, alnum-only.
  - Date logic:
    - `effectiveDate` defaults to today (`YYYY-MM-DD`)
    - `filedAt` is ISO timestamp
    - `yearFolder` derived from `effectiveDate`
  - File handling:
    - Derives `originalFilename` from `filename` or last segment of `fileUrl`.
    - Allowed extensions: `pdf, jpg, jpeg, png, docx, tiff` (throws otherwise).
    - Builds `structuredFilename`:  
      `{employeeId}_{LASTNAME}_{CONTRACTTYPESLUG}_{effectiveDate}.{ext}`
    - Sets `mimeType` via map (PDF/images/DOCX/TIFF).
  - Drive folder path:
    - `department` defaults to `General`
    - `driveFolderPath`: `HR/Contracts/{year}/{department}/{employeeName}`
  - Uses environment/workflow variables:
    - `rootFolderId: $vars.GDRIVE_ROOT_FOLDER_ID || 'root'`
  - Notification default:
    - `notifyEmployee: body.notifyEmployee !== false` (defaults true)
- **Input/Output:**
  - Input: webhook payload
  - Output → `Airtable - Duplicate Check`
- **Version notes:** TypeVersion `2` for Code node (supports modern runtime).
- **Failure modes / edge cases:**
  - Invalid file extension throws error (must be handled via error path wiring if desired).
  - `department` or `employeeEmail` may be empty; downstream nodes should tolerate.
  - Uses `$vars.GDRIVE_ROOT_FOLDER_ID`: if missing, uploads may end up in Drive root unless Drive nodes override folder usage.

---

### Block B — Duplicate Prevention (Pre-Upload) + Conflict Response

**Overview:**  
Searches Airtable for an existing filed contract for the employee. If found, immediately returns a 409 response and stops further processing.

**Nodes involved:**  
- Airtable - Duplicate Check  
- IF Duplicate Exists?  
- Respond - 409 Duplicate  

#### Node: Airtable - Duplicate Check
- **Type / role:** `Airtable` — preflight search to prevent duplicates.
- **Key configuration choices:**
  - Operation: `search`
  - Filter formula:  
    `AND({Employee ID} = '{{ $json.employeeId }}', {Contract Received} = TRUE())`
  - Returns selected fields: `Employee ID, Employee Name, Contract Received, Contract URL, Filed At`
  - **Base/Table are not set in the JSON** (empty value in `base.value` and `table.value`), implying you must configure them.
- **Credentials:** Airtable Personal Access Token.
- **Input/Output:**
  - Input: validated meta from `Validate & Enrich Payload`
  - Output → `IF Duplicate Exists?`
- **Failure modes / edge cases:**
  - Misconfigured base/table/field names cause Airtable API errors.
  - If `{Contract Received}` is not boolean or named differently, formula fails or returns no matches.
  - Rate limits / intermittent Airtable API timeouts.

#### Node: IF Duplicate Exists?
- **Type / role:** `IF` — branch based on Airtable search result count.
- **Condition:**
  - `({{ $json.records?.length ?? 0 }}) > 0`
- **Input/Output:**
  - True branch → `Respond - 409 Duplicate`
  - False branch → `Has Remote URL?`
- **Failure modes / edge cases:**
  - If Airtable node output structure changes (e.g., not `records`), condition might mis-evaluate.

#### Node: Respond - 409 Duplicate
- **Type / role:** `Respond to Webhook` — terminates request early with conflict.
- **Key configuration choices:**
  - HTTP code: `409`
  - JSON body includes:
    - `existingContractUrl` from first Airtable match
    - `filedAt` from Airtable
    - Message references normalized name/id from `Validate & Enrich Payload`
- **Dependencies:** Uses expression references to both:
  - `$('Validate & Enrich Payload').first().json...`
  - `$json.records[0].fields[...]` (from Airtable result)
- **Failure modes / edge cases:**
  - If Airtable returns records but fields are missing (`Contract URL`, `Filed At`), response body may render empty values or error if indexing fails (e.g., `records[0]` missing).

---

### Block C — Upload Chain (Remote vs Binary → UploadToURL → Drive Folder Resolve/Create → Drive Upload)

**Overview:**  
Determines whether the input is a remote URL or a binary file upload, uploads to UploadToURL accordingly, extracts a stable CDN URL, finds or creates the destination Drive folder, then uploads the document to Google Drive with the structured filename.

**Nodes involved:**  
- Has Remote URL?  
- Upload to URL - Remote  
- Upload to URL - Binary  
- Extract CDN URL  
- Drive - Find Employee Folder  
- Resolve Folder ID  
- Needs New Folder?  
- Drive - Create Employee Folder  
- Merge Folder ID  
- Drive - Upload Document  

#### Node: Has Remote URL?
- **Type / role:** `IF` — selects upload mode.
- **Condition:** checks `Validate & Enrich Payload.fileUrl` is not empty.
  - Left value: `{{ $('Validate & Enrich Payload').first().json.fileUrl }}`
  - Operator: string `notEmpty`
- **Input/Output:**
  - True → `Upload to URL - Remote`
  - False → `Upload to URL - Binary`
- **Failure modes / edge cases:**
  - If a sender provides both `fileUrl` and binary, the workflow will choose remote path.
  - If `fileUrl` exists but is invalid/unreachable, UploadToURL remote upload fails later.

#### Node: Upload to URL - Remote
- **Type / role:** `UploadToURL community node` (`n8n-nodes-uploadtourl.uploadToUrl`) — uploads a remote file.
- **Key configuration:** operation `uploadFile` (details depend on the community node’s internal defaults).
- **Credentials:** `Upload to URL API`
- **Input/Output:**
  - Input: output of `Has Remote URL?` (still carrying metadata)
  - Output → `Extract CDN URL`
- **Failure modes / edge cases:**
  - Community node must be installed on the n8n instance.
  - Remote fetch may fail (403/404, auth-required URLs, timeouts).
  - Provider may return different response structure; handled partially by extractor.

#### Node: Upload to URL - Binary
- **Type / role:** `UploadToURL community node` — uploads binary data.
- **Key configuration:** operation `uploadFile`
- **Credentials:** same as remote
- **Input/Output:** Output → `Extract CDN URL`
- **Failure modes / edge cases:**
  - Webhook must actually receive binary in a supported way (commonly `multipart/form-data`).
  - If no binary property is mapped/available, upload returns error or empty URL.

#### Node: Extract CDN URL
- **Type / role:** `Code` — normalizes UploadToURL response and merges with validated metadata.
- **Key logic:**
  - Tries multiple possible URL fields: `url`, `link`, `data.url`, `file.url`, `shortUrl`
  - Forces `https://` (replaces `http://`)
  - Adds `uploadId`, `fileSizeBytes` if present
  - Throws if no CDN URL is found, including raw response for debugging.
- **Input/Output:** Output → `Drive - Find Employee Folder`
- **Failure modes / edge cases:**
  - UploadToURL response schema changes; extractor might miss the correct field.
  - `cdnUrl.replace(...)` assumes `cdnUrl` is a string; non-string values could error.

#### Node: Drive - Find Employee Folder
- **Type / role:** `Google Drive` — searches for existing folder.
- **Key configuration:** operation `search`
- **Credentials:** Google Drive OAuth2.
- **Important note:** The node does not show the actual query in the provided JSON. To work correctly, it must be configured to search within the intended parent path/root and match folder name/path derived from metadata.
- **Input/Output:** Output → `Resolve Folder ID`
- **Failure modes / edge cases:**
  - Misconfigured search query may return nothing or wrong folder.
  - Permissions: OAuth account must have access to the HR root folder.

#### Node: Resolve Folder ID
- **Type / role:** `Code` — decides whether folder exists.
- **Key logic:**
  - Reads first folder from `searchResult.files[0]` or `searchResult.items[0]`
  - Sets:
    - `existingFolderId`
    - `needsNewFolder = !folderId`
  - Merges with metadata from `Extract CDN URL`
- **Input/Output:** Output → `Needs New Folder?`
- **Failure modes / edge cases:**
  - If Drive search returns non-folder items first, you may pick an incorrect ID unless the Drive search is constrained to folders.

#### Node: Needs New Folder?
- **Type / role:** `IF` — branch: create folder or reuse.
- **Condition:** `needsNewFolder` is true.
- **Input/Output:**
  - True → `Drive - Create Employee Folder`
  - False → `Merge Folder ID`
- **Failure modes / edge cases:**
  - If search returns false negatives, workflow may create duplicate folders.

#### Node: Drive - Create Employee Folder
- **Type / role:** `Google Drive` — creates employee folder when missing.
- **Key configuration:** operation `createFolder`
- **Credentials:** Google Drive OAuth2.
- **Critical missing details:** Folder name and parent folder ID are not visible in the JSON; must be configured to create under `HR/Contracts/{Year}/{Department}/` (and likely employeeName).
- **Input/Output:** Output → `Merge Folder ID`
- **Failure modes / edge cases:**
  - Requires correct parent folder ID resolution. If parent doesn’t exist, folder creation may fail unless intermediate folders are also created (not shown explicitly in nodes).

#### Node: Merge Folder ID
- **Type / role:** `Code` — consolidates folder ID from either branch.
- **Key logic:**
  - `targetFolderId = input.id (new folder) OR meta.existingFolderId`
  - Throws if still missing.
- **Input/Output:** Output → `Drive - Upload Document`
- **Failure modes / edge cases:**
  - If Drive create folder returns a different schema (not `id`), merge fails.

#### Node: Drive - Upload Document
- **Type / role:** `Google Drive` — uploads file to resolved folder.
- **Key configuration choices:**
  - File name: `{{ $json.structuredFilename }}`
  - `driveId: MyDrive`
  - `folderId`: `{{ $json.targetFolderId }}`
  - OCR language left empty
- **Credentials:** Google Drive OAuth2.
- **Input/Output:** Output → `Airtable - Find Employee Record`
- **Failure modes / edge cases:**
  - This configuration does **not explicitly show binary input mapping**. In n8n, Drive upload typically needs binary property. You must ensure the node is configured to take binary from webhook (or downloaded from URL) depending on your approach.
  - If you rely on CDN URL rather than binary, you may need a download step (not present) to provide binary content to Drive.

---

### Block D — Airtable Upsert (Find → Update or Create → Merge)

**Overview:**  
Finds an employee record by Employee ID. If found, updates contract fields; if not found, creates a new record. Then normalizes both results into a single payload for downstream steps.

**Nodes involved:**  
- Airtable - Find Employee Record  
- Prepare Airtable Update  
- Airtable Record Exists?  
- Airtable - Update Existing Record  
- Airtable - Create New Record  
- Merge Airtable Result  

#### Node: Airtable - Find Employee Record
- **Type / role:** `Airtable` — searches for the employee record to upsert.
- **Key configuration:**
  - Operation: `search`
  - Filter formula:  
    `{Employee ID} = '{{ $('Merge Folder ID').first().json.employeeId }}'`
  - Selected fields: `Employee ID, Employee Name, Department, Contract Received`
  - **Base/Table not set** in JSON; must be configured.
- **Input/Output:** Output → `Prepare Airtable Update`
- **Failure modes / edge cases:**
  - Field naming mismatch (`Employee ID`) causes search failure.
  - Multiple matches: only the first record is used later.

#### Node: Prepare Airtable Update
- **Type / role:** `Code` — builds Drive URL and prepares branch decision.
- **Key logic:**
  - Reads Drive upload response: `$('Drive - Upload Document').first().json`
  - `driveFileId = driveResp.id`
  - Builds `driveUrl = https://drive.google.com/file/d/{id}/view`
  - Determines `existingRecord` from Airtable search
  - Outputs flags:
    - `airtableRecordId`
    - `airtableRecordExists`
- **Input/Output:** Output → `Airtable Record Exists?`
- **Failure modes / edge cases:**
  - If Drive upload response doesn’t include `id`, `driveUrl` becomes null; Airtable update stores null link.

#### Node: Airtable Record Exists?
- **Type / role:** `IF` — chooses update vs create.
- **Condition:** `airtableRecordExists` is true.
- **Input/Output:**
  - True → `Airtable - Update Existing Record`
  - False → `Airtable - Create New Record`

#### Node: Airtable - Update Existing Record
- **Type / role:** `Airtable` — updates existing employee row.
- **Key configuration:**
  - Operation: `update`
  - Columns mapped explicitly:
    - `Contract Received` = true
    - `Contract URL` = `{{ $json.driveUrl }}`
    - `CDN Backup URL` = `{{ $json.cdnUrl }}`
    - `Structured Filename`, `Drive Folder Path`
    - `Contract Type`, `Effective Date`
    - `Filed By`, `Filed At`
  - **Record ID**: not visible in JSON; in Airtable update operations, you must supply record ID (likely should be `{{$json.airtableRecordId}}`). This must be configured for the node to work.
- **Input/Output:** Output → `Merge Airtable Result`
- **Failure modes / edge cases:**
  - Missing record ID mapping leads to runtime failure.
  - Airtable field types mismatch (date fields, checkbox fields).

#### Node: Airtable - Create New Record
- **Type / role:** `Airtable` — creates a record when employee doesn’t exist.
- **Key configuration:**
  - Operation: `create`
  - Sets:
    - Employee identity: `Employee ID`, `Employee Name`, `Department`
    - Contract filing fields similar to update path
- **Input/Output:** Output → `Merge Airtable Result`
- **Failure modes / edge cases:**
  - Required fields in Airtable schema not provided will cause create failure.

#### Node: Merge Airtable Result
- **Type / role:** `Code` — unifies update/create outputs.
- **Key logic:**
  - `airtableRecordId = airtableResp.id || meta.airtableRecordId`
  - Outputs `airtableUpdated: true`
- **Input/Output:** Output → `Send Confirmation Email`
- **Failure modes / edge cases:**
  - Airtable update response schemas differ; ensure `airtableResp.id` is correct for both operations.

---

### Block E — Notification & Webhook Success Response

**Overview:**  
Sends an email confirmation and builds a structured JSON response including an audit trail, then returns it to the webhook caller with HTTP 201.

**Nodes involved:**  
- Send Confirmation Email  
- Build Final Response  
- Respond to Webhook  

#### Node: Send Confirmation Email
- **Type / role:** `Gmail` — sends confirmation message(s).
- **Key configuration:** operation `sendEmail`
- **Credentials:** Gmail OAuth2.
- **Important missing details:** Recipient(s), subject, and body are not visible in JSON; must be configured to:
  - Always notify HR
  - Optionally notify employee when `notifyEmployee` is true
- **Input/Output:** Output → `Build Final Response`
- **Failure modes / edge cases:**
  - Gmail OAuth scope/consent issues.
  - Sending to empty/invalid employee email if not provided.
  - Rate limiting or daily send limits.

#### Node: Build Final Response
- **Type / role:** `Code` — composes the final response payload.
- **Key fields produced:**
  - `success: true`
  - Employee info: `employeeId`, `employeeName`, `department`
  - Document info: `structuredFilename`, `contractType`, `effectiveDate`
  - Storage info: `driveFileId`, `driveUrl`, `driveFolderPath`, `cdnBackupUrl`
  - Airtable info: `airtableRecordId`
  - `auditTrail`: `filedBy`, `filedAt`, `uploadId`, `fileSizeBytes`
- **Input/Output:** Output → `Respond to Webhook`
- **Failure modes / edge cases:**
  - Assumes upstream nodes produced expected fields; if any are null, response still succeeds but may be incomplete.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` — returns HTTP response.
- **Key configuration:**
  - HTTP code: `201`
  - Body: `{{ $json }}` (passes through built response)
  - Content-Type: application/json
- **Failure modes / edge cases:**
  - If workflow errors before reaching this node and error handling isn’t wired, webhook call may hang or return n8n default errors.

---

### Block F — Error & Conflict Handling

**Overview:**  
Provides a standard error payload generator and a webhook error response node. Designed to connect any node’s error output to this handler.

**Nodes involved:**  
- Error Handler  
- Respond with Error  

#### Node: Error Handler
- **Type / role:** `Code` — normalizes error to JSON.
- **Key logic:**
  - Reads error message from possible locations:  
    `err.json.message || err.error.message || err.json.error || fallback`
  - Outputs: `{ success:false, error: msg, timestamp: ISO }`
  - `alwaysOutputData: true` to ensure it emits even in edge cases.
- **Input/Output:** Output → `Respond with Error`
- **Failure modes / edge cases:**
  - Only works if upstream error data is actually passed into this node via error connections.

#### Node: Respond with Error
- **Type / role:** `Respond to Webhook` — returns error to caller.
- **Key configuration:** HTTP 400, JSON body passthrough, Content-Type JSON.
- **Failure modes / edge cases:**
  - Must be reachable from error connections or the webhook will not get this structured response.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Overview | Sticky Note | Documentation / overview | — | — | # 🗂️ Legal & Compliance Document Archiving (includes setup checklist, credentials list, example payload) |
| Entry & Duplicate Check | Sticky Note | Documentation for entry block | — | — | ## 🚪 Entry, Validation & Duplicate Check (describes Webhook → Validate → Airtable Duplicate Check → IF) |
| Upload Chain | Sticky Note | Documentation for upload block | — | — | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Airtable Update | Sticky Note | Documentation for Airtable block | — | — | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Notification & Response | Sticky Note | Documentation for notification/response | — | — | ## 📧 Notification & Response (email, final response, 201) |
| Error & Conflict Handling | Sticky Note | Documentation for errors/conflicts | — | — | ## ⚠️ Error & Conflict Handling (409 duplicate, general 400 handler, connect node error outputs) |
| Webhook - Receive Contract | Webhook | Receive contract + metadata | — | Validate & Enrich Payload | ## 🚪 Entry, Validation & Duplicate Check (describes Webhook → Validate & Enrich → Airtable Duplicate Check → IF Duplicate?) |
| Validate & Enrich Payload | Code | Validate inputs; normalize fields; generate filename/path | Webhook - Receive Contract | Airtable - Duplicate Check | ## 🚪 Entry, Validation & Duplicate Check (describes Webhook → Validate & Enrich → Airtable Duplicate Check → IF Duplicate?) |
| Airtable - Duplicate Check | Airtable | Pre-upload duplicate search | Validate & Enrich Payload | IF Duplicate Exists? | ## 🚪 Entry, Validation & Duplicate Check (describes Webhook → Validate & Enrich → Airtable Duplicate Check → IF Duplicate?) |
| IF Duplicate Exists? | IF | Branch: stop on duplicate | Airtable - Duplicate Check | Respond - 409 Duplicate; Has Remote URL? | ## 🚪 Entry, Validation & Duplicate Check (describes Webhook → Validate & Enrich → Airtable Duplicate Check → IF Duplicate?) |
| Respond - 409 Duplicate | Respond to Webhook | Return 409 conflict JSON | IF Duplicate Exists? (true) | — | ## ⚠️ Error & Conflict Handling (409 duplicate, general 400 handler, connect node error outputs) |
| Has Remote URL? | IF | Choose remote vs binary upload | IF Duplicate Exists? (false) | Upload to URL - Remote; Upload to URL - Binary | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Upload to URL - Remote | UploadToURL (community) | Upload remote document to CDN | Has Remote URL? (true) | Extract CDN URL | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Upload to URL - Binary | UploadToURL (community) | Upload binary document to CDN | Has Remote URL? (false) | Extract CDN URL | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Extract CDN URL | Code | Extract stable CDN URL; merge metadata | Upload to URL - Remote / Binary | Drive - Find Employee Folder | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Drive - Find Employee Folder | Google Drive | Search for existing employee folder | Extract CDN URL | Resolve Folder ID | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Resolve Folder ID | Code | Determine existing folder ID vs create | Drive - Find Employee Folder | Needs New Folder? | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Needs New Folder? | IF | Branch: create folder or reuse | Resolve Folder ID | Drive - Create Employee Folder; Merge Folder ID | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Drive - Create Employee Folder | Google Drive | Create missing folder | Needs New Folder? (true) | Merge Folder ID | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Merge Folder ID | Code | Normalize target folder ID | Needs New Folder? (false) / Drive - Create Employee Folder | Drive - Upload Document | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Drive - Upload Document | Google Drive | Upload file to employee folder | Merge Folder ID | Airtable - Find Employee Record | ## ☁️ Upload to URL → Google Drive (describes remote/binary, folder creation, Drive upload) |
| Airtable - Find Employee Record | Airtable | Find employee record by ID | Drive - Upload Document | Prepare Airtable Update | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Prepare Airtable Update | Code | Build driveUrl; set up upsert flags | Airtable - Find Employee Record | Airtable Record Exists? | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Airtable Record Exists? | IF | Branch: update vs create record | Prepare Airtable Update | Airtable - Update Existing Record; Airtable - Create New Record | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Airtable - Update Existing Record | Airtable | Update existing Airtable row | Airtable Record Exists? (true) | Merge Airtable Result | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Airtable - Create New Record | Airtable | Create new Airtable row | Airtable Record Exists? (false) | Merge Airtable Result | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Merge Airtable Result | Code | Normalize Airtable output | Airtable - Update Existing Record / Create New Record | Send Confirmation Email | ## 🗃️ Airtable Update (find employee, update/create record, store redundancy URLs) |
| Send Confirmation Email | Gmail | Notify HR and optionally employee | Merge Airtable Result | Build Final Response | ## 📧 Notification & Response (email, final response, 201) |
| Build Final Response | Code | Create success + audit payload | Send Confirmation Email | Respond to Webhook | ## 📧 Notification & Response (email, final response, 201) |
| Respond to Webhook | Respond to Webhook | Return 201 JSON summary | Build Final Response | — | ## 📧 Notification & Response (email, final response, 201) |
| Error Handler | Code | Normalize errors to JSON | (via error connections) | Respond with Error | ## ⚠️ Error & Conflict Handling (409 duplicate, general 400 handler, connect node error outputs) |
| Respond with Error | Respond to Webhook | Return 400 JSON error | Error Handler | — | ## ⚠️ Error & Conflict Handling (409 duplicate, general 400 handler, connect node error outputs) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create required credentials**
   1. UploadToURL credential: create an **UploadToURL API** credential (per the community node’s instructions).
   2. Google Drive: create/connect **Google Drive OAuth2** credential with access to your HR Drive structure.
   3. Airtable: create an **Airtable Personal Access Token** credential with read/write access to the base/table.
   4. Gmail: connect **Gmail OAuth2** credential for sending emails (or replace with SMTP node if preferred).

2) **Set workflow/environment variables in n8n**
   1. Set `GDRIVE_ROOT_FOLDER_ID` to the Google Drive folder ID that represents your HR root.
   2. (From the sticky note checklist) also plan to set:
      - `AIRTABLE_BASE_ID`
      - `AIRTABLE_TABLE_NAME`  
      Note: the provided Airtable nodes are not wired to these variables automatically; you must select base/table in each Airtable node (or refactor to expressions if desired).

3) **Add the entry webhook**
   1. Add node: **Webhook**
      - Name: `Webhook - Receive Contract`
      - Method: `POST`
      - Path: `hr-contract-archive`
      - Response mode: `Response Node`
      - Allowed origins: `*`

4) **Add validation/enrichment code**
   1. Add node: **Code**
      - Name: `Validate & Enrich Payload`
      - Paste the logic that:
        - validates `employeeName`
        - requires `fileUrl` or `filename`
        - normalizes name and employeeId
        - builds `structuredFilename`, `mimeType`, `driveFolderPath`
        - sets `rootFolderId` from `$vars.GDRIVE_ROOT_FOLDER_ID`
   2. Connect: `Webhook - Receive Contract` → `Validate & Enrich Payload`

5) **Add Airtable duplicate pre-check**
   1. Add node: **Airtable**
      - Name: `Airtable - Duplicate Check`
      - Operation: `Search`
      - Base: select your HR base
      - Table: select your employee/contracts table
      - Filter formula:  
        `=AND({Employee ID} = '{{ $json.employeeId }}', {Contract Received} = TRUE())`
      - Fields to return: `Employee ID, Employee Name, Contract Received, Contract URL, Filed At`
   2. Connect: `Validate & Enrich Payload` → `Airtable - Duplicate Check`

6) **Add duplicate branch logic + 409 response**
   1. Add node: **IF**
      - Name: `IF Duplicate Exists?`
      - Condition: Number → `{{ $json.records?.length ?? 0 }}` greater than `0`
   2. Add node: **Respond to Webhook**
      - Name: `Respond - 409 Duplicate`
      - Response code: `409`
      - Response headers: `Content-Type: application/json`
      - Body: JSON expression using Airtable’s first record and validated employee fields.
   3. Connect:
      - `Airtable - Duplicate Check` → `IF Duplicate Exists?`
      - `IF Duplicate Exists? (true)` → `Respond - 409 Duplicate`

7) **Add remote-vs-binary selection**
   1. Add node: **IF**
      - Name: `Has Remote URL?`
      - Condition: String → `{{ $('Validate & Enrich Payload').first().json.fileUrl }}` is not empty
   2. Connect: `IF Duplicate Exists? (false)` → `Has Remote URL?`

8) **Install and add UploadToURL nodes**
   1. Install community node `n8n-nodes-uploadtourl` on your n8n instance (as required by your deployment method).
   2. Add node: **UploadToURL**
      - Name: `Upload to URL - Remote`
      - Operation: `uploadFile`
      - Credentials: UploadToURL API
      - Configure it to upload from the remote URL (how you map `fileUrl` depends on the node’s parameters).
   3. Add node: **UploadToURL**
      - Name: `Upload to URL - Binary`
      - Operation: `uploadFile`
      - Credentials: UploadToURL API
      - Configure it to upload from incoming binary data (ensure your webhook and node use the same binary property name).
   4. Connect:
      - `Has Remote URL? (true)` → `Upload to URL - Remote`
      - `Has Remote URL? (false)` → `Upload to URL - Binary`

9) **Extract CDN URL**
   1. Add node: **Code**
      - Name: `Extract CDN URL`
      - Implement logic to pick the CDN URL from possible response fields, enforce HTTPS, and merge with validated metadata.
   2. Connect:
      - `Upload to URL - Remote` → `Extract CDN URL`
      - `Upload to URL - Binary` → `Extract CDN URL`

10) **Find/Create Google Drive folder and upload file**
   1. Add node: **Google Drive**
      - Name: `Drive - Find Employee Folder`
      - Operation: `Search`
      - Configure the query to locate the employee folder represented by `driveFolderPath` under `GDRIVE_ROOT_FOLDER_ID` (exact configuration depends on Drive node search options).
   2. Add node: **Code**
      - Name: `Resolve Folder ID`
      - Extract first folder result ID and set `needsNewFolder`.
   3. Add node: **IF**
      - Name: `Needs New Folder?`
      - Condition: Boolean → `{{ $json.needsNewFolder }}` is true
   4. Add node: **Google Drive**
      - Name: `Drive - Create Employee Folder`
      - Operation: `Create Folder`
      - Configure:
        - Folder name: employee folder name (typically `employeeName`)
        - Parent folder: the resolved parent path (year/department).  
        Note: the provided workflow does not show intermediate folder creation; you may need additional nodes to ensure `HR/Contracts/{Year}/{Department}` exists.
   5. Add node: **Code**
      - Name: `Merge Folder ID`
      - Output `targetFolderId` from created folder or existing folder.
   6. Add node: **Google Drive**
      - Name: `Drive - Upload Document`
      - Operation: Upload
      - File name: `{{ $json.structuredFilename }}`
      - Folder ID: `{{ $json.targetFolderId }}`
      - Ensure the node is configured to receive actual **binary content** (if you only have URLs, add an HTTP Request “download file” step—this is not present in the provided JSON but is commonly required).
   7. Connect:
      - `Extract CDN URL` → `Drive - Find Employee Folder` → `Resolve Folder ID` → `Needs New Folder?`
      - `Needs New Folder? (true)` → `Drive - Create Employee Folder` → `Merge Folder ID`
      - `Needs New Folder? (false)` → `Merge Folder ID`
      - `Merge Folder ID` → `Drive - Upload Document`

11) **Airtable find + upsert**
   1. Add node: **Airtable**
      - Name: `Airtable - Find Employee Record`
      - Operation: `Search`
      - Filter formula: `{Employee ID} = '{{ $('Merge Folder ID').first().json.employeeId }}'`
   2. Add node: **Code**
      - Name: `Prepare Airtable Update`
      - Build `driveUrl` from Drive file ID and set `airtableRecordExists`.
   3. Add node: **IF**
      - Name: `Airtable Record Exists?`
      - Condition: Boolean → `{{ $json.airtableRecordExists }}` true
   4. Add node: **Airtable**
      - Name: `Airtable - Update Existing Record`
      - Operation: `Update`
      - **Must set Record ID** to `{{ $json.airtableRecordId }}` (this is required for updates).
      - Map fields as in the workflow: Contract Received, URLs, metadata, timestamps.
   5. Add node: **Airtable**
      - Name: `Airtable - Create New Record`
      - Operation: `Create`
      - Map fields as in the workflow.
   6. Add node: **Code**
      - Name: `Merge Airtable Result`
      - Normalize output to include final `airtableRecordId`.
   7. Connect:
      - `Drive - Upload Document` → `Airtable - Find Employee Record` → `Prepare Airtable Update` → `Airtable Record Exists?`
      - `Airtable Record Exists? (true)` → `Airtable - Update Existing Record` → `Merge Airtable Result`
      - `Airtable Record Exists? (false)` → `Airtable - Create New Record` → `Merge Airtable Result`

12) **Send confirmation email + respond**
   1. Add node: **Gmail**
      - Name: `Send Confirmation Email`
      - Operation: `Send Email`
      - Configure recipients:
        - HR recipient(s): fixed address(es) or derived field (not provided in JSON)
        - Employee recipient: conditional on `notifyEmployee` (may require IF + two email nodes, or dynamic “to” field)
      - Build HTML/text including Drive URL, CDN URL, filename, contract type, effective date.
   2. Add node: **Code**
      - Name: `Build Final Response`
      - Compose the final JSON response including auditTrail.
   3. Add node: **Respond to Webhook**
      - Name: `Respond to Webhook`
      - Response code: `201`
      - Body: `{{ $json }}`
   4. Connect: `Merge Airtable Result` → `Send Confirmation Email` → `Build Final Response` → `Respond to Webhook`

13) **Add generic error handling**
   1. Add node: **Code**
      - Name: `Error Handler`
      - Generate `{success:false, error, timestamp}` from incoming error object.
      - Enable “Always Output Data”.
   2. Add node: **Respond to Webhook**
      - Name: `Respond with Error`
      - Response code: `400`
      - Body: `{{ $json }}`
   3. Connect: `Error Handler` → `Respond with Error`
   4. In n8n editor, connect **node error outputs** (red/error connector) from critical nodes (Airtable, Drive, UploadToURL, Code nodes) into `Error Handler`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install **Upload to URL** community node via npm | From sticky note “Setup Checklist” (required for `n8n-nodes-uploadtourl.uploadToUrl`) |
| Required credentials: UploadToURL API, Google Drive OAuth2, Airtable API, Gmail/SMTP | From sticky note “Required Credentials” |
| Set n8n variable `GDRIVE_ROOT_FOLDER_ID` (HR root folder ID) | From sticky note “Setup Checklist” |
| Set n8n variables `AIRTABLE_BASE_ID` and `AIRTABLE_TABLE_NAME` | From sticky note “Setup Checklist” (note: nodes still need to be wired to these or configured manually) |
| Example webhook payload and tip about `multipart/form-data` binary field `file` | From sticky note “Example Webhook Payload” |

If you want, I can also propose concrete configurations for the **Google Drive search/create folder nodes** (query + parent folder strategy) and the **binary vs URL download** handling, since those parts are not fully specified in the exported node parameters.