Extract data from PDF reports with Gmail, OCR, Google Sheets and OpenAI GPT-4.1-mini

https://n8nworkflows.xyz/workflows/extract-data-from-pdf-reports-with-gmail--ocr--google-sheets-and-openai-gpt-4-1-mini-12961


# Extract data from PDF reports with Gmail, OCR, Google Sheets and OpenAI GPT-4.1-mini

## 1. Workflow Overview

**Purpose:** Automatically process incoming Gmail emails that contain PDF report attachments, extract text (with OCR support implied), use OpenAI (GPT‑4.1‑mini via n8n LangChain nodes) to parse structured fields and summarize content, classify the document type, then store results in Google Sheets and notify teams in Slack.

**Typical use cases:**
- Automated invoice/contract/report intake from email
- Data extraction from semi-structured PDF documents into spreadsheets
- AI-based classification + routing + notifications

### 1.1 Email Intake & Configuration
Receives new Gmail messages, applies workflow-level defaults (intended), then checks whether a PDF attachment exists.

### 1.2 PDF Text Extraction
If a PDF attachment exists, extracts text from the file for downstream AI processing.

### 1.3 AI Field Extraction (Structured)
Uses an AI Agent with an OpenAI chat model plus a structured output parser to extract key fields from the PDF text, then normalizes the data.

### 1.4 Classification & Routing
Switches on the normalized output to route the item as invoice vs report vs contract (implied by downstream actions).

### 1.5 Storage & Notifications
Writes invoice rows and/or general extracted data to Google Sheets and posts Slack messages, including an AI-generated summary for reports.

### 1.6 Error Path
If no PDF attachment is detected, the workflow routes to a code-based error handler/logger.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Intake & Configuration
**Overview:** Triggers on new Gmail emails, then sets up configuration fields (currently empty in JSON) and checks for presence of a PDF attachment.  
**Nodes Involved:** `New PDF Email Trigger`, `Workflow Configuration`, `Check PDF Attachment Exists`

#### Node: New PDF Email Trigger
- **Type / role:** Gmail Trigger (`n8n-nodes-base.gmailTrigger`) — entry point that fires when Gmail event criteria are met.
- **Configuration choices (interpreted):** Parameters are empty in the JSON export, which typically means defaults are used or the template expects the user to configure:
  - Gmail account/credential
  - Trigger event (new email, label, query, etc.)
- **Inputs/Outputs:**
  - **Output:** to `Workflow Configuration`
- **Version notes:** typeVersion `1.3`
- **Edge cases / failures:**
  - OAuth credential expired / missing scopes
  - Trigger not filtering correctly (if query/label not configured)
  - Attachment metadata not included depending on trigger configuration

#### Node: Workflow Configuration
- **Type / role:** Set (`n8n-nodes-base.set`) — intended to define constants/fields used later (e.g., sheet IDs, doc types, slack channels).
- **Configuration choices:** Empty parameters in JSON, so it currently does not set any fields.
- **Inputs/Outputs:**
  - **Input:** from `New PDF Email Trigger`
  - **Output:** to `Check PDF Attachment Exists`
- **Version notes:** typeVersion `3.4`
- **Edge cases / failures:**
  - If downstream nodes expect fields set here (e.g., sheet IDs), they may fail with “undefined” expressions.

#### Node: Check PDF Attachment Exists
- **Type / role:** IF (`n8n-nodes-base.if`) — gate to ensure a PDF attachment is present.
- **Configuration choices:** Empty parameters in JSON; in a functional workflow this would typically check something like:
  - `{{$json.attachments}}` exists AND contains an item with `mimeType === 'application/pdf'`
- **Inputs/Outputs:**
  - **True path:** to `Extract Text from PDF`
  - **False path:** to `Error Handler and Logger`
- **Version notes:** typeVersion `2.3`
- **Edge cases / failures:**
  - Trigger payload may represent attachments differently; IF condition must match actual structure.
  - Multiple attachments: must select the correct PDF (first match vs all PDFs).

---

### Block 2 — PDF Text Extraction
**Overview:** Extracts text from the PDF attachment so it can be parsed by AI.  
**Nodes Involved:** `Extract Text from PDF`

#### Node: Extract Text from PDF
- **Type / role:** Extract From File (`n8n-nodes-base.extractFromFile`) — extracts text content from binary file input.
- **Configuration choices:** Empty parameters in JSON; typically requires:
  - Input binary property (often `attachment` or `data`)
  - Operation: “Extract text” / PDF extraction
  - OCR settings (if scanned PDFs) depending on node capabilities and n8n version
- **Inputs/Outputs:**
  - **Input:** from `Check PDF Attachment Exists` (true branch)
  - **Output:** to `Parse Key Fields with AI`
- **Version notes:** typeVersion `1.1`
- **Edge cases / failures:**
  - No binary PDF passed (common if IF gate checks incorrectly)
  - Encrypted PDFs / corrupted files
  - OCR quality issues for scanned documents
  - Large PDFs may cause timeouts or memory pressure

---

### Block 3 — AI Field Extraction (Structured) + Normalization
**Overview:** Uses OpenAI chat model with a structured output parser to extract key fields, then normalizes them into a consistent schema for classification and storage.  
**Nodes Involved:** `Parse Key Fields with AI`, `OpenAI Chat Model`, `Structured Output Parser`, `Normalize Extracted Data`

#### Node: Parse Key Fields with AI
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + parsing to produce structured fields.
- **Configuration choices:** Empty parameters in JSON; in practice should define:
  - System/user instructions (what fields to extract)
  - How to consume the extracted PDF text (e.g., `{{$json.text}}`)
  - Expected output schema (delegated to Structured Output Parser)
- **Key linked resources:**
  - Uses `OpenAI Chat Model` via **ai_languageModel** connection
  - Uses `Structured Output Parser` via **ai_outputParser** connection
- **Inputs/Outputs:**
  - **Input:** from `Extract Text from PDF`
  - **Output:** to `Normalize Extracted Data`
- **Version notes:** typeVersion `3`
- **Edge cases / failures:**
  - Model returning non-JSON or schema-mismatched output (parser failure)
  - Token limits exceeded if PDF text is large (needs chunking/summarization strategy)
  - Hallucinated fields if prompt/schema not strict

#### Node: OpenAI Chat Model
- **Type / role:** OpenAI Chat LLM (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the chat model to the agent.
- **Configuration choices:** Empty parameters in JSON; should specify:
  - Model name (user indicates **GPT‑4.1‑mini** in workflow title)
  - Temperature, max tokens (optional)
  - OpenAI credentials
- **Inputs/Outputs:**
  - Connected to `Parse Key Fields with AI` as **ai_languageModel**
- **Version notes:** typeVersion `1.3`
- **Edge cases / failures:**
  - Invalid API key / org / project configuration
  - Rate limits / quota exhaustion
  - Model name mismatch (if GPT‑4.1‑mini not available in the account/region)

#### Node: Structured Output Parser
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — validates/parses LLM output into a structured object.
- **Configuration choices:** Empty parameters in JSON; normally defines:
  - JSON schema / Zod-like schema for fields (e.g., vendor, invoice_number, total, date, doc_type)
- **Inputs/Outputs:**
  - Connected to `Parse Key Fields with AI` as **ai_outputParser**
- **Version notes:** typeVersion `1.3`
- **Edge cases / failures:**
  - If schema is too strict, minor formatting differences cause failures
  - If schema too loose, downstream classification/storage becomes unreliable

#### Node: Normalize Extracted Data
- **Type / role:** Set (`n8n-nodes-base.set`) — maps/renames/cleans parsed fields (dates, currency, doc type labels).
- **Configuration choices:** Empty parameters in JSON; typically used to:
  - Ensure consistent keys (`documentType`, `invoiceTotal`, etc.)
  - Convert strings to numbers/dates
- **Inputs/Outputs:**
  - **Input:** from `Parse Key Fields with AI`
  - **Output:** to `Classify Document Type`
- **Version notes:** typeVersion `3.4`
- **Edge cases / failures:**
  - Expression errors if expected fields are missing
  - Locale issues (decimal separators, date formats)

---

### Block 4 — Classification & Routing
**Overview:** Routes execution based on extracted/normalized document type to invoice storage, report summarization, or contract-team notification.  
**Nodes Involved:** `Classify Document Type`

#### Node: Classify Document Type
- **Type / role:** Switch (`n8n-nodes-base.switch`) — branches depending on a field (likely `documentType`).
- **Configuration choices:** Empty parameters in JSON; a working version would define:
  - Value to evaluate (e.g., `{{$json.documentType}}`)
  - Cases mapping to outputs (Invoice / Report / Contract)
- **Inputs/Outputs:**
  - **Input:** from `Normalize Extracted Data`
  - **Outputs (3):**
    1. to `Store Invoice Data`
    2. to `Summarize Report with AI`
    3. to `Notify Contract Team`
- **Version notes:** typeVersion `3.4`
- **Edge cases / failures:**
  - Unexpected labels from AI (e.g., “invoice”, “Invoice”, “bill”) causing misrouting
  - Missing classification field sends items to default branch (if configured)

---

### Block 5 — Storage & Notifications (Including Report Summary)
**Overview:** Writes extracted data to Google Sheets, creates an AI summary for report-type documents, and posts Slack notifications.  
**Nodes Involved:** `Store Invoice Data`, `Summarize Report with AI`, `OpenAI Chat Model for Summary`, `Store All Extracted Data`, `Notify Contract Team`, `Send Summary Notification`

#### Node: Store Invoice Data
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — stores invoice-specific fields.
- **Configuration choices:** Empty parameters in JSON; would normally include:
  - Spreadsheet ID, sheet name/range
  - Append vs update operation
  - Column mapping from normalized fields
- **Inputs/Outputs:**
  - **Input:** from `Classify Document Type` (invoice branch)
  - **Output:** to `Store All Extracted Data`
- **Version notes:** typeVersion `4.7`
- **Edge cases / failures:**
  - Google OAuth missing/expired
  - Sheet/tab name mismatch, insufficient permissions
  - Data type mismatch (number/date) depending on mapping mode

#### Node: Summarize Report with AI
- **Type / role:** LangChain Agent — generates a human-readable summary for report documents.
- **Configuration choices:** Empty parameters in JSON; should define:
  - Prompt template using extracted text / key fields
  - Output format (plain text, bullet points, etc.)
- **Inputs/Outputs:**
  - **Input:** from `Classify Document Type` (report branch)
  - **Output:** to `Store All Extracted Data`
  - Uses `OpenAI Chat Model for Summary` via **ai_languageModel**
- **Version notes:** typeVersion `3`
- **Edge cases / failures:**
  - Token limit issues if summarizing full text
  - Summaries may omit critical compliance details unless prompt is constrained

#### Node: OpenAI Chat Model for Summary
- **Type / role:** OpenAI Chat LLM — model backing the summarization agent.
- **Configuration choices:** Empty parameters in JSON; should specify model (likely GPT‑4.1‑mini), temperature, etc.
- **Inputs/Outputs:**
  - Connected to `Summarize Report with AI` as **ai_languageModel**
- **Version notes:** typeVersion `1.3`
- **Edge cases / failures:** same as the other OpenAI model node (auth, quota, rate limiting).

#### Node: Notify Contract Team
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts a message to a Slack channel/user for contract-type documents.
- **Configuration choices:** Empty parameters in JSON; should include:
  - Slack credential (bot token)
  - Channel ID/name
  - Message text (likely includes extracted fields and link to email/PDF)
- **Inputs/Outputs:**
  - **Input:** from `Classify Document Type` (contract branch)
  - **Output:** to `Store All Extracted Data`
- **Version notes:** typeVersion `2.4`
- **Edge cases / failures:**
  - Slack token missing/invalid
  - Posting to private channel without membership
  - Message formatting errors if expressions reference missing fields

#### Node: Store All Extracted Data
- **Type / role:** Google Sheets — centralized logging of all processed documents (any branch).
- **Configuration choices:** Empty parameters in JSON; typically:
  - Append row with normalized fields + classification + timestamps + summary (if available)
- **Inputs/Outputs:**
  - **Inputs:** from `Store Invoice Data`, `Summarize Report with AI`, `Notify Contract Team`
  - **Output:** to `Send Summary Notification`
- **Version notes:** typeVersion `4.7`
- **Edge cases / failures:**
  - If multiple branches feed this node, ensure consistent data shape (some fields may be missing)
  - Concurrency: multiple rows appended simultaneously is usually safe, but consider rate limits

#### Node: Send Summary Notification
- **Type / role:** Slack — posts a final Slack notification after storage.
- **Configuration choices:** Empty parameters in JSON; should include:
  - Channel
  - Message including: doc type, key fields, summary, sheet row link
- **Inputs/Outputs:**
  - **Input:** from `Store All Extracted Data`
- **Version notes:** typeVersion `2.4`
- **Edge cases / failures:**
  - Same Slack auth/channel issues
  - Missing summary for non-report documents unless message handles nulls

---

### Block 6 — Error Handling
**Overview:** Handles the “no PDF attachment” branch (and potentially other checks if extended) by logging or shaping an error payload.  
**Nodes Involved:** `Error Handler and Logger`

#### Node: Error Handler and Logger
- **Type / role:** Code (`n8n-nodes-base.code`) — custom JS to log/format errors.
- **Configuration choices:** Empty parameters in JSON; a real implementation might:
  - Create an error object (`reason: 'No PDF attachment'`)
  - Write to a log sheet, send Slack alert, or stop execution gracefully
- **Inputs/Outputs:**
  - **Input:** from `Check PDF Attachment Exists` (false branch)
  - **Output:** none connected
- **Version notes:** typeVersion `2`
- **Edge cases / failures:**
  - If code references missing fields from trigger payload, it can throw and mark execution failed
  - If you want non-failing runs, the code should avoid throwing

---

### Sticky Notes (Documentation Nodes)
All sticky notes in the provided JSON have **empty content**, so they do not add functional documentation.

Nodes: `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note5`  
- **Type / role:** Sticky Note (`n8n-nodes-base.stickyNote`) — canvas comments only; not executed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New PDF Email Trigger | gmailTrigger | Entry trigger on new Gmail email | — | Workflow Configuration |  |
| Workflow Configuration | set | Define constants / normalize trigger payload (currently empty) | New PDF Email Trigger | Check PDF Attachment Exists |  |
| Check PDF Attachment Exists | if | Gate: ensure PDF attachment exists | Workflow Configuration | Extract Text from PDF; Error Handler and Logger |  |
| Extract Text from PDF | extractFromFile | Extract text from PDF binary (OCR/text) | Check PDF Attachment Exists | Parse Key Fields with AI |  |
| Parse Key Fields with AI | langchain.agent | AI extraction of key fields into structured data | Extract Text from PDF | Normalize Extracted Data |  |
| OpenAI Chat Model | lmChatOpenAi | LLM for key-field extraction agent | — (AI connection) | Parse Key Fields with AI (ai_languageModel) |  |
| Structured Output Parser | outputParserStructured | Enforce schema / parse AI output | — (AI connection) | Parse Key Fields with AI (ai_outputParser) |  |
| Normalize Extracted Data | set | Clean/rename fields for consistent downstream use | Parse Key Fields with AI | Classify Document Type |  |
| Classify Document Type | switch | Route by doc type (invoice/report/contract) | Normalize Extracted Data | Store Invoice Data; Summarize Report with AI; Notify Contract Team |  |
| Store Invoice Data | googleSheets | Store invoice-specific row(s) | Classify Document Type | Store All Extracted Data |  |
| Summarize Report with AI | langchain.agent | Produce report summary text | Classify Document Type | Store All Extracted Data |  |
| OpenAI Chat Model for Summary | lmChatOpenAi | LLM for summarization agent | — (AI connection) | Summarize Report with AI (ai_languageModel) |  |
| Notify Contract Team | slack | Alert contract team in Slack | Classify Document Type | Store All Extracted Data |  |
| Store All Extracted Data | googleSheets | Central storage/log for all processed documents | Store Invoice Data; Summarize Report with AI; Notify Contract Team | Send Summary Notification |  |
| Send Summary Notification | slack | Final Slack message after storage | Store All Extracted Data | — |  |
| Error Handler and Logger | code | Handle “no PDF” (and other) errors | Check PDF Attachment Exists | — |  |
| Sticky Note | stickyNote | Canvas comment (empty) | — | — |  |
| Sticky Note1 | stickyNote | Canvas comment (empty) | — | — |  |
| Sticky Note2 | stickyNote | Canvas comment (empty) | — | — |  |
| Sticky Note3 | stickyNote | Canvas comment (empty) | — | — |  |
| Sticky Note4 | stickyNote | Canvas comment (empty) | — | — |  |
| Sticky Note5 | stickyNote | Canvas comment (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger**
   - Add node: **Gmail Trigger** named `New PDF Email Trigger`
   - Configure Gmail OAuth2 credentials
   - Configure trigger event and filtering (recommended):
     - Query/label to only match emails likely to contain PDFs (e.g., has:attachment filename:pdf)

2. **Add Set node for workflow constants**
   - Add node: **Set** named `Workflow Configuration`
   - Add fields you will reuse, for example:
     - `docSource = "gmail"`
     - `invoiceSheetId`, `allDataSheetId`
     - `slackChannelContracts`, `slackChannelSummary`
   - Connect: `New PDF Email Trigger` → `Workflow Configuration`

3. **Add IF node to verify PDF attachment**
   - Add node: **IF** named `Check PDF Attachment Exists`
   - Condition (example—must match your Gmail payload structure):
     - Check that attachments exist and at least one has PDF mime type
   - Connect: `Workflow Configuration` → `Check PDF Attachment Exists`

4. **Add Extract From File node**
   - Add node: **Extract From File** named `Extract Text from PDF`
   - Configure it to read the **binary attachment property** that contains the PDF
   - Enable OCR/extraction mode appropriate for PDFs (as available in your n8n version)
   - Connect IF **true** output → `Extract Text from PDF`

5. **Add AI Agent for structured field parsing**
   - Add node: **AI Agent** (LangChain) named `Parse Key Fields with AI`
   - In the agent instructions, specify:
     - The list of fields to extract (e.g., `documentType`, `date`, `vendor`, `amount`, `invoiceNumber`, etc.)
     - Use the extracted text field from previous node as input
   - Add node: **OpenAI Chat Model** named `OpenAI Chat Model`
     - Set model to **gpt-4.1-mini** (as intended by workflow title)
     - Add OpenAI credentials
   - Add node: **Structured Output Parser** named `Structured Output Parser`
     - Define the schema for the extracted fields
   - Connect:
     - `Extract Text from PDF` → `Parse Key Fields with AI`
     - `OpenAI Chat Model` → `Parse Key Fields with AI` (ai_languageModel connection)
     - `Structured Output Parser` → `Parse Key Fields with AI` (ai_outputParser connection)

6. **Normalize fields**
   - Add node: **Set** named `Normalize Extracted Data`
   - Map AI output to consistent names/types (date parsing, number conversion, doc type normalization)
   - Connect: `Parse Key Fields with AI` → `Normalize Extracted Data`

7. **Classify document type**
   - Add node: **Switch** named `Classify Document Type`
   - Switch on a normalized field, e.g. `{{$json.documentType}}`
   - Create 3 routes/cases (example):
     - Case 1: `invoice` → output 1
     - Case 2: `report` → output 2
     - Case 3: `contract` → output 3
   - Connect: `Normalize Extracted Data` → `Classify Document Type`

8. **Invoice storage (Google Sheets)**
   - Add node: **Google Sheets** named `Store Invoice Data`
   - Configure Google OAuth2 credentials
   - Operation: Append row (typical)
   - Map invoice columns
   - Connect: `Classify Document Type` (invoice output) → `Store Invoice Data`

9. **Report summarization (AI)**
   - Add node: **AI Agent** named `Summarize Report with AI`
   - Provide prompt to create a concise summary from extracted text/fields
   - Add node: **OpenAI Chat Model** named `OpenAI Chat Model for Summary`
     - Model: gpt-4.1-mini (or same as extraction)
   - Connect:
     - `OpenAI Chat Model for Summary` → `Summarize Report with AI` (ai_languageModel)
     - `Classify Document Type` (report output) → `Summarize Report with AI`

10. **Contract notifications (Slack)**
   - Add node: **Slack** named `Notify Contract Team`
   - Configure Slack credentials (bot token)
   - Choose channel and message format
   - Connect: `Classify Document Type` (contract output) → `Notify Contract Team`

11. **Store all extracted data (Google Sheets)**
   - Add node: **Google Sheets** named `Store All Extracted Data`
   - Append a row containing:
     - common fields + classification + (optional) summary + email metadata
   - Connect all three branches into it:
     - `Store Invoice Data` → `Store All Extracted Data`
     - `Summarize Report with AI` → `Store All Extracted Data`
     - `Notify Contract Team` → `Store All Extracted Data`

12. **Final Slack notification**
   - Add node: **Slack** named `Send Summary Notification`
   - Post a “processed” message with key fields and (if present) summary
   - Connect: `Store All Extracted Data` → `Send Summary Notification`

13. **Error handling path**
   - Add node: **Code** named `Error Handler and Logger`
   - Implement logic for missing PDF attachment (log, Slack alert, or simply output an error object)
   - Connect: `Check PDF Attachment Exists` (false output) → `Error Handler and Logger`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| All sticky notes in this workflow are present but contain no text. | Canvas documentation not provided in JSON export. |
| Many nodes have empty parameter blocks in the provided JSON; the workflow as-is is a structural template and requires configuration (credentials, sheet IDs, Slack channels, AI prompts, schemas, and IF/Switch conditions). | Applies to most functional nodes (Gmail/Sheets/Slack/OpenAI/IF/Switch/Set/Agents). |
| Disclaimer (FR): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Provided by the user as compliance context; applies to the overall workflow description. |