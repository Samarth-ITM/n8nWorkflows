Generate HIPAA-safe patient-friendly EMR summaries with GPT-4, PDF tools and Google Drive

https://n8nworkflows.xyz/workflows/generate-hipaa-safe-patient-friendly-emr-summaries-with-gpt-4--pdf-tools-and-google-drive-12583


# Generate HIPAA-safe patient-friendly EMR summaries with GPT-4, PDF tools and Google Drive

## 1. Workflow Overview

**Purpose:**  
This workflow ingests an EMR PDF, splits it into single-page “atomic” documents, classifies each page (Labs/Rx/Imaging/Clinical Note/Note), redacts likely HIPAA identifiers, runs an AI step to support patient-friendly interpretation, detects anomalies, then re-assembles outputs and performs compliance validation before archiving and notifications.

**Target use cases:**
- Generating patient-friendly summaries from EMR exports while reducing accidental cross-page PII exposure
- Routing different clinical page types through a unified redaction + AI pipeline
- Creating an audit trail (hash, redaction/anomaly counts) and notifying patients/providers

**Logical blocks (by dependency chain):**
- **1.1 Intake & Atomic Splitting:** Download EMR PDF → split into per-page items
- **1.2 Clinical Classification & Routing:** Determine page type → route (currently to same redaction node)
- **1.3 HIPAA Sanitization:** Regex-based PII scrubbing with compliance metadata
- **1.4 AI Intelligence + Anomaly Detection:** LLM agent step → rule-based anomaly detector
- **1.5 Re-assembly & QA Gate:** Merge items → merge PDF → validate compliance → archive + alerts or stop with error

---

## 2. Block-by-Block Analysis

### 2.1 Intake & Atomic Splitting
**Overview:** Downloads the source EMR PDF from a URL provided in the incoming JSON, then splits it into atomic units (typically pages) so each page can be processed independently.

**Nodes involved:**
- **HTTP: EMR PDF Intake**
- **Split PDF**

#### Node: HTTP: EMR PDF Intake
- **Type / role:** `HTTP Request` — fetches the PDF file.
- **Configuration (interpreted):**
  - **URL:** `{{ $json.emrSourceUrl }}`
  - **Response format:** File (binary)
- **Key expressions/variables:**
  - `emrSourceUrl` must exist in the incoming item JSON.
- **Connections:**
  - **In:** Not defined in this JSON (no trigger node present). This workflow expects upstream execution providing `emrSourceUrl`.
  - **Out:** → **Split PDF**
- **Failure modes / edge cases:**
  - Missing/invalid `emrSourceUrl` → expression resolves empty → request fails
  - Auth-required URL, expired signed URL, or network timeouts
  - Non-PDF response (HTML error page) still treated as “file” and later split fails
- **Version notes:** TypeVersion 4.2; response format behavior can vary across n8n versions—verify “file” output lands in the expected binary property.

#### Node: Split PDF
- **Type / role:** `htmlcsstopdf` (PDF manipulation) — splits a PDF into multiple outputs (usually one per page).
- **Configuration (interpreted):**
  - **Resource:** PDF Manipulation
  - **Operation:** Split PDF
  - Uses **htmlcsstopdfApi** credentials (“pdf munk - deepanshi”)
- **Connections:**
  - **In:** ← **HTTP: EMR PDF Intake**
  - **Out:** → **Code: Clinical Classifier**
- **Failure modes / edge cases:**
  - Input binary property mismatch (split operation can fail if it can’t find the PDF binary)
  - Large PDFs may hit provider limits/timeouts
- **Version notes:** Community/third-party node; ensure the installed node package and credential format match TypeVersion 1 expectations.

**Sticky note context (applies to this phase):**
- “🛡️ PHASE 1: Intake & Atomic Splitting … Split Node explodes the chart into atomic pages…”

---

### 2.2 Clinical Classification & Routing
**Overview:** Adds clinical metadata per page by inspecting extracted text (expected to be present per page), then routes based on `pageType`. Although routed, all branches currently lead to the same redaction node.

**Nodes involved:**
- **Code: Clinical Classifier**
- **Switch: Redaction Path**

#### Node: Code: Clinical Classifier
- **Type / role:** `Code` — classifies pages and stamps metadata (hash, type, timestamps).
- **Configuration (interpreted):**
  - Reads each incoming item and expects `item.json.text` to exist (OCR/text extraction is assumed to have happened upstream or inside the split step—this workflow does not show an OCR node).
  - Sets:
    - `pageType`: one of `Lab`, `Rx`, `Imaging`, `Clinical Note`, default `Note`
    - `pageHash`: `"SHA256-" + base64(text).substring(0,16)` (note: not actually SHA-256)
    - `processedAt`, `pageNumber`
    - `hasPII`: true if text matches `SSN|DOB|Phone|Address`
- **Key expressions/variables:**
  - Uses `Buffer.from(text).toString('base64')...` to create a short “hash-like” fingerprint
- **Connections:**
  - **In:** ← **Split PDF**
  - **Out:** → **Switch: Redaction Path**
- **Failure modes / edge cases:**
  - If `text` is missing/empty, classification defaults to `Note` and `pageHash` becomes derived from empty string (collisions likely)
  - The “SHA256-” prefix is misleading; audit/non-repudiation claims are weakened
  - Name/PII detection is simplistic (false positives/negatives likely)
- **Version notes:** TypeVersion 2 (Code node). Buffer usage depends on Node.js runtime bundled with your n8n version (generally fine).

#### Node: Switch: Redaction Path
- **Type / role:** `Switch` — routes items based on `pageType`.
- **Configuration (interpreted):**
  - **Value to evaluate:** `{{ $json.pageType }}`
  - Rules for `Lab`, `Rx`, `Imaging`, `Clinical Note`
  - Items not matching any rule go to the default output (not connected here)
- **Connections:**
  - **In:** ← **Code: Clinical Classifier**
  - **Out:** All four configured outputs go to **Code: HIPAA Redactor** (same target)
- **Failure modes / edge cases:**
  - Unmatched `pageType` (e.g., `Note`) will not be processed further unless default output is connected
- **Version notes:** TypeVersion 1.

**Sticky note context (applies to this phase):**
- “🔬 PHASE 2: Clinical Classification … Switch Node routes to specific redaction paths…”

---

### 2.3 HIPAA Sanitization
**Overview:** Performs regex-based scrubbing of common identifiers and annotates the item with redaction counts and compliance status.

**Nodes involved:**
- **Code: HIPAA Redactor**

#### Node: Code: HIPAA Redactor
- **Type / role:** `Code` — sanitizes text and sets compliance fields.
- **Configuration (interpreted):**
  - Applies multiple regex patterns to `item.text`:
    - SSN, 10-digit phone, “Firstname Lastname”, technician line, street address, email, date format `MM/DD/YYYY` labeled as DOB
  - Outputs:
    - `sanitizedText`
    - `redactionCount`
    - `sanitizedAt`
    - `complianceStatus`: `PII_REMOVED` if redactions > 0 else `CLEAN`
- **Key expressions/variables:**
  - Operates on `$input.first().json` (only first item in the incoming batch)
- **Connections:**
  - **In:** ← **Switch: Redaction Path** (all branches)
  - **Out:** → **AI Agent**
- **Failure modes / edge cases:**
  - **Batch handling bug:** using `$input.first()` can drop all but one page when multiple items arrive together. If split creates multiple items, you typically want `$input.all()` and map.
  - Over-redaction: the `[A-Z][a-z]+ [A-Z][a-z]+` name pattern will redact many non-name phrases
  - Under-redaction: many HIPAA identifiers not covered (MRN, account numbers, facility identifiers, etc.)
- **Version notes:** TypeVersion 2.

**Sticky note context (applies to this phase):**
- “🚀 PHASE 3: Sanitization & Re-Assembly … Redacts PII…”

---

### 2.4 AI Intelligence + Anomaly Detection
**Overview:** Sends sanitized content through an LLM agent (configured with an OpenAI chat model), then performs deterministic anomaly detection on sanitized text.

**Nodes involved:**
- **OpenAI Chat Model**
- **AI Agent**
- **Code: Anomaly Detector**

#### Node: OpenAI Chat Model
- **Type / role:** `LangChain Chat Model (OpenAI)` — provides the LLM for the agent.
- **Configuration (interpreted):**
  - **Model:** `gpt-4.1-mini`
  - No extra options/tools configured
  - Credential: `openAiApi` (“OpenAi account 3”)
- **Connections:**
  - **Out (ai_languageModel):** → **AI Agent**
- **Failure modes / edge cases:**
  - Model access not enabled for the API key
  - Rate limits/timeouts
  - PHI policy: you are sending “sanitizedText”, but if redaction misses identifiers, PHI could still be transmitted externally
- **Version notes:** TypeVersion 1.3; ensure your n8n instance has the `@n8n/n8n-nodes-langchain` package compatible with this version.

#### Node: AI Agent
- **Type / role:** `LangChain Agent` — orchestrates LLM reasoning/action over inputs.
- **Configuration (interpreted):**
  - Minimal options shown; no explicit system prompt or tools defined in JSON
  - Receives LLM from **OpenAI Chat Model**
  - Receives page data from **Code: HIPAA Redactor**
- **Connections:**
  - **In (main):** ← **Code: HIPAA Redactor**
  - **In (ai_languageModel):** ← **OpenAI Chat Model**
  - **Out (main):** → **Code: Anomaly Detector**
- **Failure modes / edge cases:**
  - Without a defined prompt/goal, outputs may be inconsistent; you may need to add agent instructions (e.g., “summarize for patient, 6th-grade reading level, avoid medical advice”)
  - Token limits for large pages
- **Version notes:** TypeVersion 3.

#### Node: Code: Anomaly Detector
- **Type / role:** `Code` — rule-based detection of concerning patterns.
- **Configuration (interpreted):**
  - Reads `item.sanitizedText`
  - Adds:
    - `anomalies[]` with types like `HIGH_GLUCOSE`, `HYPERTENSION`, `URGENT_FLAG`
    - `requiresReview`: true if any anomaly severity is `CRITICAL`
    - `anomalyCount`
- **Connections:**
  - **In:** ← **AI Agent**
  - **Out:** → **Merge: Combine Processed Pages**
- **Failure modes / edge cases:**
  - Regexes are simplistic and can misfire (e.g., glucose units, BP formatting typically `120/80`, not `ddd/ddd`)
  - If `sanitizedText` missing, no anomalies detected
- **Version notes:** TypeVersion 2.

**Sticky note context (applies to this phase):**
- “🤖 PHASE 4: AI Intelligence Layer … Leverages Claude AI …”  
  *Note:* The implemented model is OpenAI (`gpt-4.1-mini`), not Claude.

---

### 2.5 Re-assembly & Quality Assurance Gate (Compliance, Archival, Alerts)
**Overview:** Merges processed items, merges PDFs, validates that redaction occurred, then writes an audit row, stores in Google Drive, and sends notifications; otherwise stops with an error.

**Nodes involved:**
- **Merge: Combine Processed Pages**
- **Merge multiple PDFS into one**
- **IF: Compliance Validator**
- **Postgres: Audit Trail**
- **Google Drive: Secure Storage**
- **Email: Provider Alert (Urgent)**
- **Twilio: Patient Alert**
- **Stop and Error: Failed Validation**

#### Node: Merge: Combine Processed Pages
- **Type / role:** `Merge` — combines streams/items by position.
- **Configuration (interpreted):**
  - Mode: `combine`
  - Combination mode: `mergeByPosition`
- **Connections:**
  - **In:** ← **Code: Anomaly Detector**
  - **Out:** → **Merge multiple PDFS into one**
- **Failure modes / edge cases:**
  - With only one input connected, merge behavior may be trivial/pointless; typically “combine” expects two inputs
  - If upstream batching is broken (see redactor `$input.first()`), the merge won’t represent all pages
- **Version notes:** TypeVersion 2.1.

#### Node: Merge multiple PDFS into one
- **Type / role:** `htmlcsstopdf` (PDF manipulation) — merges PDFs into a single PDF.
- **Configuration (interpreted):**
  - Resource: PDF Manipulation
  - Operation: merge (implied by node name; operation not explicitly shown beyond resource)
  - Credential: htmlcsstopdfApi (“pdf munk - deepanshi”)
- **Connections:**
  - **In:** ← **Merge: Combine Processed Pages**
  - **Out:** → **IF: Compliance Validator**
- **Failure modes / edge cases:**
  - If the workflow never creates per-page sanitized PDF binaries (only sanitized text), this node may have no PDFs to merge
  - Provider limits on number/size of PDFs
- **Version notes:** TypeVersion 1.

#### Node: IF: Compliance Validator
- **Type / role:** `IF` — gates downstream actions based on compliance fields.
- **Configuration (interpreted):**
  - Condition requires **both**:
    - `complianceStatus` equals `PII_REMOVED`
    - `redactionCount` > 0
- **Connections:**
  - **True:** → **Postgres: Audit Trail**, **Google Drive: Secure Storage**, **Email: Provider Alert (Urgent)**
  - **False:** → **Stop and Error: Failed Validation**
- **Failure modes / edge cases:**
  - Pages that are legitimately already de-identified will be marked `CLEAN` and fail the gate (even if compliant)
  - If only one page’s fields survive due to earlier `$input.first()` usage, the gate may evaluate on incomplete data
- **Version notes:** TypeVersion 2.

#### Node: Postgres: Audit Trail
- **Type / role:** `Postgres` — writes an audit row.
- **Configuration (interpreted):**
  - Executes a raw INSERT into `compliance_audit` with:
    - event = `PDF_PROCESSED`
    - hash = `{{ $json.pageHash }}`
    - redaction_count, anomaly_count
    - status = `{{ $json.complianceStatus }}`
    - timestamp = NOW()
- **Connections:**
  - **In:** ← **IF: Compliance Validator** (true path)
  - **Out:** → **Twilio: Patient Alert**
- **Failure modes / edge cases:**
  - SQL injection risk is low here but string interpolation is used; safer to use parameterized queries
  - Table/schema missing; permission denied; connection/TLS issues
  - `pageHash` is not a true SHA-256; audit integrity is weaker than claimed
- **Version notes:** TypeVersion 2.4.

#### Node: Google Drive: Secure Storage
- **Type / role:** `Google Drive` — stores output in Drive.
- **Configuration (interpreted):**
  - Drive: “My Drive”
  - Folder: `root` (/) (Root folder)
  - Credential: Google Drive OAuth2 (“PDFMunk - Jitesh”)
  - Operation is not specified in parameters; as configured, it may be incomplete (typically you must specify upload/create/update).
- **Connections:**
  - **In:** ← **IF: Compliance Validator** (true path)
  - **Out:** → **Twilio: Patient Alert**
- **Failure modes / edge cases:**
  - Missing operation/filename/binary mapping can cause runtime failure or no-op
  - Storing HIPAA data in standard Drive may violate compliance unless configured for a HIPAA-eligible Google Workspace + BAA
- **Version notes:** TypeVersion 3.

#### Node: Email: Provider Alert (Urgent)
- **Type / role:** `Email Send` — notifies provider (intended for urgent review).
- **Configuration (interpreted):**
  - Operation: sendEmail
  - Uses SMTP credential: “Zoho Mail jitesh@mediajade.com”
  - Email fields (to/subject/body) are not shown in JSON parameters; likely not configured and would fail unless defaults exist in UI.
- **Connections:**
  - **In:** ← **IF: Compliance Validator** (true path)
  - **Out:** → **Twilio: Patient Alert**
- **Failure modes / edge cases:**
  - Missing required email fields
  - SMTP auth failures, blocked ports, SPF/DKIM issues
- **Version notes:** TypeVersion 2.1.

#### Node: Twilio: Patient Alert
- **Type / role:** `Twilio` — sends patient SMS alert.
- **Configuration (interpreted):**
  - Message body includes dynamic fields:
    - `pageType`, `redactionCount`, `requiresReview`, `pageHash`
  - “View in portal” link: `https://portal.example.com/records/{{ $json.pageHash }}`
  - Credentials not shown in JSON; must be configured in n8n for the node to work.
- **Connections:**
  - **In:** ← **Postgres: Audit Trail**, **Google Drive: Secure Storage**, **Email: Provider Alert (Urgent)** (three separate incoming paths)
  - **Out:** none
- **Failure modes / edge cases:**
  - Because three nodes feed into Twilio, the patient may receive **duplicate SMS** (one per upstream branch completion)
  - Missing `to` phone number configuration (not shown)
  - Regulatory: texting medical info may be restricted; message currently includes document type and review flags
- **Version notes:** TypeVersion 1.

#### Node: Stop and Error: Failed Validation
- **Type / role:** `Stop and Error` — hard-fails the workflow when compliance gate fails.
- **Configuration (interpreted):**
  - Error message includes `pageHash` and `redactionCount`
- **Connections:**
  - **In:** ← **IF: Compliance Validator** (false path)
- **Failure modes / edge cases:**
  - If `pageHash` missing, message becomes less useful
- **Version notes:** TypeVersion 1.

**Sticky note context (applies to this phase):**
- “⚠️ PHASE 5: Quality Assurance …”
- “🚀 PHASE 3 … Merge Node compiles the final patient-ready PDF …” (re-assembly portion)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Overall workflow description & prerequisites |  |  | ## 🏥Clinical Data Orchestrator: Multi-Stage Redaction & AI Summary Hub … (includes prerequisites and metrics) |
| Sticky_Intake | Sticky Note | Phase annotation |  |  | ## 🛡️ PHASE 1: Intake & Atomic Splitting … |
| HTTP: EMR PDF Intake | HTTP Request | Download EMR PDF from source URL | (No trigger defined) | Split PDF | ## 🛡️ PHASE 1: Intake & Atomic Splitting … |
| Split PDF | htmlcsstopdf (PDF Manipulation) | Split PDF into atomic pages | HTTP: EMR PDF Intake | Code: Clinical Classifier | ## 🛡️ PHASE 1: Intake & Atomic Splitting … |
| Sticky_Triage | Sticky Note | Phase annotation |  |  | ## 🔬 PHASE 2: Clinical Classification … |
| Code: Clinical Classifier | Code | Classify page type + metadata | Split PDF | Switch: Redaction Path | ## 🔬 PHASE 2: Clinical Classification … |
| Switch: Redaction Path | Switch | Route by pageType | Code: Clinical Classifier | Code: HIPAA Redactor (all branches) | ## 🔬 PHASE 2: Clinical Classification … |
| Sticky_Assembly | Sticky Note | Phase annotation |  |  | ## 🚀 PHASE 3: Sanitization & Re-Assembly … |
| Code: HIPAA Redactor | Code | Regex PII scrubbing + compliance fields | Switch: Redaction Path | AI Agent | ## 🚀 PHASE 3: Sanitization & Re-Assembly … |
| Sticky_AI_Intelligence | Sticky Note | Phase annotation |  |  | ## 🤖 PHASE 4: AI Intelligence Layer … |
| OpenAI Chat Model | LangChain OpenAI Chat Model | LLM backend for agent |  | AI Agent (ai_languageModel) | ## 🤖 PHASE 4: AI Intelligence Layer … |
| AI Agent | LangChain Agent | LLM-based reasoning/summarization step | Code: HIPAA Redactor; OpenAI Chat Model | Code: Anomaly Detector | ## 🤖 PHASE 4: AI Intelligence Layer … |
| Code: Anomaly Detector | Code | Detect critical patterns; flag review | AI Agent | Merge: Combine Processed Pages | ## 🤖 PHASE 4: AI Intelligence Layer … |
| Merge: Combine Processed Pages | Merge | Combine items (by position) | Code: Anomaly Detector | Merge multiple PDFS into one | ## 🚀 PHASE 3: Sanitization & Re-Assembly … |
| Merge multiple PDFS into one | htmlcsstopdf (PDF Manipulation) | Merge PDFs into one | Merge: Combine Processed Pages | IF: Compliance Validator | ## 🚀 PHASE 3: Sanitization & Re-Assembly … |
| Sticky_QA1 | Sticky Note | Phase annotation |  |  | ## ⚠️ PHASE 5: Quality Assurance … |
| IF: Compliance Validator | IF | Gate: require redactions occurred | Merge multiple PDFS into one | Postgres: Audit Trail; Google Drive: Secure Storage; Email: Provider Alert (Urgent) OR Stop and Error: Failed Validation | ## ⚠️ PHASE 5: Quality Assurance … |
| Postgres: Audit Trail | Postgres | Insert compliance audit row | IF: Compliance Validator (true) | Twilio: Patient Alert | ## ⚠️ PHASE 5: Quality Assurance … |
| Google Drive: Secure Storage | Google Drive | Store output file(s) | IF: Compliance Validator (true) | Twilio: Patient Alert | ## ⚠️ PHASE 5: Quality Assurance … |
| Email: Provider Alert (Urgent) | Email Send (SMTP) | Notify provider | IF: Compliance Validator (true) | Twilio: Patient Alert | ## ⚠️ PHASE 5: Quality Assurance … |
| Twilio: Patient Alert | Twilio | SMS patient that summary is ready | Postgres: Audit Trail; Google Drive: Secure Storage; Email: Provider Alert (Urgent) |  | ## ⚠️ PHASE 5: Quality Assurance … |
| Stop and Error: Failed Validation | Stop and Error | Fail workflow on compliance gate failure | IF: Compliance Validator (false) |  | ## ⚠️ PHASE 5: Quality Assurance … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   - Add a note explaining required inbound data: the workflow expects an input JSON field `emrSourceUrl` (no trigger is included in the provided JSON).  
   - Optionally add a **Webhook** or **Manual Trigger** node and set `emrSourceUrl` in a Set node for testing.

2. **Add “HTTP: EMR PDF Intake” (HTTP Request node)**  
   - Method: GET  
   - URL: `{{ $json.emrSourceUrl }}`  
   - Response: **File** (binary)  
   - Connect trigger (if you added one) → HTTP node.

3. **Add “Split PDF” (htmlcsstopdf → PDF Manipulation → Split PDF)**  
   - Configure credentials: **htmlcsstopdfApi**  
   - Ensure it reads the binary PDF from the HTTP node (set the expected binary property if required by the node UI).  
   - Connect: HTTP → Split PDF.

4. **Add “Code: Clinical Classifier” (Code node)**  
   - Paste classification code (adapt if your split node output does not provide `json.text`).  
   - Connect: Split PDF → Clinical Classifier.

5. **Add “Switch: Redaction Path” (Switch node)**  
   - Value: `{{ $json.pageType }}` (string)  
   - Rules: `Lab`, `Rx`, `Imaging`, `Clinical Note` mapped to outputs 1–4.  
   - Connect: Clinical Classifier → Switch.

6. **Add “Code: HIPAA Redactor” (Code node)**  
   - Paste redaction code.  
   - Important: adjust logic to process **all** incoming items if your workflow handles batches (replace `$input.first()` with `$input.all().map(...)`).  
   - Connect each Switch output (and optionally default output) → HIPAA Redactor.

7. **Add “OpenAI Chat Model” (LangChain OpenAI Chat Model node)**  
   - Credentials: **OpenAI API**  
   - Model: `gpt-4.1-mini` (or available equivalent)  
   - Leave tools empty unless needed.

8. **Add “AI Agent” (LangChain Agent node)**  
   - Connect **OpenAI Chat Model** to the Agent’s **Language Model** input.  
   - Connect **HIPAA Redactor** to the Agent’s **Main** input.  
   - In the Agent configuration, add clear instructions (recommended) describing the expected output (e.g., patient-friendly summary, avoid medical advice, reference anomalies).

9. **Add “Code: Anomaly Detector” (Code node)**  
   - Paste anomaly detection code.  
   - Connect: AI Agent → Anomaly Detector.

10. **Add “Merge: Combine Processed Pages” (Merge node)**  
   - Mode: **Combine**  
   - Combination mode: **Merge by Position**  
   - Connect: Anomaly Detector → Merge node.  
   - If you only have a single stream, consider replacing this with an **Item Lists** / **Aggregate** approach depending on desired output.

11. **Add “Merge multiple PDFS into one” (htmlcsstopdf → PDF Manipulation)**  
   - Configure it to merge multiple PDFs into a single PDF.  
   - Ensure it receives binaries that are actual PDFs; if you only have text, insert a “Generate PDF from HTML” step per page before merging.  
   - Connect: Merge → Merge PDFs.

12. **Add “IF: Compliance Validator” (IF node)**  
   - Conditions (AND):
     - `{{ $json.complianceStatus }}` equals `PII_REMOVED`
     - `{{ $json.redactionCount }}` > `0`
   - Connect: Merge PDFs → IF.

13. **Add “Postgres: Audit Trail” (Postgres node)**  
   - Credentials: Postgres connection with least privilege for inserts  
   - Operation: Execute Query  
   - Query: insert event, hash, counts, status, NOW() into `compliance_audit`  
   - Connect: IF (true) → Postgres.

14. **Add “Google Drive: Secure Storage” (Google Drive node)**  
   - Credentials: Google Drive OAuth2  
   - Choose drive/folder (the JSON uses My Drive + root; you likely want a restricted folder).  
   - Configure the node to **Upload** the merged PDF (set file name and binary property).  
   - Connect: IF (true) → Google Drive.

15. **Add “Email: Provider Alert (Urgent)” (Email Send node)**  
   - Credentials: SMTP  
   - Fill required fields: **To**, **Subject**, **Text/HTML body**.  
   - Optionally only send when `requiresReview` is true (add a second IF).  
   - Connect: IF (true) → Email.

16. **Add “Twilio: Patient Alert” (Twilio node)**  
   - Credentials: Twilio  
   - Configure **From** and **To** numbers  
   - Message body: use the provided template (adapt to your compliance policy).  
   - To prevent duplicate SMS, prefer a single path into Twilio (e.g., merge branches first).  
   - Connect: Postgres → Twilio, and/or Drive → Twilio, and/or Email → Twilio (but ideally only one).

17. **Add “Stop and Error: Failed Validation” (Stop and Error node)**  
   - Error message: include `pageHash` and counts.  
   - Connect: IF (false) → Stop and Error.

18. **Add sticky notes** (optional) to annotate phases and prerequisites (content from the provided workflow notes).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Root folder should be a “HIPAA-compliant restricted drive” | Mentioned in the overall sticky note; ensure your Google environment and agreements support this. |
| “Ensure PostgreSQL uses AES-256 encryption for the audit vault.” | Mentioned in the overall sticky note; implement via disk/database encryption and managed key practices. |
| Metrics referenced: `Redaction_Count`, `Anomaly_Severity`, `HIPAA_Audit_Hash` | Mentioned in the overall sticky note; current “pageHash” is not a true SHA-256 digest. |
| Portal link used in SMS: `https://portal.example.com/records/{{ $json.pageHash }}` | Twilio message template; replace with your real portal URL and access controls. |
| Phase notes included in workflow (Intake/Triage/Sanitization/AI/QA) | Sticky notes embedded across the canvas. |