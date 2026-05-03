Extract invoice data from Gmail to Airtable with Mistral OCR and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-gmail-to-airtable-with-mistral-ocr-and-gpt-4-1-mini-13469


# Extract invoice data from Gmail to Airtable with Mistral OCR and GPT-4.1-mini

## 1. Workflow Overview

**Purpose:** This workflow monitors Gmail for emails that include attachments and appear to be invoices, uploads the attachment to an image-hosting service to obtain a public URL, runs OCR (Mistral) to extract invoice text, uses an LLM (GPT-4.1-mini via LangChain nodes) to convert the text into structured invoice fields, validates the extracted data, and then stores the result in Airtable. It also prevents duplicate records and sends an error email if validation fails.

**Target use cases:**
- Automated invoice intake from email
- OCR-based extraction from invoice images
- Structured data capture into Airtable for tracking and reporting
- Basic duplicate prevention and human-in-the-loop error notification

### 1.1 Input Reception (Gmail)
Receives incoming emails from Gmail that have attachments.

### 1.2 Filter Logic (Invoice gating)
Ensures the email contains binary attachment data and the subject contains the word “invoice”.

### 1.3 Image Hosting (Public URL generation)
Uploads the attachment to imgbb to obtain a public image URL.

### 1.4 Duplicate Protection (Airtable lookup)
Checks Airtable for an existing record that matches a derived “invoice number” key (based on the hosted image name).

### 1.5 OCR Extraction (Mistral)
Downloads the hosted image and sends it to Mistral OCR to extract text/markdown.

### 1.6 AI Data Structuring (LLM + structured parser)
Uses GPT-4.1-mini with a Structured Output Parser to transform OCR text into a JSON invoice schema.

### 1.7 Validation + Storage (Airtable create)
Validates required fields and creates a new Airtable record when data is acceptable.

### 1.8 Error Handling (Gmail notification)
On validation failure, compiles error context and emails a human operator.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Gmail)
**Overview:** Polls Gmail on a schedule and fetches emails matching a search query, downloading attachments into binary fields for downstream processing.  
**Nodes involved:** `Gmail Trigger`

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — entry point trigger (polling).
- **Key configuration (interpreted):**
  - Query filter: `has:attachment`
  - Polling: every minute
  - Downloads attachments: enabled
  - “Simple” mode: disabled (returns richer email structure)
- **Inputs / outputs:**
  - **Output →** `If`
  - Output includes `$json.subject`, `$json.to`, `$json.from`, and `$binary.attachment_0` (and potentially more attachments).
- **Credentials:** Gmail OAuth2 (`Gmail account 2`)
- **Edge cases / failures:**
  - OAuth token expiry / revoked consent
  - Gmail API quota limits
  - Emails with attachments but attachment download failing (binary missing)
  - Multiple attachments: only `attachment_0` is used later (others ignored)
- **Version notes:** TypeVersion `1.3` (behavior may differ from earlier trigger formats).

---

### Block 2 — Filter Logic (Invoice gating)
**Overview:** Filters incoming messages to only those that have binary data and contain “invoice” in the subject. Non-matching messages are discarded via a no-op.  
**Nodes involved:** `If`, `No Operation, do nothing`

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — conditional routing.
- **Key configuration:**
  - Condition 1 (binary exists): `{{ $binary ? true : false }} == true`
  - Condition 2 (subject contains invoice): `{{ $json.subject.toLowerCase() }}` contains `"invoice"`
  - Combination: AND
- **Inputs / outputs:**
  - **Input ←** `Gmail Trigger`
  - **True output →** `host image`
  - **False output →** `No Operation, do nothing`
- **Edge cases / failures:**
  - If `$json.subject` is missing/null, `.toLowerCase()` can error (expression failure). Consider guarding: `{{$json.subject?.toLowerCase() || ''}}`.
  - Emails with invoice attachment but subject not containing “invoice” will be skipped.

#### Node: No Operation, do nothing
- **Type / role:** `n8n-nodes-base.noOp` — sink for filtered-out messages.
- **Inputs / outputs:** Receives false branch from `If`; no further outputs.
- **Edge cases:** None (intentionally does nothing).

---

### Block 3 — Image hosting (Public URL generation)
**Overview:** Uploads the invoice attachment to imgbb so OCR can access it via a public URL (and uses returned metadata as an invoice key for duplicate checking).  
**Nodes involved:** `host image`

#### Node: host image
- **Type / role:** `n8n-nodes-base.httpRequest` — uploads file via multipart/form-data.
- **Key configuration:**
  - Method: `POST`
  - URL: `https://api.imgbb.com/1/upload`
  - Body: multipart form-data
  - Form field `image`: binary file from input field `attachment_0`
  - Auth: “genericCredentialType” with `httpQueryAuth` (imgbb commonly uses an API key as a query parameter like `key=...`)
- **Inputs / outputs:**
  - **Input ←** `If` (true branch)
  - **Output →** `Search Existing Invoices`
  - Output JSON contains: `data.image.url`, `data.image.name`, etc.
- **Credentials:** `httpQueryAuth` (“Query Auth account”) — typically stores the imgbb API key.
- **Edge cases / failures:**
  - Upload fails if the attachment is **PDF** (imgbb may accept images only; your sample shows Gmail attachment is PDF but the pinned “host image” output is PNG, implying conversion happened outside this workflow or a different run).
  - Attachment too large, unsupported type, or rate-limits
  - Missing binary field `attachment_0` (if Gmail returns different binary field names)
  - Public URL exposure/security considerations (invoice data becomes accessible via a public link unless imgbb settings restrict it)

---

### Block 4 — Duplicate Protection (Airtable lookup + switch)
**Overview:** Searches Airtable for an existing invoice record using a derived key and routes execution based on whether a record exists.  
**Nodes involved:** `Search Existing Invoices`, `Check for Duplicates`, `No Operation, do nothing1`

#### Node: Search Existing Invoices
- **Type / role:** `n8n-nodes-base.airtable` — search operation.
- **Key configuration:**
  - Base: “Invoices from Gmail” (`appU9XzFqL5Aovj2M`)
  - Table: “Invoice details” (`tblQ9c8ReMH8zfafc`)
  - Operation: Search
  - Filter formula:
    - `={Invoice Number} = "#{{ $json.data.image.name }}"`
    - Uses the **imgbb image name** (e.g., `INV-00001`) and prepends `#` in the filter.
- **Inputs / outputs:**
  - **Input ←** `host image`
  - **Output →** `Check for Duplicates`
  - `alwaysOutputData: true` ensures something is output even when no records found.
- **Credentials:** Airtable Personal Access Token
- **Edge cases / failures:**
  - **Key mismatch risk:** Airtable records store `Invoice Number` as `INV-00001` (no `#` in your pinned created record), but the search uses `#INV-00001`. This likely prevents matches and will allow duplicates. Fix either the stored format or the filter formula.
  - imgbb name might not equal invoice number (it equals the uploaded filename base); if filename differs from actual invoice number, duplicates won’t be detected.
  - Airtable rate limits / permissions / wrong base/table IDs.

#### Node: Check for Duplicates
- **Type / role:** `n8n-nodes-base.switch` — branching based on expression result.
- **Key configuration:**
  - Mode: expression
  - Outputs: 2
  - Expression: `{{ $items("Search Existing Invoices")[0].json.id ? 1 : 0 }}`
    - If an Airtable record id exists → output index **1**
    - Else → output index **0**
- **Inputs / outputs:**
  - **Input ←** `Search Existing Invoices`
  - **Output 0 (no duplicate found) →** `Download Invoice`
  - **Output 1 (duplicate found) →** `No Operation, do nothing1`
- **Edge cases / failures:**
  - If `Search Existing Invoices` returns an empty item without `.json.id`, expression is fine (falls back to 0).
  - If search returns multiple items, only the first is checked.

#### Node: No Operation, do nothing1
- **Type / role:** `n8n-nodes-base.noOp` — sink for duplicates.
- **Inputs / outputs:** Receives duplicate branch; no outputs.

---

### Block 5 — OCR Extraction (Download + Mistral OCR)
**Overview:** Downloads the hosted invoice image and sends it to Mistral OCR configured for `image_url`-based document processing.  
**Nodes involved:** `Download Invoice`, `Extract text`

#### Node: Download Invoice
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches the image content.
- **Key configuration:**
  - URL: `{{ $('host image').item.json.data.image.url }}`
  - Default method: GET
  - Produces binary output (shown as `binary.data` in pinned execution)
- **Inputs / outputs:**
  - **Input ←** `Check for Duplicates` (no-duplicate branch)
  - **Output →** `Extract text`
- **Edge cases / failures:**
  - If imgbb link expired/blocked
  - If URL points to a non-image or access denied
  - Large image may cause timeouts/memory pressure depending on n8n settings

#### Node: Extract text
- **Type / role:** `n8n-nodes-base.mistralAi` — OCR extraction.
- **Key configuration:**
  - Document type: `image_url`
  - (Implicitly uses Mistral OCR model; pinned output shows `mistral-ocr-latest`)
- **Inputs / outputs:**
  - **Input ←** `Download Invoice`
  - **Output →** `Analyze Invoices`
  - Output includes: `pages[0].markdown`, `extractedText`, `pageCount`, etc.
- **Credentials:** Mistral Cloud API key
- **Edge cases / failures:**
  - If the node truly expects a URL but receives binary (configuration mismatch). Your current setup works in practice because the node output is pinned, but confirm the node’s expected input mapping in your n8n version.
  - OCR errors on low-quality scans, rotated pages, multi-page documents
  - Non-English layouts/currencies may degrade extraction

---

### Block 6 — AI Data Structuring (LLM + structured output)
**Overview:** Feeds OCR markdown to a LangChain Agent backed by GPT-4.1-mini and enforces a JSON schema using the Structured Output Parser.  
**Nodes involved:** `Analyze Invoices`, `OpenAI Chat Model`, `Structured Output Parser`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat LLM to the agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Options: default
- **Inputs / outputs:**
  - Connected via **AI** port:
    - **Output (ai_languageModel) →** `Analyze Invoices` (as its language model provider)
- **Credentials:** OpenAI API credential
- **Edge cases / failures:**
  - Model availability / API quota
  - Content too long (token limits) if OCR output is large

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — constrains LLM output to a JSON structure.
- **Key configuration:**
  - JSON schema example includes:
    - `invoiceNumber`, `from`, `to`, `invoiceDate`, `dueDate`, `description`, `amount.value`, `amount.currency`
  - Note: It uses an **example-driven schema**; strictness depends on node/version.
- **Inputs / outputs:**
  - Connected via **AI** port:
    - **Output (ai_outputParser) →** `Analyze Invoices`
- **Edge cases / failures:**
  - LLM may still produce invalid JSON in some edge cases; parser can fail
  - Date formatting inconsistency (example shows `YYYY-MM-DD`, but extracted output may be `DD.MM.YYYY` as in pinned data)

#### Node: Analyze Invoices
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model + parser to produce structured fields.
- **Key configuration:**
  - Prompt type: “define”
  - Prompt text:
    - `You are an expert in analyzing invoices. Here is an invoice that you should analyze: {{ $json.pages[0].markdown }}`
  - `hasOutputParser`: true (uses Structured Output Parser)
- **Inputs / outputs:**
  - **Input ←** `Extract text`
  - **Output →** `Validate Data`
  - Output JSON: `$json.output` containing the structured invoice fields
- **Edge cases / failures:**
  - If OCR output has no `pages[0].markdown` (empty OCR, different structure), expression fails or prompt becomes empty
  - Hallucinated values if OCR is incomplete
  - Multi-page invoices: only page 0 is used

---

### Block 7 — Validation + Storage (Airtable create)
**Overview:** Validates essential fields and amount > 0; if valid, creates an Airtable record including a link to the hosted invoice image.  
**Nodes involved:** `Validate Data`, `upload invoice & details`

#### Node: Validate Data
- **Type / role:** `n8n-nodes-base.if` — data quality gate.
- **Key configuration (AND conditions):**
  - `output.invoiceNumber` not empty
  - `output.amount.value` > 0
  - `output.invoiceDate` not empty
  - `output.from` not empty
  - `output.to` not empty
- **Inputs / outputs:**
  - **Input ←** `Analyze Invoices`
  - **True output →** `upload invoice & details`
  - **False output →** `Invalid Data Handler`
- **Edge cases / failures:**
  - If `output.amount.value` is a string (e.g., `"200.00"`), numeric comparison may fail under strict validation
  - Missing `output` object causes expression failures
  - No validation of currency, dueDate format, or date format correctness

#### Node: upload invoice & details
- **Type / role:** `n8n-nodes-base.airtable` — create record.
- **Key configuration:**
  - Base/Table: same as search
  - Operation: Create
  - Mapped fields (highlights):
    - `To` = `{{$json.output.to}}`
    - `From` = `{{$json.output.from}}`
    - `Due Date` = `{{$json.output.dueDate}}`
    - `Invoice Date` = `{{$json.output.invoiceDate}}`
    - `Invoice Number` = `{{$json.output.invoiceNumber}}`
    - `Short Discription` = `{{$json.output.description}}` (note spelling)
    - `Client Name` = `{{$('Gmail Trigger').item.json.to.value[0].name}}`
    - `Client Email Address` = `{{$('Gmail Trigger').item.json.to.value[0].address}}`
    - `Amount (USD)` = `{{$json.output.amount.value }}{{ $json.output.amount.currency }}`
      - (This concatenates without a space, e.g. `200USD`)
    - `Link to Invoice` = `[{ url: $('host image').item.json.data.image.url }]`
      - Airtable attachment field format (URL-based attachment)
- **Inputs / outputs:**
  - **Input ←** `Validate Data` (true branch)
  - Output is the created record metadata.
- **Credentials:** Airtable Personal Access Token
- **Edge cases / failures:**
  - Airtable attachment creation from a third-party URL can fail if the URL is not publicly accessible
  - Field type mismatches (dates stored as strings here; if Airtable expects date types, conversion may be needed)
  - If Gmail “To” is not the client (often invoices are **from** vendor to you); using `to` may store your own name/email instead of vendor/client

---

### Block 8 — Error Handling (validation failure email)
**Overview:** When extracted data fails validation, the workflow compiles context and sends an email notification with extracted payload for debugging.  
**Nodes involved:** `Invalid Data Handler`, `Send Error Email`

#### Node: Invalid Data Handler
- **Type / role:** `n8n-nodes-base.set` — prepares an error object.
- **Key configuration:**
  - Sets:
    - `error` = `Data validation failed`
    - `step` = `Validate Data`
    - `invoiceSubject` = `{{$('Gmail Trigger').item.json.subject}}`
    - `timestamp` = `{{$now}}`
    - `extractedData` = `{{ JSON.stringify($('Analyze Invoices').item.json.output) }}`
- **Inputs / outputs:**
  - **Input ←** `Validate Data` (false branch)
  - **Output →** `Send Error Email`
- **Edge cases / failures:**
  - If `Analyze Invoices` produced no output, `JSON.stringify(...)` may stringify `undefined` (becoming undefined/empty depending on n8n behavior)

#### Node: Send Error Email
- **Type / role:** `n8n-nodes-base.gmail` — sends notification email.
- **Key configuration:**
  - To: `user@example.com`
  - Subject: `⚠️ Invoice Processing Error - {{ $json.step }}`
  - Body includes details and additional data:
    - `{{ $json.extractedData || $json.extractedText || 'No additional data' }}`
- **Inputs / outputs:**
  - **Input ←** `Invalid Data Handler`
  - Terminal node (no further connections)
- **Credentials:** Gmail OAuth2 (`Gmail account 2`)
- **Edge cases / failures:**
  - Gmail send permission issues / OAuth scope problems
  - Invalid recipient address
  - Missing fields like `$json.errorMessage` (template references it, but it is never set in this workflow)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for emails with attachments | — | If | ## Trigger |
| If | n8n-nodes-base.if | Filter: has binary + subject contains “invoice” | Gmail Trigger | host image; No Operation, do nothing | ## Filter Logic |
| No Operation, do nothing | n8n-nodes-base.noOp | Discard non-invoice emails | If (false) | — | ## Filter Logic |
| host image | n8n-nodes-base.httpRequest | Upload attachment to imgbb to get public URL | If (true) | Search Existing Invoices | ## Image hosting |
| Search Existing Invoices | n8n-nodes-base.airtable | Search Airtable for existing invoice | host image | Check for Duplicates | ## Duplicate Protection |
| Check for Duplicates | n8n-nodes-base.switch | Route based on whether a record exists | Search Existing Invoices | Download Invoice; No Operation, do nothing1 | ## Duplicate Protection |
| No Operation, do nothing1 | n8n-nodes-base.noOp | Discard duplicates | Check for Duplicates (dup branch) | — | ## Duplicate Protection |
| Download Invoice | n8n-nodes-base.httpRequest | Download hosted invoice image | Check for Duplicates (no-dup branch) | Extract text | ## OCR Extraction |
| Extract text | n8n-nodes-base.mistralAi | OCR the invoice image | Download Invoice | Analyze Invoices | ## OCR Extraction |
| Analyze Invoices | @n8n/n8n-nodes-langchain.agent | Convert OCR text to structured invoice JSON | Extract text + (AI ports) | Validate Data | ## AI Data Structuring |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for the agent | — (AI-only) | Analyze Invoices (ai_languageModel) | ## AI Data Structuring |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON output | — (AI-only) | Analyze Invoices (ai_outputParser) | ## AI Data Structuring |
| Validate Data | n8n-nodes-base.if | Validate extracted fields before storage | Analyze Invoices | upload invoice & details; Invalid Data Handler | ## Store Invoice |
| upload invoice & details | n8n-nodes-base.airtable | Create Airtable record | Validate Data (true) | — | ## Store Invoice |
| Invalid Data Handler | n8n-nodes-base.set | Build error payload for notification | Validate Data (false) | Send Error Email | ## Error Handling |
| Send Error Email | n8n-nodes-base.gmail | Email notification on failure | Invalid Data Handler | — | ## Error Handling |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace note (setup guide) | — | — | # 🛠️ Setup Guide; Author: [Safa Khan](); Airtable base link: https://airtable.com/appU9XzFqL5Aovj2M/shrnanmtZpUgbVsEo; imgbb: https://api.imgbb.com/; Mistral keys: https://admin.mistral.ai/organization/api-keys |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## Trigger |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## Filter Logic |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## Image hosting |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## Duplicate Protection |
| Sticky Note5 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## OCR Extraction |
| Sticky Note6 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## AI Data Structuring |
| Sticky Note7 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## Store Invoice |
| Sticky Note8 | n8n-nodes-base.stickyNote | Workspace note label | — | — | ## Error Handling |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger**
   1. Add node: **Gmail Trigger**
   2. Configure:
      - Search query (`q`): `has:attachment`
      - Download attachments: **enabled**
      - Polling: **Every minute**
   3. Connect Gmail OAuth2 credentials (Google Cloud OAuth client or n8n cloud auth), ensure scopes allow reading messages and attachments.

2) **Add invoice filter**
   1. Add node: **If**
   2. Create two AND conditions:
      - Boolean equals: left `{{ $binary ? true : false }}` right `true`
      - String contains: left `{{ $json.subject.toLowerCase() }}` right `invoice`
   3. Add node: **NoOp** (name it “No Operation, do nothing”) and connect it to the **false** output of the If.
   4. Connect **Gmail Trigger → If**.

3) **Upload attachment to imgbb**
   1. Add node: **HTTP Request** (name: “host image”)
   2. Configure:
      - Method: `POST`
      - URL: `https://api.imgbb.com/1/upload`
      - Content-Type: `multipart-form-data`
      - Send Body: enabled
      - Body parameter:
        - Name: `image`
        - Type: **Binary Data**
        - Input field: `attachment_0`
      - Authentication: query auth (API key in query), via **HTTP Query Auth** credential
   3. Connect **If (true) → host image**.
   4. Create/attach credential containing the imgbb API key (commonly `key=<YOUR_KEY>`).

4) **Search Airtable to prevent duplicates**
   1. Add node: **Airtable** (name: “Search Existing Invoices”)
   2. Configure:
      - Operation: **Search**
      - Base: your base (example: `appU9XzFqL5Aovj2M`)
      - Table: your table (example: `Invoice details`)
      - Filter formula: `={Invoice Number} = "#{{ $json.data.image.name }}"`
   3. Enable “Always Output Data” (so downstream nodes still run even if empty).
   4. Connect Airtable Personal Access Token credentials.
   5. Connect **host image → Search Existing Invoices**.

5) **Branch based on duplicates**
   1. Add node: **Switch** (name: “Check for Duplicates”)
   2. Configure:
      - Mode: **Expression**
      - Number of outputs: 2
      - Output expression: `{{ $items("Search Existing Invoices")[0].json.id ? 1 : 0 }}`
   3. Add node: **NoOp** (name: “No Operation, do nothing1”) and connect it to output **1** (duplicate found).
   4. Connect **Search Existing Invoices → Check for Duplicates**.

6) **Download the hosted invoice image**
   1. Add node: **HTTP Request** (name: “Download Invoice”)
   2. Configure:
      - URL: `{{ $('host image').item.json.data.image.url }}`
      - Ensure it downloads the file as binary (default behavior varies; confirm “Response Format”/“Download” settings depending on your n8n version).
   3. Connect **Check for Duplicates (output 0) → Download Invoice**.

7) **OCR with Mistral**
   1. Add node: **Mistral AI** (name: “Extract text”)
   2. Configure:
      - Feature: OCR / document extraction
      - Document type: `image_url`
   3. Connect Mistral Cloud API credentials (API key from: https://admin.mistral.ai/organization/api-keys).
   4. Connect **Download Invoice → Extract text**.

8) **LLM-based structuring with LangChain nodes**
   1. Add node: **Structured Output Parser**
      - Paste/provide an example JSON schema like the one in the workflow (invoiceNumber, from, to, invoiceDate, dueDate, description, amount.value, amount.currency).
   2. Add node: **OpenAI Chat Model**
      - Select model: `gpt-4.1-mini`
      - Connect OpenAI API credentials.
   3. Add node: **AI Agent** (name: “Analyze Invoices”)
      - Prompt type: “Define”
      - Prompt text:  
        `You are an expert in analyzing invoices. Here is an invoice that you should analyze: {{ $json.pages[0].markdown }}`
      - Enable output parser usage.
   4. Wire the AI connections:
      - **OpenAI Chat Model (ai_languageModel) → Analyze Invoices**
      - **Structured Output Parser (ai_outputParser) → Analyze Invoices**
   5. Wire main flow:
      - **Extract text → Analyze Invoices**

9) **Validate extracted fields**
   1. Add node: **If** (name: “Validate Data”)
   2. Conditions (AND):
      - `{{ $json.output.invoiceNumber }}` not empty
      - `{{ $json.output.amount.value }}` greater than `0`
      - `{{ $json.output.invoiceDate }}` not empty
      - `{{ $json.output.from }}` not empty
      - `{{ $json.output.to }}` not empty
   3. Connect **Analyze Invoices → Validate Data**

10) **Create Airtable record**
   1. Add node: **Airtable** (name: “upload invoice & details”)
   2. Configure:
      - Operation: **Create**
      - Base/Table: same as search
      - Map fields using expressions from `Analyze Invoices`, `Gmail Trigger`, and `host image`:
        - Invoice Number, From, To, Invoice Date, Due Date, Description, Amount, Link to Invoice, Client Name, Client Email
      - For attachment field use: `[{ "url": "<public_url>" }]`
   3. Connect **Validate Data (true) → upload invoice & details**

11) **Handle invalid data + alert**
   1. Add node: **Set** (name: “Invalid Data Handler”)
      - Set fields: error, step, invoiceSubject, timestamp, extractedData
   2. Add node: **Gmail** (name: “Send Error Email”)
      - To: your operational email
      - Subject/body using the workflow’s template
   3. Connect:
      - **Validate Data (false) → Invalid Data Handler → Send Error Email**
   4. Ensure Gmail send credentials are connected and authorized for sending.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Setup Guide** (author credit + checklist) | Included in workflow sticky note |
| Airtable base link | https://airtable.com/appU9XzFqL5Aovj2M/shrnanmtZpUgbVsEo |
| imgbb API | https://api.imgbb.com/ |
| Mistral API keys | https://admin.mistral.ai/organization/api-keys |
| Key design assumption: attachment is an image suitable for imgbb + OCR | If incoming invoices are PDFs, you likely need a PDF→image conversion step before “host image” / OCR |
| Duplicate check relies on hosted image name and a `#` prefix | Verify your Airtable “Invoice Number” format; current search formula may not match created records if they store `INV-00001` (without `#`) |

Disclamer (as provided): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.*