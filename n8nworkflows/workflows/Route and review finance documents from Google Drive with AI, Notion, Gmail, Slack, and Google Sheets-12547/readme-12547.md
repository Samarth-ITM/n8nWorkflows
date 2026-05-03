Route and review finance documents from Google Drive with AI, Notion, Gmail, Slack, and Google Sheets

https://n8nworkflows.xyz/workflows/route-and-review-finance-documents-from-google-drive-with-ai--notion--gmail--slack--and-google-sheets-12547


# Route and review finance documents from Google Drive with AI, Notion, Gmail, Slack, and Google Sheets

## 1. Workflow Overview

**Purpose:** Automate intake and review of finance-related documents (bulk scanned PDFs) from Google Drive. The workflow downloads a PDF, verifies integrity, splits it into pages, extracts text, classifies the document type and key fields, then routes it for approval/alerts and logs everything to Google Sheets for auditing.

**Target use cases:**
- AP/Finance teams processing scanned invoices and purchase orders
- HR/Payroll handling timesheets/payroll documents
- Legal reviewing agreements/NDAs
- Audit/compliance needing traceable logs and escalation history

### 1.1 Intake Layer (Triggers → Google Drive download)
Receives a scheduled run or a manual webhook trigger, then downloads a PDF from Google Drive.

### 1.2 Pre-processing (Integrity check → Split PDF)
Validates that a binary file exists and splits multi-page PDFs into page-level files.

### 1.3 Extraction & Intelligence (Text extraction → Classification)
Extracts text from each PDF/page and classifies it (invoice/payroll/legal/purchase_order) while extracting structured fields and a confidence score.

### 1.4 Routing & Review (Department routing → High-value escalation → Quality check)
Routes by document type, escalates high-value invoices to Notion + Gmail (CFO), and flags low-confidence items for Slack manual review.

### 1.5 Audit & Output (Google Sheets logging)
Appends a normalized record of every processed document to an audit spreadsheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Intake Layer
**Overview:** Starts the workflow either on a recurring schedule or via an HTTP webhook, then downloads the target PDF from Google Drive.  
**Nodes involved:**  
- Trigger: Daily Schedule1  
- Trigger: Webhook Scan1  
- Fetch Bulk Scan from Drive1  

#### Trigger: Daily Schedule1
- **Type / role:** `Schedule Trigger` — time-based entry point.
- **Configuration:** Runs on an interval measured in hours (as configured in the “rule.interval”).
- **Outputs:** Connects to **Fetch Bulk Scan from Drive1**.
- **Failure modes / edge cases:** n8n schedule misconfiguration; workflow deactivated; time zone expectations.

#### Trigger: Webhook Scan1
- **Type / role:** `Webhook` — manual/remote entry point.
- **Configuration:**  
  - **Method:** POST  
  - **Path:** `manual-trigger-scan`  
  - Expects an incoming JSON body optionally containing `fileId`.
- **Outputs:** Connects to **Fetch Bulk Scan from Drive1**.
- **Failure modes / edge cases:** Missing/invalid body; authentication not enforced (if not configured at webhook level); repeated calls leading to duplicate processing.

#### Fetch Bulk Scan from Drive1
- **Type / role:** `Google Drive` (download) — fetches the PDF binary.
- **Configuration choices:**  
  - **Operation:** Download  
  - **File ID:** expression: `{{ $json.body?.fileId || 'YOUR_DEFAULT_ID' }}`
    - If triggered via webhook and `body.fileId` exists, it downloads that file.
    - Otherwise it uses a hard-coded placeholder default.
- **Credentials:** Google Drive OAuth2 required.
- **Input / output:**  
  - **Input:** from Schedule trigger (no body) or Webhook (has `body`).  
  - **Output:** binary PDF data to **Verify File Integrity1**.
- **Failure modes / edge cases:** invalid fileId; Drive permission issues; downloading non-PDF; large files/timeouts; using `YOUR_DEFAULT_ID` without replacing it.

---

### Block 2 — Pre-processing (Integrity + Split)
**Overview:** Ensures the download produced valid binary data, then splits the PDF into pages for per-page text extraction.  
**Nodes involved:**  
- Verify File Integrity1  
- Split PDF into Pages1  

#### Verify File Integrity1
- **Type / role:** `Code` — validates binary presence.
- **Key logic:** Throws an error if `$input.first().binary` or `.data` is missing.
- **Output content:** Returns JSON only: `{ status: 'Ready', receivedAt: <ISO timestamp> }`
- **Important integration detail:** This node **does not pass through the binary**; it outputs only JSON.
- **Connections:** Input from **Fetch Bulk Scan from Drive1**, output to **Split PDF into Pages1**.
- **Failure modes / edge cases:**
  - If the downstream splitter requires the original binary, the binary will be lost here and splitting will fail.
  - Fix: return the original binary in the Code node (pass-through) or enable “Include Binary Data” / use item merging so the PDF binary survives.

#### Split PDF into Pages1
- **Type / role:** `HTMLCSSToPDF` (PDF manipulation: split) — splits multi-page PDF into single-page PDFs.
- **Configuration:** `resource=pdfManipulation`, `operation=splitPdf`.
- **Credentials:** htmlcsstopdf API credentials required.
- **Connections:** Input from **Verify File Integrity1**, output to **Extract from File**.
- **Failure modes / edge cases:** missing binary due to prior node; API limits; password-protected PDFs; corrupted PDFs; very large PDFs causing timeouts.

---

### Block 3 — Extraction & Intelligence
**Overview:** Extracts text from each PDF/page, then classifies the document and extracts key fields with a confidence score and review flag.  
**Nodes involved:**  
- Extract from File  
- AI: Extract & Classify  

#### Extract from File
- **Type / role:** `Extract From File` — extracts text content from PDFs.
- **Configuration:** `operation=pdf`.
- **Connections:** Input from **Split PDF into Pages1**, output to **AI: Extract & Classify**.
- **Failure modes / edge cases:** scanned PDFs without embedded text (OCR is not actually present in this workflow); empty/garbled extraction; encoding issues; node may require binary input which can be missing due to integrity node behavior.

#### AI: Extract & Classify
- **Type / role:** `Code` — heuristic “AI-like” classifier and extractor using regex rules (no external LLM call).
- **Configuration choices (interpreted):**
  - Reads: `const item = $input.first().json; const text = item.text || ''`
  - Determines `docType` and `confidence` based on keyword matches:
    - **invoice**: extracts invoice number, amount, vendor, due date; confidence 0.95
    - **payroll**: extracts employee ID, hours worked, pay period; confidence 0.92; amount set to 0
    - **legal**: extracts contract type, parties, effective date; confidence 0.88; amount set to 0
    - **purchase_order**: extracts PO number, amount; confidence 0.90
    - default: unknown, confidence 0
  - Sets computed fields:
    - `item.type`, `item.confidence`, `item.extractedData`, `item.amount`
    - `item.processedAt` (ISO timestamp)
    - `item.requiresReview = confidence < 0.85`
    - `wordCount`, `pageLength`
- **Connections:** Input from **Extract from File**, output to **Route by Department1**.
- **Failure modes / edge cases:**
  - If extracted text lives in a different field than `item.text` (common with Extract From File), classification will always see empty text and label as `unknown`.
  - Regex over-matching (e.g., “PO” matches many contexts); locale-specific amounts/dates; multi-line vendor names not captured.
  - “Confidence” is static per category, not computed from extraction quality.

---

### Block 4 — Routing & Review
**Overview:** Routes by document type, escalates high-value invoices/POs for approval, and triggers manual review for low-confidence items.  
**Nodes involved:**  
- Route by Department1  
- Escalate High Value?1  
- Notion: Create Approval Task  
- Escalate to CFO (Gmail)1  
- IF: Quality Check Required?  
- Slack: Alert Team  

#### Route by Department1
- **Type / role:** `Switch` — routes processing paths based on document type.
- **Configuration:**  
  - `value1 = {{ $json.type }}`  
  - Rules for `invoice`, `payroll`, `legal`, `purchase_order`
- **Connections (as built):**
  - **Output 0 (invoice):** to **Escalate High Value?1** and **IF: Quality Check Required?**
  - **Output 1 (payroll):** to **Log to Audit Sheet**
  - **Output 2 (legal):** to **Log to Audit Sheet**
  - **Output 3 (purchase_order):** to **Escalate High Value?1** and **IF: Quality Check Required?**
- **Failure modes / edge cases:** unknown type not handled (falls through with no route unless switch default is configured); case-sensitivity issues if type values change.

#### Escalate High Value?1
- **Type / role:** `IF` — checks if amount exceeds approval threshold.
- **Configuration:** number condition: `{{ $json.amount }} > 5000`
- **Connections:**
  - **True:** **Notion: Create Approval Task**
  - **False:** **Log to Audit Sheet**
- **Edge cases:** amount is 0 for payroll/legal by design; amount parsing can fail and become 0; currency mismatch; threshold is hard-coded.

#### Notion: Create Approval Task
- **Type / role:** `Notion` — creates a new page in a database for approval tracking.
- **Configuration:** `resource=database`, `operation=createPage`. (Database ID and properties mapping are not shown in the JSON excerpt; they must be configured.)
- **Connections:** Output to **Escalate to CFO (Gmail)1**.
- **Failure modes / edge cases:** missing Notion credentials; wrong database ID; required properties not provided; rate limiting.

#### Escalate to CFO (Gmail)1
- **Type / role:** `Gmail` — sends escalation email for high-value items.
- **Configuration choices:**
  - **To:** `user@example.com` (placeholder; must be replaced)
  - **Subject:** `⚠️ High Value Invoice Approval Required - ${{ $json.amount }}`
  - **HTML body:** includes invoice number/vendor/amount/due date/confidence, instructs to review in Notion.
- **Connections:** Output to **Log to Audit Sheet**.
- **Failure modes / edge cases:** Gmail OAuth scope/consent issues; sending limits; HTML rendering; missing fields for non-invoices (PO path also reaches here).

#### IF: Quality Check Required?
- **Type / role:** `IF` — detects low-confidence extraction / explicit review flag.
- **Configuration:** “any/or” logic:
  - `{{ $json.confidence }} < 0.85` OR `{{ $json.requiresReview }} == true`
- **Connections:**
  - **True:** **Slack: Alert Team**
  - **False:** **Log to Audit Sheet**
- **Edge cases:** `requiresReview` already mirrors confidence rule; duplicated logic is okay but redundant.

#### Slack: Alert Team
- **Type / role:** `Slack` — notifies team for manual verification.
- **Configuration:** Message template including type, confidence, amount, invoice number, vendor.
- **Connections:** Output to **Log to Audit Sheet**.
- **Failure modes / edge cases:** Slack credential/auth; channel targeting not shown (depends on node configuration in UI); message formatting; missing extracted fields.

---

### Block 5 — Audit & Output
**Overview:** Appends a row to a Google Sheet containing extracted fields, flags, and timestamps for traceability.  
**Nodes involved:**  
- Log to Audit Sheet  

#### Log to Audit Sheet
- **Type / role:** `Google Sheets` — append audit record.
- **Configuration choices:**
  - **Operation:** Append to `Sheet1` in `documentId=YOUR_SHEET_ID` (must be replaced)
  - **Mapping mode:** “defineBelow”
  - Writes columns including:
    - Timestamp (`processedAt`)
    - Document Type (`type`)
    - Amount (`amount`)
    - Vendor, Due Date
    - Invoice Number
    - Confidence percentage string
    - Requires Review Yes/No
    - Escalated Yes/No (amount > 5000)
    - Status “Processed”
  - Converts fields to string and attempts type conversion.
- **Credentials:** Google Sheets OAuth2 required.
- **Failure modes / edge cases:** wrong sheet ID/name; missing headers; append permission; type conversion issues (amount as string, confidence with `%`); rate limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: Daily Schedule1 | Schedule Trigger | Scheduled entry point | — | Fetch Bulk Scan from Drive1 | ### 📥 INTAKE LAYER; Scheduled & webhook triggers feed documents into the system |
| Trigger: Webhook Scan1 | Webhook | Manual HTTP entry point | — | Fetch Bulk Scan from Drive1 | ### 📥 INTAKE LAYER; Scheduled & webhook triggers feed documents into the system |
| Fetch Bulk Scan from Drive1 | Google Drive | Download PDF by fileId | Trigger: Daily Schedule1; Trigger: Webhook Scan1 | Verify File Integrity1 | ### 📥 INTAKE LAYER; Scheduled & webhook triggers feed documents into the system |
| Verify File Integrity1 | Code | Validate binary exists (but currently drops binary) | Fetch Bulk Scan from Drive1 | Split PDF into Pages1 | ### 🔬 EXTRACTION & INTELLIGENCE; OCR extracts text, AI classifies documents and extracts structured data with confidence scoring |
| Split PDF into Pages1 | HTMLCSSToPDF (pdfManipulation) | Split multi-page PDF into pages | Verify File Integrity1 | Extract from File | ### 🔬 EXTRACTION & INTELLIGENCE; OCR extracts text, AI classifies documents and extracts structured data with confidence scoring |
| Extract from File | Extract From File | Extract PDF text | Split PDF into Pages1 | AI: Extract & Classify | ### 🔬 EXTRACTION & INTELLIGENCE; OCR extracts text, AI classifies documents and extracts structured data with confidence scoring |
| AI: Extract & Classify | Code | Classify document + extract fields + confidence | Extract from File | Route by Department1 | ### 🔬 EXTRACTION & INTELLIGENCE; OCR extracts text, AI classifies documents and extracts structured data with confidence scoring |
| Route by Department1 | Switch | Route by type (invoice/payroll/legal/PO) | AI: Extract & Classify | Escalate High Value?1; IF: Quality Check Required?; Log to Audit Sheet | ### 🎯 ROUTING & APPROVAL; Departmental routing with multi-tier approval workflows and quality checks |
| Escalate High Value?1 | IF | Check amount > 5000 | Route by Department1 | Notion: Create Approval Task; Log to Audit Sheet | ### 🎯 ROUTING & APPROVAL; Departmental routing with multi-tier approval workflows and quality checks |
| IF: Quality Check Required? | IF | Detect low confidence / requires review | Route by Department1 | Slack: Alert Team; Log to Audit Sheet | ### 🎯 ROUTING & APPROVAL; Departmental routing with multi-tier approval workflows and quality checks |
| Notion: Create Approval Task | Notion | Create approval task page in database | Escalate High Value?1 | Escalate to CFO (Gmail)1 | ### ✅ AUDIT & ALERTS; Notion tasks, Gmail escalations, Slack alerts, and comprehensive audit logging |
| Escalate to CFO (Gmail)1 | Gmail | Email CFO for high-value approval | Notion: Create Approval Task | Log to Audit Sheet | ### ✅ AUDIT & ALERTS; Notion tasks, Gmail escalations, Slack alerts, and comprehensive audit logging |
| Slack: Alert Team | Slack | Alert team for manual review | IF: Quality Check Required? | Log to Audit Sheet | ### ✅ AUDIT & ALERTS; Notion tasks, Gmail escalations, Slack alerts, and comprehensive audit logging |
| Log to Audit Sheet | Google Sheets | Append audit row | Route by Department1; Escalate High Value?1; IF: Quality Check Required?; Escalate to CFO (Gmail)1; Slack: Alert Team | — | ### ✅ AUDIT & ALERTS; Notion tasks, Gmail escalations, Slack alerts, and comprehensive audit logging |
| Sticky Note: Intake | Sticky Note | Comment | — | — | ### 📥 INTAKE LAYER; Scheduled & webhook triggers feed documents into the system |
| Sticky Note: AI Processing | Sticky Note | Comment | — | — | ### 🔬 EXTRACTION & INTELLIGENCE; OCR extracts text, AI classifies documents and extracts structured data with confidence scoring |
| Sticky Note: Smart Routing | Sticky Note | Comment | — | — | ### 🎯 ROUTING & APPROVAL; Departmental routing with multi-tier approval workflows and quality checks |
| Sticky Note: Output Layer | Sticky Note | Comment | — | — | ### ✅ AUDIT & ALERTS; Notion tasks, Gmail escalations, Slack alerts, and comprehensive audit logging |
| Main Documentation1 | Sticky Note | Comment / embedded operational notes | — | — | ## 🔍 How it works; This intelligent document processing workflow automatically handles bulk scans from Google Drive... (includes Setup steps 1–8 as written) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger: Daily Schedule**
   - Add **Schedule Trigger**
   - Set interval to run every N hours (as desired)
   - Connect to Google Drive download node (step 3)

2. **Create Trigger: Webhook Scan**
   - Add **Webhook**
   - Method: **POST**
   - Path: **manual-trigger-scan**
   - (Optional but recommended) Configure authentication (e.g., header key) to prevent abuse
   - Connect to Google Drive download node (step 3)

3. **Create Google Drive download**
   - Add **Google Drive** node
   - Operation: **Download**
   - File ID: `{{ $json.body?.fileId || 'YOUR_DEFAULT_ID' }}`
   - Configure **Google Drive OAuth2** credentials
   - Replace `YOUR_DEFAULT_ID` with a real file ID (or redesign to list files from a folder)
   - Connect to integrity check (step 4)

4. **Create integrity check (and preserve binary)**
   - Add **Code** node named “Verify File Integrity”
   - Implement logic that validates binary exists **and passes it through**. (In the provided workflow, binary is dropped; adjust to keep binary.)
   - Connect to PDF split node (step 5)

5. **Split PDF into pages**
   - Add **HTMLCSSToPDF** node (or the same community/installed node used)
   - Resource: **pdfManipulation**
   - Operation: **splitPdf**
   - Configure htmlcsstopdf credentials
   - Connect to text extraction (step 6)

6. **Extract text from PDF**
   - Add **Extract From File**
   - Operation: **PDF**
   - Ensure the node outputs a text field that your classifier reads (commonly `text`)
   - Connect to classification (step 7)

7. **Classify + extract fields**
   - Add **Code** node “AI: Extract & Classify”
   - Paste/implement the regex-based logic:
     - Set `type`, `confidence`, `extractedData`, `amount`, `requiresReview`, `processedAt`
   - Connect to switch routing (step 8)

8. **Route by department**
   - Add **Switch** node
   - Value to evaluate: `{{ $json.type }}`
   - Add rules for:
     - `invoice`
     - `payroll`
     - `legal`
     - `purchase_order`
   - Connect outputs:
     - invoice → IF high value AND IF quality check
     - payroll → audit logging
     - legal → audit logging
     - purchase_order → IF high value AND IF quality check

9. **High value escalation check**
   - Add **IF** node “Escalate High Value?”
   - Condition: Number `{{ $json.amount }}` **larger than** `5000`
   - True → Notion task (step 10)
   - False → audit logging (step 13)

10. **Create Notion approval task**
   - Add **Notion** node
   - Resource: **Database**
   - Operation: **Create Page**
   - Select your Notion database and map properties (recommended properties: Type, Amount, Vendor, Invoice/PO number, Due Date, Confidence, Status)
   - Configure Notion credentials
   - Connect to Gmail escalation (step 11)

11. **Send Gmail to CFO**
   - Add **Gmail** node (Send)
   - To: set CFO email (replace placeholder `user@example.com`)
   - Subject/body: use expressions for amount/vendor/invoice number/confidence
   - Configure Gmail OAuth2 credentials
   - Connect to audit logging (step 13)

12. **Quality check for manual review**
   - Add **IF** node “Quality Check Required?”
   - Condition group: OR / ANY:
     - `{{ $json.confidence }}` < `0.85`
     - `{{ $json.requiresReview }}` equals `true`
   - True → Slack alert (step 14)
   - False → audit logging (step 13)

13. **Audit logging in Google Sheets**
   - Add **Google Sheets** node
   - Operation: **Append**
   - Spreadsheet ID: replace `YOUR_SHEET_ID`
   - Sheet name: `Sheet1` (or your choice)
   - Map columns (must exist in header row) such as Timestamp, Document Type, Amount, Vendor, Due Date, Invoice Number, Confidence, Requires Review, Escalated, Status
   - Configure Google Sheets OAuth2 credentials

14. **Slack alert**
   - Add **Slack** node (Post message)
   - Format message with expressions for type/confidence/amount/vendor/invoice
   - Configure Slack credentials and channel
   - Connect Slack node to audit logging (step 13)

15. **(Optional) Add sticky notes**
   - Add sticky notes for Intake, AI Processing, Smart Routing, Output Layer, and the operational notes block to match the original layout.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This intelligent document processing workflow automatically handles bulk scans from Google Drive… After splitting PDFs page-by-page, it uses OCR technology to extract text, then applies AI-powered classification…” | Main Documentation sticky note (conceptual overview). Note: current workflow contains text extraction but no dedicated OCR node; scanned-image PDFs may return empty text unless OCR is added. |
| Setup steps listed: Google Drive folder ID, PDF split configuration, OCR node, AI classification patterns, Notion DB properties, CFO email via Gmail, Slack channel, Audit sheet headers | Main Documentation sticky note (operational setup checklist). |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.