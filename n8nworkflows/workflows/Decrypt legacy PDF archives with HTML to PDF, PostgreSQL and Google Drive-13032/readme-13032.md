Decrypt legacy PDF archives with HTML to PDF, PostgreSQL and Google Drive

https://n8nworkflows.xyz/workflows/decrypt-legacy-pdf-archives-with-html-to-pdf--postgresql-and-google-drive-13032


# Decrypt legacy PDF archives with HTML to PDF, PostgreSQL and Google Drive

## 1. Workflow Overview

**Purpose:**  
This workflow ingests legacy *encrypted/protected PDF files*, generates forensic “chain-of-custody” metadata (SHA-256, timestamps, custody ID), attempts to retrieve a matching password from a **PostgreSQL key vault**, unlocks the PDF via **HTML to PDF (unlockPdf)**, validates integrity post-unlock, then archives results to **Google Drive** and notifies stakeholders via **Slack** and **Gmail**. Failures (invalid file, missing key, corrupted/unlock errors) are routed to a **quarantine** path with database logging and Slack alerting.

**Target use cases:**
- Legal discovery / compliance decryption at scale
- Forensic archiving with auditable metadata
- Controlled handling of failures for manual review

### 1.1 Intake & Custody (Forensic preprocessing)
- Compute original SHA-256, assign custody ID, extract size/metadata, validate PDF and size limit.

### 1.2 Key Vault Lookup & Routing
- Query PostgreSQL `legacy_keys` for the latest active password for the filename.
- If password exists → attempt unlock; else → quarantine.

### 1.3 Unlock & Integrity Guard
- Unlock PDF using HTML to PDF “Unlock” operation.
- Compute unlocked SHA-256 and compare to original; set integrity and processing status.

### 1.4 Audit Logging, Archival, and Notifications
- Insert forensic audit record into PostgreSQL `audit_log`.
- Upload unlocked PDF to Drive “Audit Vault” folder.
- Notify Slack + send Gmail legal summary.

### 1.5 Quarantine & Failure Handling
- Log quarantine entry in PostgreSQL `quarantine_log`.
- Alert Slack and upload file to Drive quarantine folder.
- Separate logging for validation failures (`file_validation_failures`).

---

## 2. Block-by-Block Analysis

### Block 1 — Intake & Custody

**Overview:**  
Generates immutable forensic metadata (hash + custody ID) before modification, then validates that the asset is a PDF and under 100 MB.

**Nodes involved:**
- Sticky_Intake1 (Sticky Note)
- Code: Forensic Intake1
- IF: File Validator
- Postgres: Validation Failure

#### Sticky_Intake1 (Sticky Note)
- **Role:** Visual documentation for the intake phase.
- **Notes:** Applies to the intake/validation nodes in this area.

#### Code: Forensic Intake1 (Code node)
- **Type / role:** `n8n-nodes-base.code` — computes SHA-256 fingerprint and sets custody metadata.
- **Key configuration:**
  - Uses Node.js `crypto`.
  - Reads incoming file from `item.binary.data` (base64).
  - Sets:
    - `originalHash` (SHA-256)
    - `custodyID` = `DISCOVERY-{timestamp}-{random}`
    - `fileSize`, `intakeTimestamp`, `mimeType`, `fileName`
    - `processingStatus` = `INTAKE_COMPLETE`
- **Inputs/outputs:**
  - **Input:** Requires binary file in `binary.data`.
  - **Output:** Same item with enriched `json` fields.
- **Edge cases / failures:**
  - Missing `binary.data` or unexpected binary structure → runtime error.
  - `item.json.name` not set → defaults `fileName` to `unknown.pdf` (may prevent key lookup).
- **Version requirements:** Code node v2 semantics; uses `$input.all()`.

#### IF: File Validator (IF node)
- **Type / role:** `n8n-nodes-base.if` — ensures file is a PDF and below size limit.
- **Conditions (AND):**
  - `{{$json.mimeType}}` **contains** `application/pdf`
  - `{{$json.fileSize}}` **smaller than** `104857600` (100 MB)
- **Routing:**
  - **True** → `Postgres: Key Vault1`
  - **False** → `Postgres: Validation Failure`
- **Edge cases:**
  - If `mimeType` is incorrect/missing (or not containing `application/pdf`) valid PDFs may be rejected.
  - `fileSize` is computed from decoded bytes; if binary parsing changes, may mis-measure.

#### Postgres: Validation Failure (Postgres node)
- **Type / role:** `n8n-nodes-base.postgres` — logs invalid files.
- **Configuration choices:**
  - Executes an INSERT into `file_validation_failures` with custody/file metadata.
  - Uses `{{$now.toISO()}}` for timestamp.
- **Input/Output:**
  - **Input:** Comes only from validation failure branch.
  - **Output:** No downstream connections (terminal logging).
- **Failure modes:**
  - DB connection/auth failures.
  - Table/schema mismatch (columns must exist).
  - SQL injection risk if `fileName` etc. can include quotes (this workflow uses string interpolation, not parameterized queries).

---

### Block 2 — Key Vault Lookup & Routing

**Overview:**  
Matches the file name against the PostgreSQL key vault to obtain a password, then routes to unlock or quarantine if missing.

**Nodes involved:**
- Sticky_Unlock1 (Sticky Note)
- Postgres: Key Vault1
- IF: Key Found

#### Sticky_Unlock1 (Sticky Note)
- **Role:** Visual documentation for key vault + unlock + quarantine branching.

#### Postgres: Key Vault1 (Postgres node)
- **Type / role:** Queries `legacy_keys` for an active password for the current filename.
- **Configuration choices:**
  - SQL:
    - Filters `filename = '{{ $json.fileName }}'`
    - `active = true`
    - Orders by `created_date DESC`
    - `LIMIT 1`
  - Returns: `password, key_id, encryption_type, created_date`
- **Input/Output:**
  - **Input:** From `IF: File Validator` (true).
  - **Output:** To `IF: Key Found`.
- **Important n8n behavior note:**
  - Postgres node returns rows; downstream nodes will receive the returned row(s) as items. If no row is found, it may output **0 items**, meaning `IF: Key Found` may receive nothing and the workflow may silently stop for that file unless n8n is configured to continue on fail or a “no items” handler exists.
- **Failure modes / edge cases:**
  - Filename mismatch (case sensitivity, different naming, path prefixes) → no key found.
  - SQL injection risk due to string interpolation.
  - Database latency/timeouts.

#### IF: Key Found (IF node)
- **Type / role:** Checks that `password` exists.
- **Condition:**
  - `{{$json.password}}` **notEmpty**
- **Routing:**
  - **True** → `HTML to PDF: Unlock Engine1`
  - **False** → `Postgres: Quarantine Log`
- **Edge cases:**
  - If Postgres returned no items, this node might not run for that file at all (see note above).
  - Password might exist but be incorrect for the file → unlock will fail later (captured by integrity guard / quarantine only if separately routed; here, unlock failures are not automatically sent to quarantine—see Block 3/5 notes).

---

### Block 3 — Unlock & Integrity Guard

**Overview:**  
Unlocks the PDF using the HTML to PDF provider, then verifies output integrity by hashing the unlocked binary and comparing to the original.

**Nodes involved:**
- HTML to PDF: Unlock Engine1
- Code: Integrity Guard1

#### HTML to PDF: Unlock Engine1 (HTML to PDF node)
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — performs PDF manipulation `unlockPdf`.
- **Configuration choices:**
  - `resource: pdfManipulation`
  - `operation: unlockPdf`
  - Uses credential: `htmlcsstopdfApi` (“pdf munk - deepanshi”)
- **Inputs/Outputs:**
  - **Input:** From `IF: Key Found` true branch. Must include:
    - Binary PDF (expected in `binary.data`)
    - Password field available as `json.password` (the node typically needs a password parameter; however, in this workflow JSON provided, no explicit mapping is shown—ensure the node is configured to use `{{$json.password}}` in its password field in the n8n UI).
  - **Output:** Decrypted/unlocked PDF binary to `Code: Integrity Guard1`.
- **Failure modes:**
  - API auth/credit limits.
  - Wrong password → API error or unchanged PDF.
  - Input binary field mismatch (provider expects a specific binary property name).
  - Large files near limit causing timeouts.

#### Code: Integrity Guard1 (Code node)
- **Type / role:** Post-process integrity verification + metrics.
- **Configuration choices:**
  - Reads unlocked output from `item.binary.data.data` (note: this assumes the unlock node outputs nested binary structure).
  - Computes `unlockedHash`, `unlockedSize`, sets `unlockTimestamp`.
  - Compares `originalHash` vs `unlockedHash`:
    - same → `integrityStatus = WARNING...`, `processingStatus = UNLOCK_SUSPICIOUS`
    - different → `integrityStatus = VERIFIED...`, `processingStatus = UNLOCK_SUCCESS`
  - Calculates `processingTimeMs = unlockTime - intakeTime`.
  - Catches errors and sets `processingStatus = UNLOCK_FAILED`.
- **Input/Output:**
  - **Input:** From unlock node.
  - **Output:** To `Postgres: Audit Logger`.
- **Edge cases / failures:**
  - Binary path mismatch: some nodes expose binary as `item.binary.data` (not `item.binary.data.data`). If structure differs, Buffer conversion fails and sets `UNLOCK_FAILED`.
  - Hash-equality heuristic: some unlock operations may rewrite the PDF but still produce same content hash in rare cases, or conversely may change bytes without actually unlocking (false positive/negative).
  - If unlock fails upstream but still outputs something, status may be misleading.

---

### Block 4 — Audit Logging, Archival, and Notifications

**Overview:**  
Writes a forensic audit record to Postgres, uploads the unlocked PDF to a custody-based Google Drive folder, then notifies Slack and emails Legal.

**Nodes involved:**
- Sticky_Audit1 (Sticky Note)
- Postgres: Audit Logger
- Google Drive: Audit Vault1
- Slack: Compliance Log1
- Gmail: Legal Summary1

#### Sticky_Audit1 (Sticky Note)
- **Role:** Visual documentation for audit + vault + notifications.

#### Postgres: Audit Logger (Postgres node)
- **Type / role:** Inserts processing/audit metadata into `audit_log`.
- **Configuration choices:**
  - SQL INSERT includes:
    - custody_id, file_name, original_hash, unlocked_hash
    - file_size, intake_timestamp, unlock_timestamp, processing_time_ms
    - integrity_status, processing_status, key_id, encryption_type
  - `RETURNING audit_id;`
- **Input/Output:**
  - **Input:** From `Code: Integrity Guard1`.
  - **Output:** To `Google Drive: Audit Vault1`.
- **Critical data-shape note:**
  - The INSERT returns a row containing `audit_id`, but unless you merge it into the existing item, downstream nodes may lose earlier fields or may only have the returned row depending on n8n node behavior. In many n8n DB nodes, the output becomes the query result rows, not the original item with appended fields.
  - However, `Gmail: Legal Summary1` references `{{$json.audit_id}}` and also references many other fields (`custodyID`, hashes, timestamps). If the Postgres node output does not include those, email/slack content will break.
  - To make this robust, typically you would enable “Include Input Data” (if available) or add a Merge node.
- **Failure modes:**
  - Schema mismatch.
  - Quoting issues / SQL injection risk.
  - Timestamp format expectations in DB.

#### Google Drive: Audit Vault1 (Google Drive node)
- **Type / role:** Uploads/places decrypted file into an Audit Vault folder.
- **Configuration choices:**
  - Uses Drive OAuth2 credential “PDFMunk - Jitesh”.
  - Folder ID expression:  
    `{{ 'AUDIT_VAULT_' + $json.custodyID.split('-')[1] }}`
    - This assumes `custodyID` has `DISCOVERY-{timestamp}-{rand}`; split index `[1]` becomes the timestamp.
- **Input/Output:**
  - **Input:** From `Postgres: Audit Logger`.
  - **Output:** Fans out to:
    - `Slack: Compliance Log1`
    - `Gmail: Legal Summary1`
- **Edge cases / failures:**
  - The expression returns a *string*, but Google Drive “folderId” typically expects an actual Drive folder ID, not a folder name. If `AUDIT_VAULT_...` is not a valid folder ID, upload will fail.
  - If the Postgres node replaced the item content and `custodyID` is missing, the expression fails.
  - OAuth token expiry / permission errors.
  - Missing binary data if prior nodes didn’t pass binary through.

#### Slack: Compliance Log1 (Slack node)
- **Type / role:** Posts a compliance log message to Slack.
- **Configuration choices:**
  - Operation: `create`
  - Authentication: OAuth2 (credential “Mediajade Slack”)
  - A `webhookId` is present in JSON, but OAuth2 is specified; ensure the Slack node is configured consistently (either webhook-based or OAuth-based).
- **Input/Output:**
  - **Input:** From `Google Drive: Audit Vault1`.
  - **Output:** Terminal.
- **Failure modes:**
  - OAuth scope/channel mismatch.
  - Message payload not configured (JSON doesn’t show message text fields; may be incomplete in export).

#### Gmail: Legal Summary1 (Gmail node)
- **Type / role:** Sends a structured legal/compliance email summary.
- **Configuration choices:**
  - To: `user@example.com`
  - Subject: `✅ Decryption Complete: {{ $json.custodyID }}`
  - Body: includes custody ID, file name, hashes, size in MB, timestamps, duration, status, integrity, audit_id, vault location.
  - Uses Gmail OAuth2 credential `jitesh0dugar@gmail.com`.
- **Input/Output:** From `Google Drive: Audit Vault1`, terminal.
- **Edge cases / failures:**
  - Missing fields (see Postgres output-shape issue).
  - Gmail API quota, permission, or “From” address restrictions.
  - If `fileSize` missing or not number, MB formatting fails.

---

### Block 5 — Quarantine & Failure Handling

**Overview:**  
Captures missing-key and other failure cases into a quarantine log, alerts Slack, and stores the file in a quarantine Drive folder for manual review.

**Nodes involved:**
- Sticky_Quarantine (Sticky Note)
- Postgres: Quarantine Log
- Slack: Quarantine Alert
- Google Drive: Quarantine Vault

#### Sticky_Quarantine (Sticky Note)
- **Role:** Visual documentation for quarantine zone behavior.

#### Postgres: Quarantine Log (Postgres node)
- **Type / role:** Inserts a quarantine record for manual review.
- **Configuration choices:**
  - Inserts into `quarantine_log`:
    - custody_id, file_name, failure_reason (defaults to `"Key not found in vault"`), timestamp, original_hash
    - `quarantine_status = 'PENDING_REVIEW'`
  - `RETURNING quarantine_id;`
- **Input/Output:**
  - **Input:** From `IF: Key Found` false branch.
  - **Output:** To `Slack: Quarantine Alert`.
- **Edge cases / failures:**
  - As with audit logger, returned row may replace upstream fields unless input is preserved—yet later nodes may still need original binary to upload.
  - If binary is not carried forward, quarantine upload cannot happen.

#### Slack: Quarantine Alert (Slack node)
- **Type / role:** Sends an alert to Slack when a file is quarantined.
- **Configuration choices:**
  - Operation `create`
  - Uses `webhookId: "quarantine-webhook-id"` (suggesting webhook-based posting)
- **Input/Output:** To `Google Drive: Quarantine Vault`.
- **Failure modes:**
  - Invalid webhook ID / missing webhook URL mapping in n8n.
  - Message payload not shown; may require configuring text/blocks.

#### Google Drive: Quarantine Vault (Google Drive node)
- **Type / role:** Stores quarantined file in Drive folder `QUARANTINE_VAULT`.
- **Configuration choices:**
  - Drive credential “PDFMunk - Jitesh”
  - Folder ID: `QUARANTINE_VAULT` (must be a real Drive folder ID)
- **Input/Output:** Terminal.
- **Edge cases / failures:**
  - Folder ID vs folder name confusion (must be actual folder ID).
  - Missing binary on the quarantine branch if Postgres output overwrote the item.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Global workflow description & prerequisites | — | — | ## ⚖️ Sovereign Decryption & Compliance Gateway v10.0… (full note content) |
| Sticky_Intake1 | n8n-nodes-base.stickyNote | Phase label: intake & custody | — | — | ## 🛡️ PHASE 1: Intake & Custody… |
| Code: Forensic Intake1 | n8n-nodes-base.code | Compute original SHA-256, custody ID, metadata | (not shown in JSON; implied entry) | IF: File Validator | ## 🛡️ PHASE 1: Intake & Custody… |
| IF: File Validator | n8n-nodes-base.if | Validate PDF MIME and size < 100MB | Code: Forensic Intake1 | Postgres: Key Vault1; Postgres: Validation Failure | ## 🛡️ PHASE 1: Intake & Custody… |
| Postgres: Validation Failure | n8n-nodes-base.postgres | Log invalid file type/oversize to DB | IF: File Validator (false) | — | ## 🛡️ PHASE 1: Intake & Custody… |
| Sticky_Unlock1 | n8n-nodes-base.stickyNote | Phase label: key vault & unlock | — | — | ## 🔐 PHASE 2: Key Vault & Unlock… |
| Postgres: Key Vault1 | n8n-nodes-base.postgres | Lookup password/key metadata in `legacy_keys` | IF: File Validator (true) | IF: Key Found | ## 🔐 PHASE 2: Key Vault & Unlock… |
| IF: Key Found | n8n-nodes-base.if | Route based on password presence | Postgres: Key Vault1 | HTML to PDF: Unlock Engine1; Postgres: Quarantine Log | ## 🔐 PHASE 2: Key Vault & Unlock… |
| HTML to PDF: Unlock Engine1 | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Unlock PDF using external API | IF: Key Found (true) | Code: Integrity Guard1 | ## 🔐 PHASE 2: Key Vault & Unlock… |
| Code: Integrity Guard1 | n8n-nodes-base.code | Hash unlocked PDF, set status, compute duration | HTML to PDF: Unlock Engine1 | Postgres: Audit Logger | ## ⚖️ PHASE 3: Audit & Archival… |
| Sticky_Audit1 | n8n-nodes-base.stickyNote | Phase label: audit & archival | — | — | ## ⚖️ PHASE 3: Audit & Archival… |
| Postgres: Audit Logger | n8n-nodes-base.postgres | Insert audit log record (returns audit_id) | Code: Integrity Guard1 | Google Drive: Audit Vault1 | ## ⚖️ PHASE 3: Audit & Archival… |
| Google Drive: Audit Vault1 | n8n-nodes-base.googleDrive | Upload decrypted file to audit vault folder | Postgres: Audit Logger | Slack: Compliance Log1; Gmail: Legal Summary1 | ## ⚖️ PHASE 3: Audit & Archival… |
| Slack: Compliance Log1 | n8n-nodes-base.slack | Post success/compliance message to Slack | Google Drive: Audit Vault1 | — | ## ⚖️ PHASE 3: Audit & Archival… |
| Gmail: Legal Summary1 | n8n-nodes-base.gmail | Email legal summary report | Google Drive: Audit Vault1 | — | ## ⚖️ PHASE 3: Audit & Archival… |
| Sticky_Quarantine | n8n-nodes-base.stickyNote | Quarantine zone description | — | — | ## ⚠️ ERROR HANDLING… |
| Postgres: Quarantine Log | n8n-nodes-base.postgres | Log quarantine case to DB | IF: Key Found (false) | Slack: Quarantine Alert | ## ⚠️ ERROR HANDLING… |
| Slack: Quarantine Alert | n8n-nodes-base.slack | Alert Slack about quarantine | Postgres: Quarantine Log | Google Drive: Quarantine Vault | ## ⚠️ ERROR HANDLING… |
| Google Drive: Quarantine Vault | n8n-nodes-base.googleDrive | Upload quarantined file to quarantine folder | Slack: Quarantine Alert | — | ## ⚠️ ERROR HANDLING… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **“Decrypt legacy PDF archives with HTML to PDF, PostgreSQL and Google Drive”**

2. **Add an entry/trigger node (required, missing in JSON).** Choose one depending on your intake mechanism:
   - **Manual Trigger** (for testing), or
   - **Google Drive Trigger** (new file), or
   - **Webhook** (file upload), etc.  
   Ensure the trigger outputs a binary PDF in a consistent property (ideally `binary.data`) and sets a filename in `json.name` or similar.

3. **Add “Code” node** named **“Code: Forensic Intake1”** and paste logic that:
   - Reads `binary.data` base64
   - Computes `originalHash` (SHA-256)
   - Sets `custodyID`, `fileSize`, `intakeTimestamp`, `mimeType`, `fileName`, `processingStatus`  
   (Use the provided code content from the workflow.)

4. **Add “IF” node** named **“IF: File Validator”**:
   - Condition group: AND
   - String condition: `{{$json.mimeType}}` contains `application/pdf`
   - Number condition: `{{$json.fileSize}}` smaller than `104857600`
   - True output → Key vault lookup
   - False output → Validation failure logger

5. **Add “Postgres” node** named **“Postgres: Validation Failure”** (connect from IF false):
   - Operation: Execute Query
   - Insert into `file_validation_failures` with the fields used in the SQL:
     - custody_id, file_name, mime_type, file_size, failure_reason, timestamp
   - Configure **PostgreSQL credentials** (host, db, user, password, SSL as required).

6. **Add “Postgres” node** named **“Postgres: Key Vault1”** (connect from IF true):
   - Operation: Execute Query
   - Query `legacy_keys` filtered by `filename = {{$json.fileName}}` and `active = true`, ordered by `created_date DESC`, limit 1.
   - Ensure table exists with: `filename, password, key_id, encryption_type, created_date, active`.

7. **Add “IF” node** named **“IF: Key Found”** (connect from Key Vault1):
   - Condition: `{{$json.password}}` notEmpty
   - True output → Unlock node
   - False output → Quarantine logging

8. **Add “HTML to PDF” node** named **“HTML to PDF: Unlock Engine1”** (connect from IF true):
   - Resource: `pdfManipulation`
   - Operation: `unlockPdf`
   - Configure **HTMLCSS to PDF credentials** (API key) via `htmlcsstopdfApi`.
   - In node parameters (in UI), ensure:
     - The input binary property is set to the incoming PDF (commonly `data`)
     - The password parameter is mapped to `{{$json.password}}`  
     (This mapping is essential even though it’s not explicit in the exported snippet.)

9. **Add “Code” node** named **“Code: Integrity Guard1”** (connect from Unlock):
   - Compute `unlockedHash`, `unlockTimestamp`, `processingTimeMs`
   - Set `integrityStatus` and `processingStatus`
   - Note: adjust binary read path if needed (`item.binary.data` vs `item.binary.data.data`) based on the unlock node’s actual output.

10. **Add “Postgres” node** named **“Postgres: Audit Logger”** (connect from Integrity Guard):
   - Operation: Execute Query
   - INSERT into `audit_log` with the listed columns and `RETURNING audit_id;`
   - Ensure table exists with:  
     `custody_id, file_name, original_hash, unlocked_hash, file_size, intake_timestamp, unlock_timestamp, processing_time_ms, integrity_status, processing_status, key_id, encryption_type`.

11. **Preserve original item fields after DB insert (recommended).**  
   Because the audit insert returns `audit_id`, add one of these (to ensure Gmail/Slack still see custody/hash fields):
   - Enable Postgres node option to **include input data** (if available in your n8n version), or
   - Add a **Merge** node: merge original item + audit insert result, then continue.

12. **Add “Google Drive” node** named **“Google Drive: Audit Vault1”**:
   - Credential: Google Drive OAuth2
   - Operation: upload/create file (set as needed in UI)
   - Folder: use a *real folder ID*. If you want dynamic folders, implement a “Find/Create Folder” step first, then use its ID.
   - If you keep the existing expression, ensure it returns a valid folder ID (not just a name).

13. **Add “Slack” node** named **“Slack: Compliance Log1”** (connect from Drive Audit Vault):
   - Configure Slack OAuth2 credentials
   - Operation: create message
   - Set channel and message content (include custodyID, status, drive link if available).

14. **Add “Gmail” node** named **“Gmail: Legal Summary1”** (also connect from Drive Audit Vault):
   - Gmail OAuth2 credential
   - Configure recipient, subject, and body using the workflow’s expressions.
   - Ensure the incoming item still contains: custodyID, fileName, hashes, timestamps, processing status, and `audit_id`.

15. **Quarantine branch (from IF: Key Found false):**
   - Add “Postgres” node **“Postgres: Quarantine Log”**
     - INSERT into `quarantine_log` with default failure reason and `PENDING_REVIEW`, `RETURNING quarantine_id;`
   - Add “Slack” node **“Slack: Quarantine Alert”**
     - Either configure webhook-based Slack (incoming webhook URL) or OAuth2—keep it consistent.
   - Add “Google Drive” node **“Google Drive: Quarantine Vault”**
     - Upload the problematic file to a quarantine folder (must be actual folder ID).
   - As with audit, ensure binary + metadata survive the Postgres return (Merge/include-input).

16. **Add Sticky Notes (optional but matching original):**
   - Place the four sticky notes with the provided contents to document phases and error handling.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “⚖️ Sovereign Decryption & Compliance Gateway v10.0 … Metrics: `Unlock_Success_Rate`, `Processing_Latency`, `Compliance_Audit_Hash`.” | Global sticky note describing phases, prerequisites, and metrics (top-level workflow documentation). |
| “Compulsory Node: HTML to PDF (Unlock) for core decryption.” | Operational constraint from the global sticky note—workflow depends on the HTMLCSS to PDF provider’s unlock capability. |
| “Databases: PostgreSQL tables for `legacy_keys`, `audit_log`, and `quarantine_log`.” | Ensure schemas exist and credentials are configured before execution. |
| “Folders: Define Drive IDs for the Audit Vault and Quarantine Zone.” | Google Drive nodes require actual folder IDs; dynamic naming requires folder lookup/create logic. |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.