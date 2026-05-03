Generate PDF documents from HTML with PDF Generator API, Gmail and Supabase

https://n8nworkflows.xyz/workflows/generate-pdf-documents-from-html-with-pdf-generator-api--gmail-and-supabase-12554


# Generate PDF documents from HTML with PDF Generator API, Gmail and Supabase

## 1. Workflow Overview

**Purpose:** Automatically generate a PDF from an HTML template using incoming webhook data, validate the recipient email, deliver the PDF via Gmail, store it in Supabase Storage, and record the transaction in Postgres.

**Typical use cases:** invoices, confirmations, reports, certificates, and any dynamic HTML-to-PDF document pipeline with email delivery + storage + audit logging.

### 1.1 Input Reception
Receives a structured JSON payload via an HTTP POST webhook and acts as the workflow entry point.

### 1.2 Validation (Email Verification)
Verifies the client email via Hunter; rejects the request with an HTTP 400 response if invalid.

### 1.3 Template Retrieval + HTML Population
Fetches the correct HTML template from Postgres based on `documentType`, then fills `{{ ... }}` placeholders using the webhook payload.

### 1.4 PDF Generation
Converts the filled HTML to a PDF file using PDF Generator API.

### 1.5 Delivery + Storage
Sends the generated PDF to the client via Gmail and uploads it to Supabase Storage.

### 1.6 Transaction Recording
Inserts an audit/trace record into Postgres including metadata and the Supabase file URL.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception
**Overview:** Accepts a POST request with JSON describing the document, client, and data to inject into the template.  
**Nodes involved:** `Receive Document Request (Webhook)`

#### Node: Receive Document Request (Webhook)
- **Type / role:** Webhook trigger (HTTP entry point).
- **Key configuration:**
  - Method: **POST**
  - Path: `422d2c5b-d1b2-47a2-a1a6-5785885b6374` (also used as webhookId)
- **Key data expectations (from pinned example):**
  - `documentType` (e.g., `"invoice"`)
  - `client.name`, `client.email`, `client.address`
  - `request.id`, `request.createdAt`
  - `data` object containing template fields (e.g., `date`, `price`, etc.)
- **Connections:**
  - **Output →** `Verify Client Email`
- **Potential failures / edge cases:**
  - Missing required fields (later nodes will fail on expressions like `$json.client.email`).
  - Non-JSON body or wrong content-type if caller misconfigures request.
- **Version notes:** Node typeVersion `1` (standard webhook behavior).

---

### Block 2 — Validation (Email Verification)
**Overview:** Ensures the provided client email is valid before doing any PDF generation or sending.  
**Nodes involved:** `Verify Client Email`, `Email Is Valid?`, `Respond – Invalid Email`

#### Node: Verify Client Email
- **Type / role:** Hunter node (email verifier).
- **Key configuration:**
  - Operation: **Email Verifier**
  - Email: `={{ $json.client.email }}`
  - Credentials: `Hunter account 2`
- **Connections:**
  - **Input ←** `Receive Document Request (Webhook)`
  - **Output →** `Email Is Valid?`
- **Potential failures / edge cases:**
  - Hunter auth/plan limits; API rate limiting.
  - Hunter response may not contain the expected `status` (would break the IF logic).
  - Email field missing/null.
- **Version notes:** typeVersion `1`.

#### Node: Email Is Valid?
- **Type / role:** IF node to branch based on Hunter result.
- **Key configuration:**
  - Condition: `={{ $json.status }}` **equals** `"valid"`
- **Connections:**
  - **Input ←** `Verify Client Email`
  - **True output →** `Load HTML Template`
  - **False output →** `Respond – Invalid Email`
- **Potential failures / edge cases:**
  - If Hunter returns statuses like `accept_all`, `unknown`, etc., they will be treated as invalid (false branch).
  - If `$json.status` is undefined, strict validation may force false path.
- **Version notes:** IF node typeVersion `2.3` with conditions version `3`.

#### Node: Respond – Invalid Email
- **Type / role:** Respond to Webhook (early exit with error).
- **Key configuration:**
  - Response code: **400**
  - JSON body:
    ```json
    {
      "error": {
        "code": "400 Bad Request",
        "message": "Email validation failed: invalid email format."
      }
    }
    ```
- **Connections:**
  - **Input ←** `Email Is Valid?` (false branch)
  - **Output:** none (ends request)
- **Potential failures / edge cases:**
  - If the workflow continues in parallel elsewhere (not the case here), multiple responses could occur; here it’s a terminal branch.
- **Version notes:** Respond node typeVersion `1.5`.

---

### Block 3 — Template Retrieval + HTML Population
**Overview:** Loads the HTML template for the requested document type from Postgres, then replaces `{{path}}` placeholders with values from webhook JSON.  
**Nodes involved:** `Load HTML Template`, `Populate HTML Template`

#### Node: Load HTML Template
- **Type / role:** Postgres select from `document_templates`.
- **Key configuration:**
  - Schema: `public`
  - Table: `document_templates`
  - Filter: `document_type` equals `={{ $('Receive Document Request (Webhook)').item.json.documentType }}`
  - Limit: `1`
  - Output columns: `html_template`
- **Connections:**
  - **Input ←** `Email Is Valid?` (true branch)
  - **Output →** `Populate HTML Template`
- **Potential failures / edge cases:**
  - No matching template: result set empty; downstream code expects `.json.html_template` and may become `undefined`, producing failures or empty PDF.
  - DB auth/connectivity issues.
- **Version notes:** Postgres node typeVersion `2.6`.

#### Node: Populate HTML Template
- **Type / role:** Code node (template filling + filename generation).
- **Key configuration / logic (interpreted):**
  - Reads webhook payload:
    - `const data = $node["Receive Document Request (Webhook)"].json;`
  - Reads template string:
    - `const html = $node["Load HTML Template"].json.html_template;`
  - Fills placeholders of the form `{{ some.path }}` including array indexes like `items[0].field`.
  - If a placeholder cannot be resolved, it leaves the original `{{...}}` text in place.
  - Outputs:
    - `json.html` = filled HTML
    - `json.filename` = `${data.documentType}_${data.request.id}`
- **Connections:**
  - **Input ←** `Load HTML Template`
  - **Output →** `Convert HTML to PDF`
- **Potential failures / edge cases:**
  - If `Load HTML Template` returns no row, `html` becomes undefined and `.replace(...)` will throw.
  - If `data.request.id` missing, filename becomes e.g. `invoice_undefined` (later used in storage paths and subjects).
  - Placeholders not matching payload paths remain unreplaced (might be desired, but can also leak template tokens).
- **Version notes:** Code node typeVersion `2`.

---

### Block 4 — PDF Generation
**Overview:** Converts filled HTML into a PDF binary file using PDF Generator API and returns it as a file output.  
**Nodes involved:** `Convert HTML to PDF`

#### Node: Convert HTML to PDF
- **Type / role:** PDF Generator API node (HTML → PDF conversion).
- **Key configuration:**
  - Resource: `conversion`
  - HTML content: `={{ $json.html }}`
  - Filename: `={{ $json.filename }}`
  - Options:
    - `paper_size`: `a4`
    - `orientation`: `portrait`
    - Output: `file` (binary)
  - Credentials: `PDF Generator API`
- **Connections:**
  - **Input ←** `Populate HTML Template`
  - **Output →** `Upload PDF to Supabase Storage` and `Send PDF to Client Email` (fan-out)
- **Potential failures / edge cases:**
  - API auth errors, quota, invalid HTML, conversion timeouts.
  - Output binary property naming matters downstream; if the node returns a binary key different from what downstream expects, attachments/upload will fail.
- **Version notes:** Custom node `@pdfgeneratorapi/n8n-nodes-pdf-generator-api` typeVersion `1`.

---

### Block 5 — Delivery + Storage
**Overview:** Delivers the PDF to the client email and uploads the PDF to Supabase Storage.  
**Nodes involved:** `Send PDF to Client Email`, `Upload PDF to Supabase Storage`

#### Node: Send PDF to Client Email
- **Type / role:** Gmail send (outbound email with attachment).
- **Key configuration:**
  - To: `={{ $('Receive Document Request (Webhook)').item.json.client.email }}`
  - Subject: derived from `$json.filename`:
    - removes trailing `.pdf` if present
    - capitalizes first character
    - replaces first `_` with a space
  - Body: templated text including client name from webhook
  - Attachment configuration:
    - Uses UI attachment setting referencing a **binary property** named: `invoice_2025-0001.pdf`
- **Connections:**
  - **Input ←** `Convert HTML to PDF`
  - **Output:** none
- **Potential failures / edge cases (important):**
  - **Attachment binary key mismatch:** The workflow hard-codes `invoice_2025-0001.pdf`. If the PDF node outputs binary under a different property (or if filename changes), Gmail node will send without attachment or error.
  - Gmail OAuth scopes/refresh token expiration, “From” restrictions, sending limits.
  - If `$json.filename` does not include `.pdf`, subject formatting still works but may look odd.
- **Version notes:** Gmail node typeVersion `2.1`.

#### Node: Upload PDF to Supabase Storage
- **Type / role:** HTTP Request node to Supabase Storage object upload endpoint.
- **Key configuration:**
  - Method: **POST**
  - URL:
    - `https://nwnahpmsaiaekfwuxpft.supabase.co/storage/v1/object/documents/{{documentType}}/{{ $json.filename }}`
  - Body: **binaryData**
  - Input binary field name: `={{ $json.filename }}`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Content-Type: application/pdf`
    - `x-upsert: false`
- **Connections:**
  - **Input ←** `Convert HTML to PDF`
  - **Output →** `Record Document Transaction`
- **Potential failures / edge cases (important):**
  - **Authorization is placeholder** (`YOUR_TOKEN_HERE`): must be replaced with a valid Supabase service role key or a properly scoped JWT.
  - **Binary property mismatch:** `inputDataFieldName` expects a binary key equal to `$json.filename` (e.g., `invoice_2025-0001`). If the actual binary key differs (often something like `data`), upload will fail.
  - `x-upsert: false` will fail if the object already exists (409 conflict).
  - Supabase bucket/path assumptions: the URL implies bucket `documents` and folder per `documentType`.
- **Version notes:** HTTP Request node typeVersion `4.3`.

---

### Block 6 — Transaction Recording
**Overview:** Writes a row into a `documents` table to store metadata and the storage location of the generated PDF.  
**Nodes involved:** `Record Document Transaction`

#### Node: Record Document Transaction
- **Type / role:** Postgres execute query (INSERT + RETURNING).
- **Key configuration:**
  - SQL inserts: `document_type, document_name, request_id, client_name, client_email, file_path, file_url`
  - Uses `$1..$7` placeholders with “Query Replacement” values built from expressions referencing:
    - `Receive Document Request (Webhook)` (documentType, request.id, client.name, client.email)
    - `Populate HTML Template` (filename)
    - `Upload PDF to Supabase Storage` (Key)
    - Builds file_url as:
      - `https://nwnahpmsaiaekfwuxpft.supabase.co/storage/v1/object/{{Key}}`
  - Returns inserted `id`
- **Connections:**
  - **Input ←** `Upload PDF to Supabase Storage`
  - **Output:** none
- **Potential failures / edge cases:**
  - If Supabase response does not include `.Key`, expressions will fail or store nulls.
  - SQL schema mismatch (table/columns not present).
  - “Query replacement” formatting issues (quoting/escaping) depending on how n8n applies replacements; ensure values are passed as parameters, not string-concatenated SQL.
- **Version notes:** Postgres node typeVersion `2.6`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Document Request (Webhook) | n8n-nodes-base.webhook | Entry point: receive POST JSON payload | — | Verify Client Email | ## 1. Webhook input<br>This pinned data simulates a test JSON payload from the webhook. <br><br>It includes sample information such as the document type, client details, request metadata, and dynamic data used to populate the document template. |
| Verify Client Email | n8n-nodes-base.hunter | Verify client email with Hunter | Receive Document Request (Webhook) | Email Is Valid? | ## 2. Validation – Email verification |
| Email Is Valid? | n8n-nodes-base.if | Branch based on Hunter status == valid | Verify Client Email | Load HTML Template; Respond – Invalid Email | ## 2. Validation – Email verification |
| Respond – Invalid Email | n8n-nodes-base.respondToWebhook | Return HTTP 400 if email invalid | Email Is Valid? (false) | — | ## 2.1. Invalid email |
| Load HTML Template | n8n-nodes-base.postgres | Fetch HTML template by document_type | Email Is Valid? (true) | Populate HTML Template | ## 3. Generate PDF document |
| Populate HTML Template | n8n-nodes-base.code | Replace {{placeholders}} and create filename | Load HTML Template | Convert HTML to PDF | ## 3. Generate PDF document |
| Convert HTML to PDF | @pdfgeneratorapi/n8n-nodes-pdf-generator-api.pdfGeneratorApi | Convert HTML to PDF binary | Populate HTML Template | Upload PDF to Supabase Storage; Send PDF to Client Email | ## 3. Generate PDF document |
| Upload PDF to Supabase Storage | n8n-nodes-base.httpRequest | Upload PDF binary to Supabase Storage | Convert HTML to PDF | Record Document Transaction | ## 4. Email & storage |
| Send PDF to Client Email | n8n-nodes-base.gmail | Email PDF attachment via Gmail | Convert HTML to PDF | — | ## 4. Email & storage |
| Record Document Transaction | n8n-nodes-base.postgres | Insert transaction record into Postgres | Upload PDF to Supabase Storage | — | ## 5. Record transaction |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## Automate document creation from HTML using PDF Generator API, Gmail & Supabase<br><br>This workflow automates the creation, validation, and delivery of PDF documents generated from HTML templates using **[PDF Generator API](https://pdfgeneratorapi.com/)**.<br><br> ### How it works<br>1. Receives a document generation request via an HTTP **POST webhook**.<br>2. Validates the client’s email address using **Hunter Email Verification**.<br>    2.1 If the email is invalid, the workflow stops and returns an error response.<br>3. Generates the document by loading the HTML template, populating it with request data, and converting it into a PDF using **PDF Generator API**.<br>4. Sends the generated PDF to the client via **Gmail** and uploads it to **Supabase Storage**.<br>5. Records the document generation transaction in the database.<br><br>### Setup<br>Before using this workflow, make sure the following components are set up:<br><br>* A **POST webhook endpoint** that accepts structured JSON input.<br>* **Hunter API credentials** for validating client email addresses.<br>* **[PDF Generator API credentials](https://support.pdfgeneratorapi.com/en/article/how-to-integrate-pdf-generator-api-with-n8n-zulhli/#4-step-2-set-your-credentials)** for converting HTML documents to PDF.<br>* A **Postgres database** with tables for:<br>   * HTML document templates,<br>   * document generation records.<br>* **Gmail or SMTP credentials** for sending generated documents via email.<br>* **Supabase Storage** for storing generated PDF files. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | (see node content in workflow; applies to webhook block) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 2. Validation – Email verification |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 2.1. Invalid email |
| Sticky Note6 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 3. Generate PDF document |
| Sticky Note7 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 5. Record transaction |
| Sticky Note8 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 4. Email & storage |

> Note: Sticky notes are included as nodes in the workflow JSON; they don’t connect to the execution graph.

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: generate a unique path (store it; this is your endpoint)
   - (Optional) Add pinned sample payload similar to:
     - `documentType`, `client {name,email}`, `request {id,createdAt}`, `data {…}`

2) **Add Hunter email verification**
   - Add node: **Hunter**
   - Operation: **Email Verifier**
   - Email field: expression `{{$json.client.email}}`
   - Create/configure **Hunter API credentials** in n8n and select them.
   - Connect: **Webhook → Hunter**

3) **Branch on validity**
   - Add node: **IF**
   - Condition: `{{$json.status}}` equals `valid`
   - Connect: **Hunter → IF**

4) **Invalid branch response**
   - Add node: **Respond to Webhook**
   - Response code: **400**
   - Respond with: **JSON**
   - Body: the error object used in the workflow (or your own)
   - Connect: **IF (false) → Respond to Webhook**

5) **Load the HTML template from Postgres (valid branch)**
   - Add node: **Postgres**
   - Operation: **Select**
   - Schema: `public`
   - Table: `document_templates`
   - Where: `document_type` = `{{$('Webhook').item.json.documentType}}`
   - Limit: `1`
   - Output columns: `html_template`
   - Create/configure **Postgres credentials** in n8n.
   - Connect: **IF (true) → Postgres (Load HTML Template)**

6) **Populate the HTML**
   - Add node: **Code**
   - Paste logic that:
     - reads webhook JSON
     - reads `html_template`
     - replaces `{{path}}` placeholders
     - outputs `{ html: filledHtml, filename: documentType_requestId }`
   - Connect: **Load HTML Template → Populate HTML Template**

7) **Convert HTML to PDF (PDF Generator API)**
   - Add node: **PDF Generator API** (community/custom node)
   - Resource: **Conversion**
   - HTML content: `{{$json.html}}`
   - Filename: `{{$json.filename}}`
   - Options: A4, portrait, output as **file**
   - Create/configure **PDF Generator API credentials** in n8n.
   - Connect: **Populate HTML Template → Convert HTML to PDF**

8) **Upload to Supabase Storage**
   - Add node: **HTTP Request**
   - Method: **POST**
   - URL:
     - `https://<your-project>.supabase.co/storage/v1/object/documents/{{$('Webhook').item.json.documentType}}/{{$json.filename}}`
   - Send Body: **true**
   - Body Content Type: **Binary Data**
   - Input Data Field Name: set to the binary property that actually contains the PDF (align with the PDF node output)
   - Headers:
     - `Authorization: Bearer <SUPABASE_TOKEN>`
     - `Content-Type: application/pdf`
     - `x-upsert: false` (or true if you want overwrites)
   - Connect: **Convert HTML to PDF → Upload to Supabase Storage**

9) **Send email via Gmail**
   - Add node: **Gmail**
   - Operation: **Send**
   - To: `{{$('Webhook').item.json.client.email}}`
   - Subject: derive from filename (optional)
   - Message: include `client.name` (optional)
   - Attachments: select the PDF binary property from the PDF conversion output
   - Create/configure **Gmail OAuth2 credentials** in n8n.
   - Connect: **Convert HTML to PDF → Gmail**

10) **Record transaction in Postgres**
   - Add node: **Postgres**
   - Operation: **Execute Query**
   - SQL INSERT into `documents` with fields:
     - `document_type, document_name, request_id, client_name, client_email, file_path, file_url`
   - Map parameters from:
     - Webhook (`documentType`, `request.id`, `client.name`, `client.email`)
     - Populate HTML Template (`filename`)
     - Supabase upload response (key/path)
   - Connect: **Upload to Supabase Storage → Record Document Transaction**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PDF Generator API main site | https://pdfgeneratorapi.com/ |
| PDF Generator API credentials setup in n8n | https://support.pdfgeneratorapi.com/en/article/how-to-integrate-pdf-generator-api-with-n8n-zulhli/#4-step-2-set-your-credentials |
| Supabase upload auth header is a placeholder in this workflow (`Bearer YOUR_TOKEN_HERE`) and must be replaced | Supabase Storage integration |
| Attachment configuration in Gmail node appears hard-coded to `invoice_2025-0001.pdf` and should be aligned to the actual PDF binary property | Gmail delivery step |