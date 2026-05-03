Extract invoice data from Gmail PDFs to JSON, Google Sheets and Airtable

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-gmail-pdfs-to-json--google-sheets-and-airtable-13033


# Extract invoice data from Gmail PDFs to JSON, Google Sheets and Airtable

## 1. Workflow Overview

**Workflow title:** Extract invoice data from Gmail PDFs to JSON, Google Sheets and Airtable

**Purpose:**  
Automatically monitor Gmail for incoming emails containing **PDF invoice attachments**, extract key invoice fields from the PDF text, calculate a **confidence score**, then either:
- **Auto-post** the invoice to a **Google Sheets ledger**, **Airtable**, and **archive the PDF to Google Drive**, or
- **Route to a manual review queue** (Slack + Google Sheets + Gmail alert) if the invoice is **high value** or **low confidence**.

### 1.1 Intake & Validation (Phase 1)
Triggered by Gmail, checks that an attachment exists and that it is a PDF, then enriches the item with metadata and IDs.

### 1.2 PDF Parsing & Field Extraction (Phase 2)
Uses an external “HTML to PDF” service node to parse PDF → JSON/text, then a Code node extracts invoice fields (vendor, totals, dates, etc.) and computes confidence.

### 1.3 Auto-Approval Gate (Phase 3/4 logic split)
An IF node decides whether to auto-approve (green path) or divert to review (amber path) based on `requiresReview`.

### 1.4 Posting, Sync, and Archival (Phase 3)
On approval: append to Google Sheets ledger, create Airtable record, archive PDF in Google Drive, then notify Slack.

### 1.5 Review Queue Escalation (Phase 4)
On review required: Slack alert → append to Review Queue Google Sheet tab → send a Gmail review email.

---

## 2. Block-by-Block Analysis

### Block A — Intake & Validation
**Overview:** Watches Gmail for invoice emails and ensures there’s at least one attachment and that the first attachment is a PDF. Adds standardized metadata used downstream.

**Nodes involved:**
- Gmail: Watch Invoices
- IF: Valid PDF Attachment
- Code: Pre-Processor

#### Node: Gmail: Watch Invoices
- **Type / role:** `gmailTrigger` — entry point that triggers workflow execution from Gmail events.
- **Configuration choices (interpreted):**
  - Uses a webhook-based Gmail trigger (webhook id set).
  - No additional filters are defined in parameters (so filtering is expected to be configured in the trigger UI or handled downstream).
- **Inputs:** None (trigger node).
- **Outputs:** Emits one item per triggering email; includes fields like `attachments`, `subject`, `from`, `date`, `id`.
- **Edge cases / failures:**
  - OAuth permission issues (Gmail OAuth2 scope not granted).
  - Trigger may fire for emails without attachments depending on trigger settings; this is handled by the downstream IF.
  - Gmail payload shape may vary (e.g., `from` missing name/address in some messages).

#### Node: IF: Valid PDF Attachment
- **Type / role:** `if` — validates attachment presence and MIME type.
- **Key conditions:**
  - `attachments` array is **not empty**: `{{ $json.attachments }}`
  - First attachment MIME type equals `application/pdf`: `{{ $json.attachments[0].mimeType }}`
- **Connections:**
  - **True** → Code: Pre-Processor
  - **False** → no connection (items effectively drop)
- **Edge cases / failures:**
  - If the email has multiple attachments and the PDF is not at index 0, this will incorrectly fail.
  - Some invoices come as `application/octet-stream` or other MIME types; will be rejected.
  - If `attachments` exists but is not an array, expression/type validation may fail.

#### Node: Code: Pre-Processor
- **Type / role:** `code` — normalizes metadata and generates tracking identifiers.
- **Main configuration logic:**
  - Uses the first attachment: `item.json.attachments[0]`
  - Adds:
    - `emailSubject`, `senderEmail`, `senderName`, `receivedDate`, `gmailMessageId`
    - `attachmentName`, `attachmentSize`, `attachmentMimeType`
    - `processingId` (format: `INV-<timestamp>-<random>`)
    - `intakeTimestamp`
- **Inputs:** Items from IF True path.
- **Outputs:** Same items enriched with new JSON fields.
- **Edge cases / failures:**
  - If `attachments[0]` exists but missing fields (e.g., `name`), downstream may get undefined.
  - Default sender fallback uses `user@example.com` which can hide data quality issues.
  - `Date.now()` means IDs are time-based; concurrency can still collide in rare cases.

---

### Block B — PDF Parsing & AI-Style Extraction
**Overview:** Converts the PDF into text/JSON, then extracts invoice fields with regex pattern matching and produces a confidence score plus a `requiresReview` flag.

**Nodes involved:**
- HTML to PDF: Parse to JSON
- Code: AI Ledger Mapper

#### Node: HTML to PDF: Parse to JSON
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — external service node performing PDF parsing.
- **Configuration choices (interpreted):**
  - Resource: `pdfManipulation`
  - Operation: `parsePdfToJson`
  - Credential: “pdf munk - deepanshi” (htmlcsstopdfApi)
- **Inputs:** Enriched email item from Pre-Processor.
- **Outputs:** Parsed result is expected to include `item.json.text` (used by the next code node).
- **Version-specific requirements:**
  - This is a **community/custom node** (`n8n-nodes-htmlcsstopdf`), so it must be installed on the n8n instance.
- **Edge cases / failures:**
  - API auth failures or quota limits.
  - PDF parsing may return empty/partial `text` for scanned PDFs (image-only).
  - Large PDFs may time out or exceed service limits.

#### Node: Code: AI Ledger Mapper
- **Type / role:** `code` — pattern-based extraction + validation + scoring.
- **Key logic (interpreted):**
  - Reads `const text = item.json.text || ''`
  - Regex-based extraction for:
    - `total`, `tax`, `subtotal`
    - `vendor`
    - `invoiceNumber`
    - `invoiceDate`, `dueDate`
  - Parses numeric fields and computes:
    - `subtotalAmount` defaulting to `totalAmount - taxAmount` if subtotal missing
  - Confidence scoring (max 0.99) based on found fields:
    - Amount found (0.4), vendor (0.3), invoice # (0.15), tax (0.1), date (0.05), math check bonus (0.05)
  - Validation:
    - `isValidAmount` (0 < total < 1,000,000)
    - `isValidVendor` (not “Unknown Vendor”)
    - `mathCheck` allows $1 rounding tolerance
  - Sets flags and categorization:
    - `category`: HIGH_VALUE > 5000, MEDIUM_VALUE > 1000 else LOW_VALUE
    - `requiresReview = confidence < 0.7 || totalAmount > 5000`
- **Outputs:**
  - Adds a nested object `extracted` plus flattened fields:
    - `vendorName`, `totalAmount`, `taxAmount`, `subtotalAmount`, `invoiceNumber`, `invoiceDate`, `dueDate`, `confidence`, `validationStatus`, `category`, `requiresReview`
- **Edge cases / failures:**
  - If `text` is empty, outputs default values (0 amounts, “Unknown Vendor”), almost certainly forcing review.
  - Vendor extraction regex may incorrectly capture headers, addresses, or “Bill To”.
  - Date formats outside `dd/mm/yyyy` or `mm-dd-yyyy` patterns will not parse.
  - Currency formats like `1 234,56` (EU) won’t parse correctly.
  - Multi-page invoices may include multiple totals; first match wins.

---

### Block C — Auto-Approval Gate
**Overview:** Routes invoices either to auto-sync (approved) or to a review escalation path based on `requiresReview`.

**Nodes involved:**
- IF: Auto-Approve

#### Node: IF: Auto-Approve
- **Type / role:** `if` — branching gate.
- **Condition:**
  - `{{ $json.requiresReview }}` equals `false`
- **Connections:**
  - **True** → Google Sheets: Add Row, Airtable: Create Record, Google Drive: Archive Invoice (in parallel)
  - **False** → Slack: Review Required
- **Edge cases / failures:**
  - If `requiresReview` is missing or not boolean, strict validation may cause unexpected results.
  - Since three nodes run in parallel on the “approved” branch, partial success is possible (e.g., Sheets succeeds but Airtable fails) without explicit compensation logic.

---

### Block D — Ledger Sync & Archival (Approved Path)
**Overview:** Writes the extracted invoice data to Google Sheets, creates a record in Airtable, archives the invoice to Google Drive, and notifies Slack on success (based on Drive completion).

**Nodes involved:**
- Google Sheets: Add Row
- Airtable: Create Record
- Google Drive: Archive Invoice
- Slack: Success Notification

#### Node: Google Sheets: Add Row
- **Type / role:** `googleSheets` — appends a row to the main invoice ledger.
- **Configuration choices (interpreted):**
  - Operation: Append
  - Document: `YOUR_SHEET_ID` (must be replaced)
  - Sheet tab: `Sheet1` (gid=0)
  - Mapping mode: “define below” with explicit column mapping:
    - Category, Due Date, Subtotal, Tax Amount, Sender Name/Email, Vendor Name, Invoice Date/Number, totals, Processing ID, Received Date, Confidence Score, Gmail Message ID, Validation Status, Processed At
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account - Deepanshi”)
- **Edge cases / failures:**
  - Sheet/columns mismatch (renamed headers, wrong gid).
  - Locale issues with decimals/dates.
  - Rate limits for Google Sheets API.

#### Node: Airtable: Create Record
- **Type / role:** `airtable` — creates an invoice record in Airtable.
- **Configuration choices (interpreted):**
  - Operation: Create
  - Base: `YOUR_BASE_ID` (must be replaced)
  - Table: `Invoices`
  - Field mapping is not shown in the JSON snippet (likely configured in UI or omitted). As-is, record creation may fail if Airtable requires fields.
- **Edge cases / failures:**
  - Missing required fields in Airtable schema.
  - Base/table IDs wrong.
  - Airtable rate limits.

#### Node: Google Drive: Archive Invoice
- **Type / role:** `googleDrive` — archives invoice file into a Drive folder.
- **Configuration choices (interpreted):**
  - Drive: “My Drive”
  - Folder: `INVOICE_ARCHIVE_FOLDER_ID`
  - The operation specifics (upload/create) are not explicit in provided parameters; typically this node needs binary data mapping to upload.
- **Credentials:** Google Drive OAuth2 (“PDFMunk - Jitesh”)
- **Connections:**
  - On completion → Slack: Success Notification
- **Edge cases / failures:**
  - If the workflow never keeps the PDF binary (only parses it), this node may have no file content to upload unless the trigger item contains binary data or the parsing node outputs binary.
  - Folder permissions and shared drive vs “My Drive” mismatches.
  - File name collisions if not uniquely set.

#### Node: Slack: Success Notification
- **Type / role:** `slack` — sends a Slack message (operation: create).
- **Configuration choices (interpreted):**
  - Uses a Slack webhook configured in the node (webhook id set).
  - Message content is not specified in the JSON snippet; may rely on Slack node defaults or UI configuration.
- **Edge cases / failures:**
  - Slack webhook misconfigured or revoked.
  - Missing channel target (depends on Slack credential type).

---

### Block E — Review Queue Escalation (Review Path)
**Overview:** For flagged invoices, posts a Slack alert, adds an entry to a Google Sheets “Review Queue” tab, and emails a human reviewer with details.

**Nodes involved:**
- Slack: Review Required
- Google Sheets: Review Queue
- Gmail: Review Alert

#### Node: Slack: Review Required
- **Type / role:** `slack` — creates a Slack message for manual review escalation.
- **Connections:** Output → Google Sheets: Review Queue
- **Edge cases / failures:**
  - Same Slack webhook concerns as above.
  - No message text shown; ensure it contains key fields (processingId, vendor, total, confidence).

#### Node: Google Sheets: Review Queue
- **Type / role:** `googleSheets` — appends a row to a dedicated review queue tab.
- **Configuration choices (interpreted):**
  - Operation: Append
  - Document: `YOUR_SHEET_ID`
  - Sheet tab: `Review Queue` (gid=1)
  - Columns mapped:
    - Status = `PENDING_REVIEW`
    - Category, Confidence, Flagged At (`{{ $now.toISO() }}`)
    - Vendor Name, Total Amount, Processing ID, Review Reason (low confidence vs high value), Gmail Message ID
- **Edge cases / failures:**
  - Sheet tab gid mismatch.
  - Concurrent reviews require unique `Processing ID` to avoid ambiguity.

#### Node: Gmail: Review Alert
- **Type / role:** `gmail` (send email) — not a trigger; sends an email to the reviewer.
- **Configuration choices (interpreted):**
  - To: `user@example.com` (placeholder)
  - CC: `user@example.com`
  - Subject includes vendor + amount: `Invoice Review Required: {{ $json.vendorName }} - ${{ $json.totalAmount }}`
  - Body includes dynamic fields and explicit instructions to review Gmail Message ID
  - Attachments UI configured but effectively empty (`attachmentsBinary` contains an empty object), so no PDF is actually attached as configured.
- **Credentials:** Gmail OAuth2 (“jitesh0dugar@gmail.com”)
- **Edge cases / failures:**
  - Sending limits / spam policies.
  - If `totalAmount` is a number, `.toFixed(2)` is safe; if it became a string, expression could fail.
  - Reviewers may need a direct Gmail link; only message ID is provided.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky_Intake | Sticky Note | Documentation (Phase label) |  |  | ## 📥 PHASE 1: Intake & Validation<br>Triggers on Gmail attachments. Validates PDF format, filters out non-invoice emails, and performs initial data quality checks. |
| Sticky_Parsing | Sticky Note | Documentation (Phase label) |  |  | ## 🔬 PHASE 2: AI Parsing & Extraction<br>**Parse PDF to JSON:** Converts binary PDF to structured data. **AI Ledger Mapper:** Extracts financial fields with intelligent pattern matching and confidence scoring. |
| Sticky_Sync | Sticky Note | Documentation (Phase label) |  |  | ## 🚀 PHASE 3: Ledger Sync & Archival<br>Updates Google Sheets ledger, creates Airtable records, archives to Drive. High-value or low-confidence items trigger Slack escalations for manual review. |
| Sticky_ReviewQueue | Sticky Note | Documentation (Phase label) |  |  | ## ⚠️ PHASE 4: Review Queue<br>Low confidence scores (<0.7) or high-value invoices (>$5000) require manual approval before final posting. |
| Gmail: Watch Invoices | Gmail Trigger | Workflow entry: watch Gmail events | — | IF: Valid PDF Attachment | ## 📥 PHASE 1: Intake & Validation<br>Triggers on Gmail attachments. Validates PDF format, filters out non-invoice emails, and performs initial data quality checks. |
| IF: Valid PDF Attachment | IF | Validate attachment exists and is PDF | Gmail: Watch Invoices | Code: Pre-Processor (true path) | ## 📥 PHASE 1: Intake & Validation<br>Triggers on Gmail attachments. Validates PDF format, filters out non-invoice emails, and performs initial data quality checks. |
| Code: Pre-Processor | Code | Add metadata + processingId | IF: Valid PDF Attachment | HTML to PDF: Parse to JSON | ## 📥 PHASE 1: Intake & Validation<br>Triggers on Gmail attachments. Validates PDF format, filters out non-invoice emails, and performs initial data quality checks. |
| HTML to PDF: Parse to JSON | htmlcsstopdf | Parse PDF to text/JSON | Code: Pre-Processor | Code: AI Ledger Mapper | ## 🔬 PHASE 2: AI Parsing & Extraction<br>**Parse PDF to JSON:** Converts binary PDF to structured data. **AI Ledger Mapper:** Extracts financial fields with intelligent pattern matching and confidence scoring. |
| Code: AI Ledger Mapper | Code | Extract invoice fields + confidence + flags | HTML to PDF: Parse to JSON | IF: Auto-Approve | ## 🔬 PHASE 2: AI Parsing & Extraction<br>**Parse PDF to JSON:** Converts binary PDF to structured data. **AI Ledger Mapper:** Extracts financial fields with intelligent pattern matching and confidence scoring. |
| IF: Auto-Approve | IF | Route approved vs review | Code: AI Ledger Mapper | Google Sheets: Add Row; Airtable: Create Record; Google Drive: Archive Invoice (true) / Slack: Review Required (false) | ## 🚀 PHASE 3: Ledger Sync & Archival<br>Updates Google Sheets ledger, creates Airtable records, archives to Drive. High-value or low-confidence items trigger Slack escalations for manual review. |
| Google Sheets: Add Row | Google Sheets | Append invoice row to ledger tab | IF: Auto-Approve (true) | — | ## 🚀 PHASE 3: Ledger Sync & Archival<br>Updates Google Sheets ledger, creates Airtable records, archives to Drive. High-value or low-confidence items trigger Slack escalations for manual review. |
| Airtable: Create Record | Airtable | Create invoice record | IF: Auto-Approve (true) | — | ## 🚀 PHASE 3: Ledger Sync & Archival<br>Updates Google Sheets ledger, creates Airtable records, archives to Drive. High-value or low-confidence items trigger Slack escalations for manual review. |
| Google Drive: Archive Invoice | Google Drive | Store invoice PDF in archive folder | IF: Auto-Approve (true) | Slack: Success Notification | ## 🚀 PHASE 3: Ledger Sync & Archival<br>Updates Google Sheets ledger, creates Airtable records, archives to Drive. High-value or low-confidence items trigger Slack escalations for manual review. |
| Slack: Success Notification | Slack | Notify success (after archive) | Google Drive: Archive Invoice | — | ## 🚀 PHASE 3: Ledger Sync & Archival<br>Updates Google Sheets ledger, creates Airtable records, archives to Drive. High-value or low-confidence items trigger Slack escalations for manual review. |
| Slack: Review Required | Slack | Escalate flagged invoices | IF: Auto-Approve (false) | Google Sheets: Review Queue | ## ⚠️ PHASE 4: Review Queue<br>Low confidence scores (<0.7) or high-value invoices (>$5000) require manual approval before final posting. |
| Google Sheets: Review Queue | Google Sheets | Append flagged invoice to review tab | Slack: Review Required | Gmail: Review Alert | ## ⚠️ PHASE 4: Review Queue<br>Low confidence scores (<0.7) or high-value invoices (>$5000) require manual approval before final posting. |
| Gmail: Review Alert | Gmail | Send review email to human | Google Sheets: Review Queue | — | ## ⚠️ PHASE 4: Review Queue<br>Low confidence scores (<0.7) or high-value invoices (>$5000) require manual approval before final posting. |
| Sticky Note | Sticky Note | Global workflow description |  |  | ## 💰 Autonomous Invoice Extraction Hub (v11.0)<br><br>Industrial-grade pipeline for automated financial data entry: Gmail → JSON Parsing → AI Ledger Mapping → Multi-Tier Sync.<br><br>### ⚙️ Core Sovereign Logic<br>* **PHASE 1: Intake & Validation:** Monitors Gmail for PDF attachments and performs binary integrity checks.<br>* **PHASE 2: AI Parsing:** Uses **HTML to PDF (Parse)** to convert unstructured PDF data into structured JSON objects.<br>* **PHASE 3: Intelligence Mapping:** Custom Code Node extracts Vendor, Total, Tax, and Due Date with pattern-matching confidence scoring.<br>* **PHASE 4: Auto-Approval Gating:** - **Green Path:** Confidence > 0.7 & Value < $5k → Auto-posts to Sheets/Airtable.<br>    - **Amber Path:** Low confidence or high value → Diverts to Review Queue & Slack Alert.<br>* **PHASE 5: Compliance Archival:** Vaults original invoices in Google Drive with forensic metadata.<br><br>### 📋 Setup & Prerequisites<br>1. **Compulsory Node:** **HTML to PDF (Parse)** for data atomization.<br>2. **Targets:** Set your Google Sheet ID (Ledger & Review tabs) and Airtable Base.<br>3. **Alerts:** Configure Slack Webhook for manual review escalations.<br><br>**Metrics:** `Extraction_Confidence`, `Tax_Variance`, `Approval_Latency`. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   “Extract invoice data from Gmail PDFs to JSON, Google Sheets and Airtable”.

2. **Add Gmail Trigger node**
   - Node: **Gmail Trigger**
   - Name: `Gmail: Watch Invoices`
   - Authenticate with **Gmail OAuth2** (Google project / n8n Gmail credentials).
   - Configure it to watch the mailbox/label/search appropriate for invoices (recommended: filter to invoice label to reduce noise).

3. **Add an IF node to validate PDF**
   - Node: **IF**
   - Name: `IF: Valid PDF Attachment`
   - Condition group (AND):
     - `attachments` **not empty**: `{{ $json.attachments }}`
     - `{{ $json.attachments[0].mimeType }}` **equals** `application/pdf`
   - Connect: Gmail Trigger → IF

4. **Add Code node: Pre-Processor**
   - Node: **Code**
   - Name: `Code: Pre-Processor`
   - Paste the pre-processor JS (metadata + `processingId` generation).
   - Connect: IF (true output) → Code: Pre-Processor

5. **Install and add the PDF parsing node**
   - Ensure the community node package **`n8n-nodes-htmlcsstopdf`** is installed on your n8n instance.
   - Add node: **HTML to PDF** (from the installed package)
   - Name: `HTML to PDF: Parse to JSON`
   - Set:
     - Resource: **PDF Manipulation**
     - Operation: **Parse PDF to JSON**
   - Create/configure its API credential (htmlcsstopdfApi).
   - Connect: Code: Pre-Processor → HTML to PDF: Parse to JSON

6. **Add Code node: AI Ledger Mapper**
   - Node: **Code**
   - Name: `Code: AI Ledger Mapper`
   - Paste the extraction JS (regex extraction, confidence scoring, `requiresReview`).
   - Connect: HTML to PDF: Parse to JSON → Code: AI Ledger Mapper

7. **Add IF node: Auto-Approve**
   - Node: **IF**
   - Name: `IF: Auto-Approve`
   - Condition: `{{ $json.requiresReview }}` equals `false`
   - Connect: Code: AI Ledger Mapper → IF: Auto-Approve

8. **Approved branch: Google Sheets ledger append**
   - Node: **Google Sheets**
   - Name: `Google Sheets: Add Row`
   - Credential: Google Sheets OAuth2
   - Operation: **Append**
   - Document ID: set to your ledger spreadsheet (replace `YOUR_SHEET_ID`)
   - Sheet: select `Sheet1` (or your ledger tab)
   - Map columns to expressions:
     - Processing ID, Vendor Name, Invoice Number, Invoice Date, Due Date, Subtotal, Tax Amount, Total Amount, Sender info, Confidence, Validation Status, Category, Received Date, Processed At, Gmail Message ID
   - Connect: IF: Auto-Approve (true) → Google Sheets: Add Row

9. **Approved branch: Airtable create record**
   - Node: **Airtable**
   - Name: `Airtable: Create Record`
   - Credential: Airtable (token/personal access token)
   - Base: choose your base (replace `YOUR_BASE_ID`)
   - Table: `Invoices`
   - Map Airtable fields (must match your schema; add at least Processing ID, Vendor, Total, Invoice #, Dates, Confidence).
   - Connect: IF: Auto-Approve (true) → Airtable: Create Record

10. **Approved branch: Google Drive archive**
   - Node: **Google Drive**
   - Name: `Google Drive: Archive Invoice`
   - Credential: Google Drive OAuth2
   - Folder ID: set to your archive folder (`INVOICE_ARCHIVE_FOLDER_ID`)
   - Configure the node to **upload the original PDF binary**:
     - Ensure the Gmail trigger provides binary attachment data OR add a Gmail “Download Attachment” step (recommended) and pass the binary to Drive upload.
   - Connect: IF: Auto-Approve (true) → Google Drive: Archive Invoice

11. **Approved branch: Slack success**
   - Node: **Slack**
   - Name: `Slack: Success Notification`
   - Configure Slack credential (webhook or OAuth) and set channel/message text.
   - Connect: Google Drive: Archive Invoice → Slack: Success Notification

12. **Review branch: Slack review required**
   - Node: **Slack**
   - Name: `Slack: Review Required`
   - Configure channel/message text to include `processingId`, `vendorName`, `totalAmount`, `confidence`.
   - Connect: IF: Auto-Approve (false) → Slack: Review Required

13. **Review branch: Append to Review Queue sheet**
   - Node: **Google Sheets**
   - Name: `Google Sheets: Review Queue`
   - Operation: Append
   - Document: same sheet as ledger
   - Sheet: `Review Queue`
   - Map fields (Status, Category, Confidence, Flagged At, Vendor, Total, Processing ID, Review Reason, Gmail Message ID)
   - Connect: Slack: Review Required → Google Sheets: Review Queue

14. **Review branch: Send Gmail alert email**
   - Node: **Gmail**
   - Name: `Gmail: Review Alert`
   - Credential: Gmail OAuth2
   - Set To/CC (replace `user@example.com`)
   - Use subject/body expressions as in the workflow for dynamic content.
   - If you want the PDF attached, configure attachments properly by referencing the correct **binary property**.
   - Connect: Google Sheets: Review Queue → Gmail: Review Alert

15. **Add sticky notes (optional but recommended)**
   - Add sticky notes for phases (Intake, Parsing, Sync, Review Queue) and the global description.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Autonomous Invoice Extraction Hub (v11.0)… Metrics: `Extraction_Confidence`, `Tax_Variance`, `Approval_Latency`.” | Global workflow positioning and design intent (from the large sticky note). |
| “Compulsory Node: HTML to PDF (Parse) for data atomization.” | Requires installing/configuring the `n8n-nodes-htmlcsstopdf` node and its API credential. |
| Placeholders to replace: `YOUR_SHEET_ID`, `YOUR_BASE_ID`, `INVOICE_ARCHIVE_FOLDER_ID`, `user@example.com` | Must be set before production use. |
| Potential functional gap: Drive archive expects PDF binary | Consider adding a Gmail “download attachment” node or ensuring binary is preserved from the trigger/parser. |
| Current PDF validation only checks `attachments[0]` | If invoices arrive as the 2nd+ attachment, adjust logic to scan attachments array. |