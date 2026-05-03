Extract & organize email invoices with Gmail, Drive & OpenAI GPT

https://n8nworkflows.xyz/workflows/extract---organize-email-invoices-with-gmail--drive---openai-gpt-12173


# Extract & organize email invoices with Gmail, Drive & OpenAI GPT

## 1. Workflow Overview

**Workflow name:** `Invoice_Documentation.`  
**Purpose:** Automatically detect invoice emails in Gmail, extract invoice metadata with OpenAI, and store results in Google Sheets. If an invoice PDF exists, upload it to Google Drive and write back a Drive link; otherwise store a Gmail permalink. Finally, mark the email as read.

**Typical use cases**
- SMB/founder finance ops: capture invoice details from incoming emails into a single ledger (Google Sheets).
- Reduce manual downloading/filing of invoice PDFs; keep traceability to either the PDF in Drive or the originating Gmail message.

### 1.1 Email ingestion (poll + fetch full message)
Triggered hourly, fetches the full Gmail message including attachments.

### 1.2 Attachment normalization + PDF text extraction
Restructures Gmail attachments into individual items and extracts text from PDFs when possible.

### 1.3 Invoice detection (AI classification gate)
An AI agent determines if the email/attachments actually contain an invoice. Non-invoices stop.

### 1.4 Invoice field extraction (AI structured extraction)
A second AI agent extracts required invoice fields into a strict schema (dates, amounts, VAT, vendor, etc.).

### 1.5 Branch: attachment-based invoice vs body-text invoice
If there are attachments, the workflow uploads the file to Drive and updates Sheets with a Drive link. If not, it updates Sheets with a Gmail link.

### 1.6 Finalization
Marks the processed Gmail message as read.

---

## 2. Block-by-Block Analysis

### Block 1 — Email ingestion (poll + fetch full message)

**Overview:** Polls Gmail on a schedule and retrieves the complete message content with attachments to prepare it for processing.

**Nodes involved:**
- `Gmail Trigger`
- `Get a message`

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — entry point; triggers workflow on a polling schedule.
- **Key configuration:**
  - Polling: **every hour**
  - Filters: empty (no label/query restriction configured).
- **Input / output:**
  - No input (trigger).
  - Output feeds `Get a message` with message metadata including `id`.
- **Credentials:** Gmail OAuth2.
- **Edge cases / failures:**
  - OAuth scope/consent issues (cannot list messages).
  - Polling may pick up unwanted emails without filters (noise, higher cost downstream).
  - Duplicates possible depending on trigger state/history; consider label-based filtering or “processed” label.

#### Node: Get a message
- **Type / role:** `n8n-nodes-base.gmail` (operation: `get`) — fetch full email including headers, plain text, and attachments.
- **Key configuration:**
  - `operation`: **get**
  - `messageId`: `={{ $json.id }}`
  - `simple`: false (returns rich message object)
  - `downloadAttachments`: true
  - Attachments prefix: `attachment_` (stored under `binary.attachment_...`)
- **Input / output:**
  - Input: `Gmail Trigger`
  - Output: `Code in JavaScript`
- **Credentials:** Gmail OAuth2.
- **Edge cases / failures:**
  - Large attachments may exceed instance limits/timeouts.
  - Some messages may not have `text` (HTML-only); the workflow relies heavily on `.json.text`.
  - Attachment download failures can yield missing `binary` keys.

**Sticky note coverage (context):**
- “## Gmail Trigger based on user customized interval … get gmail parameter inclusive text and attachments and extract all pdf files from attachments with js-code”

---

### Block 2 — Attachment normalization + PDF text extraction

**Overview:** Converts Gmail’s multiple attachments into one item per attachment and attempts PDF text extraction. Continues even if extraction fails.

**Nodes involved:**
- `Code in JavaScript`
- `Extract from File`

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — transforms the Gmail message’s attachment binaries into separate n8n items, each with a uniform binary property name (`binary.data`).
- **Key configuration choices:**
  - **Runs as “Function” (not per-item)**: uses `items.flatMap(...)` to output multiple items.
  - Looks for binary keys starting with `attachment_`, sorts them, then emits one item per attachment:
    - `json.attachmentKey`, `json.fileName`, `json.mimeType`
    - `binary.data = bin[k]`
  - `alwaysOutputData: true`
  - `onError: continueRegularOutput` (workflow continues even if something breaks in this node).
- **Input / output:**
  - Input: `Get a message`
  - Output (two branches):
    - To `Extract from File` (main index 0)
    - To `Merge` (main index 1) — passes attachment items forward for later Drive upload.
- **Edge cases / failures:**
  - If the email has **no attachments**, `keys` becomes empty and the node returns `[]`. That can prevent downstream nodes from running unless n8n still passes an empty execution (depends on node behaviors). This matters because the “text-invoice” path later expects to run even without attachments.
  - If Gmail returns binary keys not matching `attachment_` prefix, attachments won’t be processed.
  - If attachments exist but `bin[k]` lacks `fileName`/`mimeType`, naming/upload may degrade.

#### Node: Extract from File
- **Type / role:** `n8n-nodes-base.extractFromFile` — extracts text from PDF attachment binaries.
- **Key configuration:**
  - `operation`: `pdf`
  - `onError: continueRegularOutput` — if extraction fails (password-protected, scanned image, corrupted), continue with regular output.
- **Input / output:**
  - Input: `Code in JavaScript` (attachment items with `binary.data`)
  - Output: `Invoice Recognition Agent`
- **Edge cases / failures:**
  - Scanned PDFs without embedded text yield poor/empty extraction.
  - Non-PDF attachments still get passed in; extraction may fail or output empty text.
  - Very large PDFs can cause memory/timeouts.

**Sticky note coverage (context):**
- “## Extract the text from the pdf attachment if exists **otherwise ignore and further process**”

---

### Block 3 — Invoice detection (AI classification gate)

**Overview:** Uses an AI agent to decide whether the email contains an invoice (body and/or attachment). Only invoices proceed to extraction; others stop.

**Nodes involved:**
- `OpenAI Chat Model`
- `Structured Output Parser`
- `Invoice Recognition Agent`
- `Check wether it is an invoice`
- `No Operation, do nothing`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for the recognition agent.
- **Key configuration:**
  - Model: `gpt-5.1`
- **Input / output connections:**
  - Connected as **AI Language Model** to `Invoice Recognition Agent`.
- **Credentials:** OpenAI API credential.
- **Edge cases / failures:**
  - Model availability / API quota / rate limits.
  - Latency can slow hourly polling batches.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — forces the recognition agent to return structured JSON.
- **Schema:** Object with one required boolean:
  - `email_contains_invoice` (boolean)
- **Input / output connections:**
  - Connected as **AI Output Parser** to `Invoice Recognition Agent`.
- **Edge cases / failures:**
  - If model responds non-JSON, parsing fails; unlike Parser1, this one does **not** enable auto-fix.

#### Node: Invoice Recognition Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompt-driven classifier that evaluates email subject/body/headers and extracted attachment text.
- **Key configuration:**
  - `promptType`: define
  - `hasOutputParser`: true (uses `Structured Output Parser`)
  - Prompt references:
    - Subject/text/from/date via: `$('Get a message').all()[0].json...`
    - Attachment extraction via: `{{ $json.text }}` (from `Extract from File`)
- **Input / output:**
  - Input: `Extract from File` output items (one per attachment) with `json.text` from extraction.
  - Output: `Check wether it is an invoice`
- **Edge cases / failures:**
  - If there are multiple attachments, the agent runs per extracted item; you may get multiple classifications for the same email (one per attachment), which can duplicate downstream rows unless controlled.
  - If the email has no attachments and the earlier Code node produced no items, this block may never run (workflow might miss “body-only invoices”).
  - Prompt contains a “Policy toggle” sentence with typos (`fals`, `rale`)—doesn’t break execution but can confuse the model.

#### Node: Check wether it is an invoice
- **Type / role:** `n8n-nodes-base.if` — gating condition.
- **Condition:** `={{ $json.output.email_contains_invoice }}` is **true** (boolean true operator).
- **Branches:**
  - **True** → `Invoice Data extractor`
  - **False** → `No Operation, do nothing`
- **Edge cases / failures:**
  - If `output.email_contains_invoice` is missing due to parser failure, IF evaluation may error or be treated as false depending on runtime.
  - Multiple items (multiple attachments) can cause multiple “true” paths for one email.

#### Node: No Operation, do nothing
- **Type / role:** `n8n-nodes-base.noOp` — terminates non-invoice branch.
- **Edge cases:** None; it is a sink.

**Sticky note coverage (context):**
- “## Analyse the context of the mail and the attachments and process further in case it is an invoice **otherwise ignore and stop the process**”

---

### Block 4 — Invoice field extraction (AI structured extraction)

**Overview:** Extracts normalized invoice metadata into a strict schema. Parser auto-fixes formatting problems.

**Nodes involved:**
- `OpenAI Chat Model1`
- `Structured Output Parser1`
- `OpenAI Chat Model2` (connected to parser)
- `Invoice Data extractor`

#### Node: OpenAI Chat Model1
- **Type / role:** OpenAI chat model provider for the extractor agent.
- **Key configuration:** Model `gpt-5.1`
- **Connections:** AI Language Model → `Invoice Data extractor`
- **Credentials:** OpenAI API
- **Edge cases:** quota, rate limit, long prompts (invoice text can be large).

#### Node: Structured Output Parser1
- **Type / role:** structured JSON parser for extracted invoice fields.
- **Key configuration:**
  - `schemaType`: manual
  - `autoFix`: **true** (attempts to repair near-valid JSON to match schema)
  - Required fields:
    - `date_email` (date-time), `date_invoice` (date),
    - `invoice_nr`, `description`, `provider`,
    - `net_amount` (number), `vat` (number), `gross_amount` (number),
    - `label` enum: `saas|hardware|other`,
    - `currency` (ISO 4217)
- **Connections:** AI Output Parser → `Invoice Data extractor`
- **Additionally connected:** `OpenAI Chat Model2` as AI Language Model to this parser (used by the auto-fix mechanism).
- **Edge cases / failures:**
  - Auto-fix can still fail if the model output is far from schema.
  - “All fields required” may force hallucinated values (by design in the prompt).

#### Node: OpenAI Chat Model2
- **Type / role:** additional LLM connection used for parser auto-fix.
- **Key configuration:** Model `gpt-5.1`
- **Connections:** AI Language Model → `Structured Output Parser1`
- **Edge cases:** extra API calls (cost) when auto-fix engages.

#### Node: Invoice Data extractor
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — extracts and normalizes invoice fields.
- **Key configuration:**
  - `hasOutputParser`: true (uses `Structured Output Parser1`)
  - Prompt sources:
    1. Invoice text: `{{ $('Extract from File').item.json.text }}`
    2. Email text: `{{ $('Get a message').all()[0].json.text }}`
    3. Email date: `{{ $('Get a message').all()[0].json.headers.date }}`
  - Enforces: “VALID JSON”, all fields required, no null/empty, compute VAT% if needed, normalize formats.
- **Input / output:**
  - Input: from IF true branch.
  - Output: `Check wether it was a file-invoice or text invoice`
- **Edge cases / failures:**
  - If `Extract from File` did not produce `item.json.text` (e.g., extraction failed), the prompt’s invoice text may be empty, leading to weaker extraction.
  - If the email body has only HTML and no `.json.text`, missing context reduces accuracy.
  - Since all fields must exist, the model may invent invoice numbers or dates if not present.

**Sticky note coverage (context):**
- “## Extract the relevant invoice parameters … (fields list)”

---

### Block 5 — Branching: attachment invoice vs body-text invoice

**Overview:** Decides whether to treat the invoice as a file-based invoice (upload to Drive + Drive link) or a text-only invoice (store Gmail link).

**Nodes involved:**
- `Check wether it was a file-invoice or text invoice`
- `Document the invoice parameters`
- `Merge`
- `Upload file`
- `Update Invoice parameters`
- `Update text Invoice parameters`

#### Node: Check wether it was a file-invoice or text invoice
- **Type / role:** `n8n-nodes-base.if`
- **Condition:** checks whether `$('Get a message').all()[0].binary` **exists**.
  - This is intended to detect attachments, but it tests existence of **any binary**, not specifically an invoice PDF.
- **Branches:**
  - **True** → `Document the invoice parameters` (then Drive route)
  - **False** → `Update text Invoice parameters` (text-only route)
- **Edge cases / failures:**
  - If Gmail always provides some binary data when attachments exist, OK; but if the workflow’s earlier attachment handling changed keys, this may mis-detect.
  - If there are attachments that are not invoices (e.g., logos), it still takes the “file” path.
  - If Code node returned no items (no attachments), this IF might never be reached due to upstream item flow issues.

#### Node: Document the invoice parameters
- **Type / role:** `n8n-nodes-base.googleSheets` (operation: append) — writes extracted invoice fields to a sheet (initial insert before Drive link is known).
- **Key configuration:**
  - Operation: `append`
  - Spreadsheet: `invoices` (documentId placeholder `1 your_spreadsheet_id`)
  - Sheet: `gid=0`
  - Columns mapped from AI output:
    - Uses `={{ $json.output.<field> }}` for all invoice fields
  - Includes a `link` column in schema, but **not populated** in this append step.
- **Input / output:**
  - Input: from IF (file path)
  - Output: `Merge`
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - Sheet columns must match; mismatches cause append errors.
  - Using `append` can create duplicates if the same invoice is reprocessed.

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines AI-extracted invoice fields with attachment item(s) for upload.
- **Key configuration:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
- **Inputs:**
  - Input 1: from `Document the invoice parameters` (invoice fields)
  - Input 2: from `Code in JavaScript` (attachments) on input index 1
- **Output:** `Upload file`
- **Edge cases / failures:**
  - `combineByPosition` is fragile: if there are multiple attachments/items, you may combine the wrong invoice data with the wrong attachment or create multiple combined items.
  - If the code node produces 0 items (no attachments), merge output can be empty.

#### Node: Upload file
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads invoice file to Drive.
- **Key configuration:**
  - Target Drive: `My Drive`
  - Folder ID: `1rbR8RAmXZDCbqQAhypq2m_YDxYbOiXGa` (named “invoices”)
  - File name expression:
    - `Invoice_{{ $('Document the invoice parameters').item.json.provider }}_{{ $('Document the invoice parameters').item.json.date_invoice }}`
- **Inputs / outputs:**
  - Input: `Merge` output should include `binary.data` for file content.
  - Output: `Update Invoice parameters`
- **Credentials:** Google Drive OAuth2.
- **Edge cases / failures:**
  - The naming expression references `$('Document the invoice parameters').item.json.provider`, but that node’s output item structure is not guaranteed to contain those fields at that path (Google Sheets node outputs its own metadata). More robust would be `$('Invoice Data extractor').item.json.output.provider` or `$('Merge').item.json.provider`.
  - Folder permission issues; upload size limits; mimeType handling.
  - If multiple attachments exist, it may upload non-invoice files too.

#### Node: Update Invoice parameters
- **Type / role:** `n8n-nodes-base.googleSheets` — appends or updates the row with a Drive link.
- **Key configuration:**
  - Operation: `appendOrUpdate`
  - Matching column: `invoice_nr`
  - Link column built from Drive upload output:
    - `link = https://drive.google.com/file/d/{{ $json.id }}/view`
  - Other fields pulled from `$('Merge').item.json.<field>`
- **Input / output:**
  - Input: `Upload file` (expects `json.id` from Drive file)
  - Output: `Mark a message as read`
- **Edge cases / failures:**
  - If invoice numbers repeat across vendors, `appendOrUpdate` may overwrite the wrong row (matching solely on `invoice_nr`).
  - If the sheet has different header names/types, updates may fail silently or error.
  - If `Merge` didn’t carry expected fields (combine mismatch), values become empty.

#### Node: Update text Invoice parameters
- **Type / role:** `n8n-nodes-base.googleSheets` — appends/updates a row and stores a Gmail permalink when invoice is in email text.
- **Key configuration:**
  - Operation: `appendOrUpdate`
  - Matching column: `invoice_nr`
  - Link:
    - `={{ 'https://mail.google.com/mail/u/0/#inbox/' + $('Get a message').first().json.id }}`
  - Fields from `{{$json.output...}}`
- **Input / output:**
  - Input: IF false branch (no attachments)
  - Output: `Mark a message as read`
- **Edge cases / failures:**
  - Gmail permalinks can vary by mailbox/label; using `#inbox/` might not open if the message is archived/labelled.
  - Same overwrite risk as above due to matching only on `invoice_nr`.

**Sticky note coverage (context):**
- “## Was the invoice in attachments or in the mail body?”
- “## Document the paramters in sheet, uoload the invoice file to drive and update the sheet with the invoice-url … mark the invoice message as read …”
- “## Document the paramters in sheet, generate a link to the email”

---

### Block 6 — Finalization

**Overview:** Marks the Gmail message as read after successfully writing invoice data.

**Nodes involved:**
- `Mark a message as read`

#### Node: Mark a message as read
- **Type / role:** `n8n-nodes-base.gmail` (operation: `markAsRead`)
- **Key configuration:**
  - `messageId`: `={{ $('Gmail Trigger').all()[0].json.id }}`
- **Input / output:**
  - Input: from either `Update Invoice parameters` or `Update text Invoice parameters`
  - Output: none
- **Credentials:** Gmail OAuth2
- **Edge cases / failures:**
  - If multiple items are processed for a single email (multiple attachments), this may run multiple times (usually harmless).
  - If the trigger ID differs from the processed message ID in multi-item flows, it may mark the wrong message (rare here but possible if batching/parallelization is introduced).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | gmailTrigger | Poll Gmail for new emails | — | Get a message | ## Gmail Trigger based on user customized interval<br>**then get the gmail parameter inclusive text and attachments and extract all pdf files from attachments with js-code** |
| Get a message | gmail | Fetch full email + attachments | Gmail Trigger | Code in JavaScript | ## Gmail Trigger based on user customized interval<br>**then get the gmail parameter inclusive text and attachments and extract all pdf files from attachments with js-code** |
| Code in JavaScript | code | Split attachments into items; normalize binary key to `data` | Get a message | Extract from File; Merge | ## Gmail Trigger based on user customized interval<br>**then get the gmail parameter inclusive text and attachments and extract all pdf files from attachments with js-code** |
| Extract from File | extractFromFile | Extract text from PDF attachments | Code in JavaScript | Invoice Recognition Agent | ## Extract the text from the pdf attachment if exists<br>**otherwise ignore and further process** |
| OpenAI Chat Model | lmChatOpenAi | LLM for invoice recognition | — | Invoice Recognition Agent (AI) | ## Analyse the context of the mail and the attachments and process further in case it is an invoice<br>**otherwise ignore and stop the process** |
| Structured Output Parser | outputParserStructured | Parse recognition result into `{email_contains_invoice:boolean}` | — | Invoice Recognition Agent (AI) | ## Analyse the context of the mail and the attachments and process further in case it is an invoice<br>**otherwise ignore and stop the process** |
| Invoice Recognition Agent | agent | Classify whether email contains invoice | Extract from File | Check wether it is an invoice | ## Analyse the context of the mail and the attachments and process further in case it is an invoice<br>**otherwise ignore and stop the process** |
| Check wether it is an invoice | if | Gate: proceed only if invoice detected | Invoice Recognition Agent | Invoice Data extractor (true); No Operation, do nothing (false) | ## Analyse the context of the mail and the attachments and process further in case it is an invoice<br>**otherwise ignore and stop the process** |
| No Operation, do nothing | noOp | Stop path for non-invoices | Check wether it is an invoice (false) | — | ## Analyse the context of the mail and the attachments and process further in case it is an invoice<br>**otherwise ignore and stop the process** |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for invoice field extraction | — | Invoice Data extractor (AI) | ## Extract the relevant invoice parameters<br>**date_email, date_invoice, invoice_nr, description, provider, net_amount, vat, gross_amount, label, currency** |
| OpenAI Chat Model2 | lmChatOpenAi | LLM helper for parser auto-fix | — | Structured Output Parser1 (AI) | ## Extract the relevant invoice parameters<br>**date_email, date_invoice, invoice_nr, description, provider, net_amount, vat, gross_amount, label, currency** |
| Structured Output Parser1 | outputParserStructured | Parse extracted invoice fields (strict schema, autoFix) | — | Invoice Data extractor (AI) | ## Extract the relevant invoice parameters<br>**date_email, date_invoice, invoice_nr, description, provider, net_amount, vat, gross_amount, label, currency** |
| Invoice Data extractor | agent | Extract normalized invoice fields from text/PDF | Check wether it is an invoice (true) | Check wether it was a file-invoice or text invoice | ## Extract the relevant invoice parameters<br>**date_email, date_invoice, invoice_nr, description, provider, net_amount, vat, gross_amount, label, currency** |
| Check wether it was a file-invoice or text invoice | if | Branch: attachment-based vs body-text invoice | Invoice Data extractor | Document the invoice parameters (true); Update text Invoice parameters (false) | ## Was the invoice in attachments or in the mail body? |
| Document the invoice parameters | googleSheets | Append invoice fields (pre-upload) | Check wether it was a file-invoice or text invoice (true) | Merge | ## Document the paramters in sheet, uoload the invoice file to drive and update the sheet with the invoice-url in your drive, and finally mark the invoice message as read in you gmail |
| Merge | merge | Combine invoice fields + attachment item by position | Document the invoice parameters; Code in JavaScript | Upload file | ## Document the paramters in sheet, uoload the invoice file to drive and update the sheet with the invoice-url in your drive, and finally mark the invoice message as read in you gmail |
| Upload file | googleDrive | Upload invoice file to Drive | Merge | Update Invoice parameters | ## Document the paramters in sheet, uoload the invoice file to drive and update the sheet with the invoice-url in your drive, and finally mark the invoice message as read in you gmail |
| Update Invoice parameters | googleSheets | Append/update row with Drive link | Upload file | Mark a message as read | ## Document the paramters in sheet, uoload the invoice file to drive and update the sheet with the invoice-url in your drive, and finally mark the invoice message as read in you gmail |
| Update text Invoice parameters | googleSheets | Append/update row with Gmail link (no attachment) | Check wether it was a file-invoice or text invoice (false) | Mark a message as read | ## Document the paramters in sheet, generate a link to the email |
| Mark a message as read | gmail | Mark processed email as read | Update Invoice parameters; Update text Invoice parameters | — | ## Document the paramters in sheet, uoload the invoice file to drive and update the sheet with the invoice-url in your drive, and finally mark the invoice message as read in you gmail |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1) **Gmail OAuth2** credential in n8n (scopes to read messages, modify/mark read, and download attachments).  
   2) **Google Drive OAuth2** credential (upload files into target folder).  
   3) **Google Sheets OAuth2** credential (read/write spreadsheet).  
   4) **OpenAI** credential (API key) for LangChain OpenAI nodes.

2. **Create the Google Sheet**
   - Spreadsheet: e.g., `invoices`
   - Sheet/tab: first tab (gid=0)
   - Header columns (exactly):  
     `date_email, date_invoice, invoice_nr, description, provider, net_amount, vat, gross_amount, label, currency, link`

3. **Create/select the Google Drive folder**
   - Folder name e.g. `invoices`
   - Copy the **Folder ID** for use in the Drive node.

4. **Add node: “Gmail Trigger”**
   - Node type: **Gmail Trigger**
   - Polling interval: **Every hour**
   - (Recommended) Add filters (label/query) to limit to invoice-related senders.

5. **Add node: “Get a message” (Gmail)**
   - Operation: **Get**
   - Message ID: `={{ $json.id }}`
   - Options:
     - Download attachments: **true**
     - Attachment property prefix: `attachment_`
     - Simple: **false**
   - Connect: `Gmail Trigger → Get a message`

6. **Add node: “Code in JavaScript”**
   - Node type: **Code**
   - Paste logic to split attachments into items and rename binary to `data` (as in workflow):
     - Iterate `item.binary` keys starting with `attachment_`
     - Output items each with `binary.data = bin[k]`
   - Set: **Always output data** = true
   - Connect: `Get a message → Code in JavaScript`

7. **Add node: “Extract from File”**
   - Node type: **Extract from File**
   - Operation: **PDF**
   - Error handling: set to **Continue (regular output)** so failures don’t stop.
   - Connect: `Code in JavaScript → Extract from File`

8. **Add AI nodes for invoice recognition**
   1) Node: **OpenAI Chat Model**  
      - Model: `gpt-5.1`
   2) Node: **Structured Output Parser**  
      - Manual schema with required boolean `email_contains_invoice`
   3) Node: **Invoice Recognition Agent**  
      - Prompt: classifier instructions; include variables:
        - `$('Get a message').all()[0].json.subject`
        - `$('Get a message').all()[0].json.text`
        - `$('Get a message').all()[0].json.headers.from`
        - `$('Get a message').all()[0].json.headers.date`
        - `{{ $json.text }}` for extracted attachment text
      - Enable “Has Output Parser”.
      - Connect **AI Language Model**: `OpenAI Chat Model → Invoice Recognition Agent`
      - Connect **AI Output Parser**: `Structured Output Parser → Invoice Recognition Agent`
   4) Connect: `Extract from File → Invoice Recognition Agent`

9. **Add node: “Check wether it is an invoice” (IF)**
   - Condition: boolean is true
   - Left value: `={{ $json.output.email_contains_invoice }}`
   - True branch continues; false branch stops.
   - Connect: `Invoice Recognition Agent → IF`

10. **Add node: “No Operation, do nothing”**
   - Node type: **NoOp**
   - Connect: `IF (false) → NoOp`

11. **Add AI nodes for invoice data extraction**
   1) Node: **OpenAI Chat Model1** (model `gpt-5.1`)
   2) Node: **OpenAI Chat Model2** (model `gpt-5.1`) — used for parser auto-fix
   3) Node: **Structured Output Parser1**
      - Manual schema with required fields:
        `date_email, date_invoice, invoice_nr, description, provider, net_amount, vat, gross_amount, label, currency`
      - Enable **autoFix**
      - Connect **AI Language Model**: `OpenAI Chat Model2 → Structured Output Parser1`
   4) Node: **Invoice Data extractor** (Agent)
      - Prompt references:
        - `$('Extract from File').item.json.text`
        - `$('Get a message').all()[0].json.text`
        - `$('Get a message').all()[0].json.headers.date`
      - Enable “Has Output Parser”
      - Connect **AI Language Model**: `OpenAI Chat Model1 → Invoice Data extractor`
      - Connect **AI Output Parser**: `Structured Output Parser1 → Invoice Data extractor`
   5) Connect: `IF (true) → Invoice Data extractor`

12. **Add node: “Check wether it was a file-invoice or text invoice” (IF)**
   - Condition: object exists
   - Left value: `={{ $('Get a message').all()[0].binary }}`
   - True: attachment path; False: text-only path.
   - Connect: `Invoice Data extractor → IF`

13. **Attachment path (Drive + Sheets update)**
   1) Node: **Document the invoice parameters** (Google Sheets)
      - Operation: **Append**
      - Document: select your spreadsheet
      - Sheet: `gid=0`
      - Map columns from AI output: `={{ $json.output.<field> }}`
      - Leave `link` empty at this stage (or set later)
      - Connect: `IF (true) → Document the invoice parameters`
   2) Node: **Merge**
      - Mode: **Combine**
      - Combine by: **Position**
      - Connect inputs:
        - `Document the invoice parameters → Merge (Input 1)`
        - `Code in JavaScript → Merge (Input 2)` (use the node’s second output index if needed)
   3) Node: **Upload file** (Google Drive)
      - Operation: upload (default in Drive node)
      - Drive: My Drive
      - Folder: your invoices folder ID
      - Binary property: `data`
      - File name: `Invoice_<provider>_<date_invoice>` (use a robust expression sourced from the merged item)
      - Connect: `Merge → Upload file`
   4) Node: **Update Invoice parameters** (Google Sheets)
      - Operation: **Append or Update**
      - Matching column: `invoice_nr`
      - Set `link` to: `https://drive.google.com/file/d/{{ $json.id }}/view`
      - Map other fields from the merged data
      - Connect: `Upload file → Update Invoice parameters`

14. **Text-only path (Sheets update with Gmail link)**
   - Node: **Update text Invoice parameters** (Google Sheets)
     - Operation: **Append or Update**
     - Matching column: `invoice_nr`
     - Link: `https://mail.google.com/mail/u/0/#inbox/<messageId>` using `$('Get a message').first().json.id`
     - Map fields from `={{ $json.output.<field> }}`
   - Connect: `IF (false) → Update text Invoice parameters`

15. **Add node: “Mark a message as read”**
   - Node type: **Gmail**
   - Operation: **Mark as read**
   - Message ID: `={{ $('Gmail Trigger').all()[0].json.id }}`
   - Connect:
     - `Update Invoice parameters → Mark a message as read`
     - `Update text Invoice parameters → Mark a message as read`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This template is for founders, finance teams, and solo operators…” plus setup/customization notes (filters, categories, add notifications, extend data model). | Provided in the large sticky note describing intended audience, behavior, requirements, and customization guidance. |
| Tags: `sme`, `invoice`, `ai`, `finance` | Workflow metadata tags in n8n. |
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | User-provided disclaimer (French) asserting legality and policy compliance. |