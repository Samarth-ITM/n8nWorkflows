Validate shipments & generate freight documents with GPT-4o, Google Sheets & Drive

https://n8nworkflows.xyz/workflows/validate-shipments---generate-freight-documents-with-gpt-4o--google-sheets---drive-11913


# Validate shipments & generate freight documents with GPT-4o, Google Sheets & Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Validate shipments & generate freight documents with GPT-4o, Google Sheets & Drive

**Purpose:**  
When a new shipment row is added to a Google Sheet, the workflow validates the shipment details using an OpenAI (GPT‑4o) agent (with a structured JSON output). If valid, it generates a PDF shipment report (HTML → PDF via ConvertAPI), emails a download link to both shipper and receiver, uploads the PDF to Google Drive, and updates the Google Sheet with status + Drive link. If invalid, it emails the shipper with the mismatch reason and updates the sheet state accordingly.

**Primary use cases:**
- Automated compliance/consistency checks of shipping records before document generation
- Hands-free generation and distribution of shipment reports
- Centralized audit trail (Google Sheets state + Drive storage)

### 1.1 Input Reception (Google Sheets → items)
- Watches a specific spreadsheet/sheet for **newly added rows**, polling every minute.

### 1.2 Item Processing Loop
- Processes rows **one at a time** using Split In Batches.

### 1.3 AI Validation (GPT‑4o agent + structured JSON)
- Uses an OpenAI Agent configured with:
  - GPT‑4o chat model
  - Google Sheets “Carrier Details” tool as a reference lookup (Active carriers)
  - Structured Output Parser enforcing `{ valid, reason }`

### 1.4 Error Handling (Invalid shipments)
- If validation fails: email shipper with the reason and key shipment fields; update Google Sheet “State” to `Mismatch_Details`.

### 1.5 PDF Generation (Valid shipments)
- Builds an HTML shipment report via Code node → converts to PDF using ConvertAPI.

### 1.6 Distribution + Storage + Logging
- Emails the PDF link to shipper and receiver.
- Downloads the PDF file from ConvertAPI URL → uploads to Google Drive → updates Google Sheet with `shipment_pdf_dispatch` and Drive link.
- Loops back for the next row.

---

## 2. Block-by-Block Analysis

### Block A — Data Ingestion & Loop Control
**Overview:**  
Triggers on newly added Google Sheet rows and processes them sequentially to avoid parallel document generation and emailing issues.

**Nodes Involved:**
- Google Sheets Trigger
- Loop Over Items (Split In Batches)

#### Node: Google Sheets Trigger
- **Type / role:** `googleSheetsTrigger` — polling trigger that emits new rows.
- **Configuration (interpreted):**
  - Event: **rowAdded**
  - Polling: **every minute**
  - Spreadsheet: “Freight Documentation” (documentId `1Fdaa...UhEM`)
  - Sheet: “Sheet1” / `gid=0` (note: cached name shows “Sheet1”; other nodes refer to “Shipping Details” but also `gid=0`)
- **Outputs:** New row item(s) as JSON with columns as keys (e.g., `Shipment ID / Reference Number`, `Carrier Name`, emails, etc.)
- **Connections:** → `Loop Over Items`
- **Potential failures / edge cases:**
  - Google OAuth2 expired/insufficient scopes
  - Polling duplicates if sheet edits are interpreted as additions (depends on trigger behavior)
  - Column header changes break downstream expressions (exact header names are referenced)

#### Node: Loop Over Items
- **Type / role:** `splitInBatches` — iterates items in controlled batches (default batch size behavior if not set).
- **Configuration:**
  - No explicit batch size configured (uses n8n defaults; typically batch size = 1 unless changed in UI).
- **Connections:**
  - Input: from `Google Sheets Trigger`
  - Output (main, index 1): → `Validate Shipment details`
  - “Continue” loops return into this node from:
    - `Update State` (mismatch path)
    - `Append or update row in sheet` (success path)
- **Edge cases / failures:**
  - If no items are passed, downstream nodes won’t run.
  - If batch size is >1 (manually changed), some nodes referencing `$('Loop Over Items').item` may behave unexpectedly.

**Sticky note covering this block:**
- **“Data Ingestion & AI Validation”**  
  “Monitors Google Sheet for new shipment rows and processes them one at a time. Uses OpenAI to validate shipment data against carrier database and compliance rules”

---

### Block B — AI Validation (Agent + Tools + Structured Output)
**Overview:**  
Validates shipment fields against formatting/compliance rules and a carrier reference list. Produces a strict JSON result used for branching.

**Nodes Involved:**
- Validate Shipment details (AI Agent)
- OpenAI Chat Model
- Structured Output Parser
- Carrier Details (Google Sheets tool)

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the LLM backend for the agent.
- **Configuration:**
  - Model: **gpt-4o**
  - Options: default
- **Connections:** Provides `ai_languageModel` to → `Validate Shipment details`
- **Failures / edge cases:**
  - Missing/invalid OpenAI API key
  - Model not available for the account/region
  - Rate limiting / timeouts
  - Output variability mitigated by the structured output parser, but agent may still fail if it cannot comply

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON schema output.
- **Configuration:**
  - Example schema:
    ```json
    { "valid": "", "reason": "" }
    ```
  - Practically, downstream expects:
    - `valid` as boolean-like string/boolean
    - `reason` as string
- **Connections:** Provides `ai_outputParser` to → `Validate Shipment details`
- **Failures / edge cases:**
  - If the agent outputs non-JSON or mismatched types, parsing fails.
  - The schema example uses empty strings; the prompt demands boolean `true/false`. This mismatch can increase parse/type ambiguity.

#### Node: Carrier Details
- **Type / role:** `googleSheetsTool` — an AI tool the agent can call to query carrier reference data.
- **Configuration:**
  - Spreadsheet: same document (`1Fdaa...UhEM`)
  - Sheet: “Carrier Details” (`gid=839895459`)
  - Filter: `Status` = `Active`
- **Connections:** Exposed as `ai_tool` to → `Validate Shipment details`
- **Failures / edge cases:**
  - OAuth scope/permission issues
  - Sheet name/columns changed (e.g., missing `Status`)
  - Agent may not call the tool even when asked; prompt suggests “must exist in carrier_Details”, but enforcement depends on agent behavior.

#### Node: Validate Shipment details
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + tool use + structured output.
- **Configuration:**
  - Prompt includes shipment fields interpolated from `$json[...]` (current row item).
  - Validation rules include:
    - Shipment ID format
    - Complete sender/receiver addresses
    - Carrier existence in “carrier_Details” reference list (via tool)
    - Port format validity
    - Goods description matches HS codes
    - Quantity/Weight positive and consistent
    - HS/HSN codes compliance
  - Output requirement: **only JSON**:
    - `{ "valid": true/false, "reason": "..." }`
  - `hasOutputParser`: true
- **Connections:** main output → `If Validate`
- **Failures / edge cases:**
  - If any referenced column is missing/null, prompt content degrades and can cause false failures.
  - “carrier_Details” naming mismatch: the actual sheet tool is “Carrier Details”; prompt says `"carrier_Details"`. Agent tool selection may still work, but ambiguity can reduce reliability.
  - Strict branching later compares `$json.output.valid` to the string `"true"`; if parser returns boolean `true`, the IF may fail (see next block).

---

### Block C — Branching Decision
**Overview:**  
Routes flow to either mismatch handling or PDF generation based on AI output.

**Nodes Involved:**
- If Validate

#### Node: If Validate
- **Type / role:** `if` — conditional branching.
- **Configuration:**
  - Condition: `{{ $json.output.valid }}` **equals** `"true"` (string comparison)
- **Connections:**
  - True (output 0) → `Set Shipment Data`
  - False (output 1) → `Send Mismatch Items Details`
- **Edge cases / failures:**
  - **Type mismatch risk:** if `valid` is boolean `true` (not string), condition fails → wrongly treated as invalid.
  - If the agent output structure differs (e.g., `valid` at root, not under `output`), condition fails.

---

### Block D — Error Handling (Notify + Log State)
**Overview:**  
Sends an email to the shipper with the AI “reason” and updates the sheet state to indicate mismatch.

**Nodes Involved:**
- Send Mismatch Items Details (Gmail)
- Update State (Google Sheets)

#### Node: Send Mismatch Items Details
- **Type / role:** `gmail` — sends email notification to shipper.
- **Configuration:**
  - To: `{{ $('Loop Over Items').item.json['Shipper email'] }}`
  - Subject: “⚠️ Action Required: Shipment Item Mismatch Detected”
  - Body includes `{{ $json.output.reason }}` plus shipment field recap.
  - `appendAttribution`: false
  - emailType: text (body includes markdown-like bold; Gmail text mode may show raw `**...**`)
- **Connections:** → `Update State`
- **Failures / edge cases:**
  - Gmail OAuth2 missing/expired
  - Invalid email address in sheet
  - If `$json.output.reason` missing, email contains blank issue
  - Using text mode with markdown formatting may reduce readability

#### Node: Update State
- **Type / role:** `googleSheets` — append or update row to track state.
- **Configuration:**
  - Operation: **appendOrUpdate**
  - Matching column: `Shipment ID / Reference Number`
  - Sets:
    - `State` = `Mismatch_Details`
    - `Shipment ID / Reference Number` from loop item
  - Target sheet: “Shipping Details” but `gid=0` (same as trigger; ensure correct sheet tab)
- **Connections:** → `Loop Over Items` (continue loop)
- **Failures / edge cases:**
  - If Shipment ID is missing/duplicate, updates may overwrite wrong row or create new row unexpectedly.
  - Permissions/sheet protected ranges can block updates.

**Sticky note covering this block:**
- **“Error Handling”**  
  “Notifies shipper of validation errors and logs mismatch state”

---

### Block E — PDF Generation (HTML → PDF via ConvertAPI)
**Overview:**  
Constructs a styled HTML shipment report, converts it to PDF using ConvertAPI, and extracts the resulting PDF URL for later emailing/downloading.

**Nodes Involved:**
- Set Shipment Data (Code)
- PDF Generate (HTTP Request)
- Get PDF (Extract from File)

#### Node: Set Shipment Data
- **Type / role:** `code` — generates HTML and attaches it as binary data (`data`) to the item.
- **Configuration details:**
  - Reads data from: `$('Loop Over Items').first().json` (note: uses **first()**, not current item)
  - Builds HTML string with shipment fields:
    - Shipment ID, shipper, receiver, goods description, qty, weight, HS codes, carrier name
  - Creates binary:
    - `mimeType: 'text/html'`
    - `fileName: 'Shipment_Report.html'`
    - assigns to `items[0].binary.data`
- **Connections:** → `PDF Generate`
- **Edge cases / failures:**
  - **Potential bug:** using `first()` instead of `item` can generate the wrong PDF when multiple items are in play.
  - Missing fields produce `undefined` in HTML.
  - Very large HTML could increase memory usage.

#### Node: PDF Generate
- **Type / role:** `httpRequest` — calls ConvertAPI to convert HTML to PDF.
- **Configuration:**
  - POST `https://v2.convertapi.com/convert/html/to/pdf`
  - Content-Type: `multipart/form-data`
  - Body:
    - `StoreFile=true`
    - `File` from binary field `data`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE` (must be replaced)
  - Response format: **file**
  - Notes in flow enabled
- **Connections:** → `Get PDF`
- **Failures / edge cases:**
  - Missing/invalid ConvertAPI token
  - ConvertAPI quota/rate limiting
  - If binary field name changes, conversion fails
  - Response format/file parsing differences could affect downstream nodes

#### Node: Get PDF
- **Type / role:** `extractFromFile` (`fromJson`) — parses JSON from the ConvertAPI response file to get `Files[0].Url`.
- **Configuration:**
  - Operation: `fromJson`
  - Expects the ConvertAPI response body (stored as a file/binary) to be JSON content.
- **Connections:** → `Send Shipment PDF To Sender`
- **Failures / edge cases:**
  - If ConvertAPI returns non-JSON error payload, parse fails.
  - If `Files[0].Url` missing, later URL references break.

**Sticky note covering this block:**
- **“PDF Generation”**  
  “Creates HTML report and converts to PDF via ConvertAPI”

---

### Block F — Email Distribution, Drive Storage & Sheet Logging
**Overview:**  
Sends the ConvertAPI PDF link to both parties, downloads the PDF, stores it in Drive, and updates the tracking sheet.

**Nodes Involved:**
- Send Shipment PDF To Sender (Gmail)
- Send Shipment PDF To Receiver (Gmail)
- Download File (HTTP Request)
- Upload file (Google Drive)
- Append or update row in sheet (Google Sheets)

#### Node: Send Shipment PDF To Sender
- **Type / role:** `gmail` — emails shipper with PDF link.
- **Configuration:**
  - To: `{{ $('Loop Over Items').item.json['Shipper email'] }}`
  - Subject: “📦 Shipment Report”
  - Body includes ConvertAPI link: `{{ $json.data.Files[0].Url }}`
  - Uses several loop item fields for personalization
  - `appendAttribution`: false; `emailType: text` (message includes emoji and markdown-like link formatting)
- **Connections:** → `Send Shipment PDF To Receiver`
- **Failures / edge cases:**
  - `$json.data.Files[0].Url` must exist (depends on Get PDF output structure)
  - Text mode may not render markdown link `[View Shipment Report]:URL` as intended

#### Node: Send Shipment PDF To Receiver
- **Type / role:** `gmail` — emails receiver with PDF link.
- **Configuration:** Similar to sender, but `sendTo` uses `Receiver email`.
- **Connections:** → `Download File`
- **Failures / edge cases:** same as above + invalid receiver email

#### Node: Download File
- **Type / role:** `httpRequest` — downloads the PDF from ConvertAPI-hosted URL.
- **Configuration:**
  - URL: `{{ $('Get PDF').item.json.data.Files[0].Url }}`
  - Default response handling (no explicit “download as file” configuration shown)
- **Connections:** → `Upload file`
- **Failures / edge cases:**
  - The node may return binary or JSON depending on n8n settings/defaults; Drive upload usually expects binary.
  - Link expiration from ConvertAPI can break download.
  - Large PDFs may time out.

#### Node: Upload file
- **Type / role:** `googleDrive` — uploads the downloaded PDF to a folder.
- **Configuration:**
  - Drive: “My Drive”
  - Folder: “Freight Documentation” (folderId `16aQLw-...`)
  - Name: `{{ $('Loop Over Items').item.json["Shipment ID / Reference Number"] }}`
  - Binary source: implied (must be the downloaded PDF binary from previous node)
- **Connections:** → `Append or update row in sheet`
- **Failures / edge cases:**
  - If Download File didn’t output binary as expected, upload fails.
  - Missing Drive permissions / folder access
  - File naming collisions (same Shipment ID) may overwrite or create duplicates depending on Drive node behavior/settings

#### Node: Append or update row in sheet
- **Type / role:** `googleSheets` — logs successful dispatch state and Drive link.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Matching column: `Shipment ID / Reference Number`
  - Sets:
    - `State` = `shipment_pdf_dispatch`
    - `Drive_link` = `{{ $json.webViewLink }}`
    - Shipment ID from loop item
  - Sheet: “Shipping Details” (`gid=0`)
- **Connections:** → `Loop Over Items` (continue loop)
- **Failures / edge cases:**
  - If `$json.webViewLink` missing (Drive node output changes), link is blank.
  - Duplicate IDs can cause unintended updates.

**Sticky note covering this block:**
- **“Storage and Logging”**  
  “Uploads PDF to Google Drive and updates sheet with status and link.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Sheets Trigger | googleSheetsTrigger | Trigger on new shipment rows | — | Loop Over Items | ## Data Ingestion & AI Validation<br>- Monitors Google Sheet for new shipment rows and processes them one at a time<br>- Uses OpenAI to validate shipment data against carrier database and compliance rules |
| Loop Over Items | splitInBatches | Sequential processing / loop control | Google Sheets Trigger; Update State; Append or update row in sheet | Validate Shipment details | ## Data Ingestion & AI Validation<br>- Monitors Google Sheet for new shipment rows and processes them one at a time<br>- Uses OpenAI to validate shipment data against carrier database and compliance rules |
| Validate Shipment details | langchain agent | AI validation + tool use + structured output | Loop Over Items; OpenAI Chat Model (AI); Structured Output Parser (AI); Carrier Details (AI tool) | If Validate | ## Data Ingestion & AI Validation<br>- Monitors Google Sheet for new shipment rows and processes them one at a time<br>- Uses OpenAI to validate shipment data against carrier database and compliance rules |
| OpenAI Chat Model | lmChatOpenAi | LLM backend (GPT‑4o) | — | Validate Shipment details (ai_languageModel) |  |
| Structured Output Parser | outputParserStructured | Enforce JSON response schema | — | Validate Shipment details (ai_outputParser) |  |
| Carrier Details | googleSheetsTool | AI tool: query active carriers reference | — | Validate Shipment details (ai_tool) |  |
| If Validate | if | Branch valid vs invalid | Validate Shipment details | Set Shipment Data; Send Mismatch Items Details |  |
| Send Mismatch Items Details | gmail | Notify shipper of validation errors | If Validate (false path) | Update State | ## Error Handling<br>Notifies shipper of validation errors and logs mismatch state |
| Update State | googleSheets | Log mismatch state in sheet | Send Mismatch Items Details | Loop Over Items | ## Error Handling<br>Notifies shipper of validation errors and logs mismatch state |
| Set Shipment Data | code | Generate HTML report as binary | If Validate (true path) | PDF Generate | ## PDF Generation<br>Creates HTML report and converts to PDF via ConvertAPI |
| PDF Generate | httpRequest | Convert HTML → PDF (ConvertAPI) | Set Shipment Data | Get PDF | ## PDF Generation<br>Creates HTML report and converts to PDF via ConvertAPI |
| Get PDF | extractFromFile | Parse ConvertAPI JSON to get PDF URL | PDF Generate | Send Shipment PDF To Sender | ## PDF Generation<br>Creates HTML report and converts to PDF via ConvertAPI |
| Send Shipment PDF To Sender | gmail | Email shipper PDF link | Get PDF | Send Shipment PDF To Receiver | ## Storage and Logging<br>Uploads PDF to Google Drive and updates sheet with status and link. |
| Send Shipment PDF To Receiver | gmail | Email receiver PDF link | Send Shipment PDF To Sender | Download File | ## Storage and Logging<br>Uploads PDF to Google Drive and updates sheet with status and link. |
| Download File | httpRequest | Download PDF from ConvertAPI URL | Send Shipment PDF To Receiver | Upload file | ## Storage and Logging<br>Uploads PDF to Google Drive and updates sheet with status and link. |
| Upload file | googleDrive | Upload PDF to Drive folder | Download File | Append or update row in sheet | ## Storage and Logging<br>Uploads PDF to Google Drive and updates sheet with status and link. |
| Append or update row in sheet | googleSheets | Log Drive link + success state | Upload file | Loop Over Items | ## Storage and Logging<br>Uploads PDF to Google Drive and updates sheet with status and link. |
| Sticky Note | stickyNote | Comment: sample sheet link | — | — | ## Sample Sheet<br>https://docs.google.com/spreadsheets/d/1FdaaTU8TXMZGKV8w8QikhwLg5Vm0pZbQ8U4vtIWUhEM/edit?gid=0#gid=0 |
| Workflow Overview | stickyNote | Comment: overall description + setup steps | — | — | ## How it works<br>This workflow automates freight documentation processing...<br><br>## Setup steps<br>1. Connect Google Sheets OAuth2 credentials<br>2. Add Gmail OAuth2 credentials<br>3. Set OpenAI API key for validation<br>4. Configure ConvertAPI bearer token in PDF Generate node<br>5. Connect Google Drive OAuth2 for file storage |
| Data Ingestion | stickyNote | Comment: ingestion/validation block | — | — | ## Data Ingestion & AI Validation<br>- Monitors Google Sheet for new shipment rows and processes them one at a time<br>- Uses OpenAI to validate shipment data against carrier database and compliance rules |
| PDF Generation | stickyNote | Comment: PDF generation block | — | — | ## PDF Generation<br>Creates HTML report and converts to PDF via ConvertAPI |
| Storage & Tracking | stickyNote | Comment: storage/logging block | — | — | ## Storage and Logging<br>Uploads PDF to Google Drive and updates sheet with status and link. |
| Error Handling | stickyNote | Comment: error handling block | — | — | ## Error Handling<br>Notifies shipper of validation errors and logs mismatch state |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Sheets Trigger**
   - Node: **Google Sheets Trigger**
   - Event: **Row Added**
   - Polling: **Every minute**
   - Select Spreadsheet: *Freight Documentation*
   - Select Sheet tab: the shipment intake sheet (the workflow uses `gid=0`)
   - Credentials: **Google Sheets OAuth2**

2) **Add loop controller**
   - Node: **Split In Batches** (rename to “Loop Over Items”)
   - Connect: `Google Sheets Trigger → Loop Over Items`
   - Keep default batch behavior (commonly 1)

3) **Create OpenAI model node**
   - Node: **OpenAI Chat Model** (LangChain)
   - Model: **gpt-4o**
   - Credentials: **OpenAI API key**
   - This node will be connected to the agent via the AI language model connector.

4) **Create Structured Output Parser**
   - Node: **Structured Output Parser** (LangChain)
   - Define schema example to include:
     - `valid` (boolean) and `reason` (string)
   - Connect to the agent via the AI output parser connector.

5) **Create Carrier reference tool**
   - Node: **Google Sheets Tool** (rename to “Carrier Details”)
   - Spreadsheet: *Freight Documentation*
   - Sheet: *Carrier Details*
   - Filter: `Status` equals `Active`
   - Credentials: Google Sheets OAuth2
   - Connect to the agent via the **AI Tool** connector.

6) **Create the AI Agent**
   - Node: **AI Agent** (rename to “Validate Shipment details”)
   - Prompt: “define” mode; paste the validation instructions and interpolate the shipment fields from the incoming item JSON.
   - Enable “Has Output Parser”
   - Connect:
     - `Loop Over Items (main) → Validate Shipment details (main)`
     - `OpenAI Chat Model (ai_languageModel) → Validate Shipment details`
     - `Structured Output Parser (ai_outputParser) → Validate Shipment details`
     - `Carrier Details (ai_tool) → Validate Shipment details`

7) **Create the branching IF**
   - Node: **IF** (rename to “If Validate”)
   - Condition: check agent result field:
     - In this workflow: `{{$json.output.valid}}` equals `"true"`
   - Connect: `Validate Shipment details → If Validate`

8) **Invalid path: notify shipper**
   - Node: **Gmail** (rename “Send Mismatch Items Details”)
   - To: `{{ $('Loop Over Items').item.json['Shipper email'] }}`
   - Subject/body: include `{{ $json.output.reason }}` and shipment details
   - Credentials: **Gmail OAuth2**
   - Connect: `If Validate (false) → Send Mismatch Items Details`

9) **Invalid path: update sheet state**
   - Node: **Google Sheets** (rename “Update State”)
   - Operation: **Append or Update**
   - Match column: `Shipment ID / Reference Number`
   - Set `State` = `Mismatch_Details`
   - Set Shipment ID value from loop item expression
   - Connect: `Send Mismatch Items Details → Update State`
   - Connect back to loop: `Update State → Loop Over Items` (to continue processing next row)

10) **Valid path: generate HTML**
   - Node: **Code** (rename “Set Shipment Data”)
   - Create HTML template and attach it as **binary** under field name `data` with `mimeType: text/html`.
   - Connect: `If Validate (true) → Set Shipment Data`

11) **Convert HTML to PDF via ConvertAPI**
   - Node: **HTTP Request** (rename “PDF Generate”)
   - Method: POST
   - URL: `https://v2.convertapi.com/convert/html/to/pdf`
   - Authentication header: `Authorization: Bearer <YOUR_CONVERTAPI_TOKEN>`
   - Send body as multipart/form-data:
     - `StoreFile=true`
     - `File` from binary property `data`
   - Configure response to be treated as **File**
   - Connect: `Set Shipment Data → PDF Generate`

12) **Extract PDF URL from ConvertAPI response**
   - Node: **Extract From File** (rename “Get PDF”)
   - Operation: **From JSON**
   - Connect: `PDF Generate → Get PDF`

13) **Email PDF link to shipper and receiver**
   - Node: **Gmail** (rename “Send Shipment PDF To Sender”)
     - To: shipper email expression
     - Include link: `{{ $json.data.Files[0].Url }}`
   - Node: **Gmail** (rename “Send Shipment PDF To Receiver”)
     - To: receiver email expression
     - Include same link
   - Connect: `Get PDF → Send Shipment PDF To Sender → Send Shipment PDF To Receiver`

14) **Download the PDF**
   - Node: **HTTP Request** (rename “Download File”)
   - URL: `{{ $('Get PDF').item.json.data.Files[0].Url }}`
   - Configure to download as binary if required by your n8n version/settings.
   - Connect: `Send Shipment PDF To Receiver → Download File`

15) **Upload to Google Drive**
   - Node: **Google Drive** (rename “Upload file”)
   - Folder: “Freight Documentation”
   - Name: Shipment ID expression
   - Input binary: the downloaded PDF binary from “Download File”
   - Credentials: **Google Drive OAuth2**
   - Connect: `Download File → Upload file`

16) **Update the sheet with success state + Drive link**
   - Node: **Google Sheets** (rename “Append or update row in sheet”)
   - Operation: **Append or Update**
   - Match column: `Shipment ID / Reference Number`
   - Set:
     - `State` = `shipment_pdf_dispatch`
     - `Drive_link` = `{{ $json.webViewLink }}`
   - Connect: `Upload file → Append or update row in sheet`
   - Loop back: `Append or update row in sheet → Loop Over Items`

17) **(Optional) Add Sticky Notes**
   - Add notes for: overview, ingestion/validation, error handling, PDF generation, storage/logging, and sample sheet link.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sample Sheet | https://docs.google.com/spreadsheets/d/1FdaaTU8TXMZGKV8w8QikhwLg5Vm0pZbQ8U4vtIWUhEM/edit?gid=0#gid=0 |
| How it works + Setup steps (credentials + ConvertAPI token) | Provided in workflow sticky note “Workflow Overview” |
| ConvertAPI authentication required | Replace `Bearer YOUR_TOKEN_HERE` in **PDF Generate** node |
| Credentials required | Google Sheets OAuth2, Gmail OAuth2, OpenAI API key, Google Drive OAuth2 |

