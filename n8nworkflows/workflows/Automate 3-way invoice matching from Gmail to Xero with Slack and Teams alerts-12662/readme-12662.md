Automate 3-way invoice matching from Gmail to Xero with Slack and Teams alerts

https://n8nworkflows.xyz/workflows/automate-3-way-invoice-matching-from-gmail-to-xero-with-slack-and-teams-alerts-12662


# Automate 3-way invoice matching from Gmail to Xero with Slack and Teams alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automate 3-way invoice matching from Gmail to Xero with Slack and Teams alerts

**Purpose:**  
This workflow automates an Accounts Payable intake-to-posting process: it watches for incoming invoices in **Gmail**, retrieves a vendor-specific **PDF decryption key** (“Vendor Vault” in Google Sheets), **unlocks password-protected PDFs**, runs a lightweight “intelligence” step (vendor identification + currency conversion placeholder), fetches supporting documents (PO/Receipt) from **Google Drive**, **merges PDFs** into a single audit bundle, creates a **Bill in Xero**, then routes alerts to **Slack** or **Microsoft Teams** based on invoice type.

**Primary use cases:**
- High-volume invoice intake from email
- Handling encrypted/protected vendor invoices
- Producing an audit bundle (invoice + PO + receipt)
- ERP sync to Xero + differentiated alerting for urgent vs standard items

### 1.1 Logical Blocks
1. **Phase 1: Multi-Channel Ingestion & Security (Gmail → Vendor Vault → PDF Unlock)**
2. **Phase 2: Intelligence & Extraction (Code-based vendor/currency logic placeholder)**
3. **Phase 3: 3-Way Matching & Archival (Fetch PO/Receipt → Merge PDFs → Create Xero Bill)**
4. **Phase 4: ERP Sync & Multi-Channel Alerting (Switch → Slack or Teams)**

> Note: The sticky notes mention “IMAP, Webhooks, Parse PDF to JSON, and 3-way match confirmation”, but those nodes are not present in this JSON. The implemented workflow is a simplified version of that concept.

---

## 2. Block-by-Block Analysis

### Block 1 — Phase 1: Multi-Channel Ingestion & Security
**Overview:**  
Captures invoice emails from Gmail, looks up a decryption key from a “Vendor Vault” Google Sheet, then unlocks password-protected PDFs using HTMLCSS to PDF.

**Nodes involved:**
- Gmail: Watch Invoices
- Vault: Get Decryption Key
- Unlock password protected PDF

#### Node: Gmail: Watch Invoices
- **Type / role:** `n8n-nodes-base.gmail` — Gmail trigger to detect incoming invoice emails.
- **Configuration (interpreted):**
  - Uses Gmail node (v2.1). The JSON doesn’t show the specific trigger event or query/label filters—only `options: {}` is present.
  - Has a `webhookId`, which is used internally by n8n for trigger registrations.
- **Inputs/outputs:**
  - **Output:** Connects to **Vault: Get Decryption Key**.
- **Key variables/expressions:** None shown.
- **Failure/edge cases:**
  - Gmail OAuth credential missing/expired → auth errors.
  - Trigger not filtered → may pick up non-invoice emails (noise).
  - Attachments not handled explicitly in the workflow (no “download attachments” node shown).
- **Version-specific notes:** Gmail node v2.x has different operations and trigger behavior than older versions; ensure you select the correct trigger mode in UI.

#### Node: Vault: Get Decryption Key
- **Type / role:** `n8n-nodes-base.googleSheets` — looks up vendor-related data (intended: decryption password/key).
- **Configuration (interpreted):**
  - Operation: **lookup**
  - Document ID: `VENDOR_VAULT_ID` (placeholder)
  - The lookup criteria (which column/value to match) is not present in the snippet; in practice this must be configured in the node UI.
- **Inputs/outputs:**
  - **Input:** From Gmail trigger.
  - **Output:** To **Unlock password protected PDF**.
- **Key variables/expressions:** Not defined, but typically you’d map sender/vendor or extracted vendor name to the lookup value.
- **Failure/edge cases:**
  - Google Sheets credentials missing/expired.
  - Lookup returns no rows → downstream unlock will fail if password is missing.
  - Vendor name normalization issues (case, punctuation).
- **Version-specific notes:** Google Sheets node v4 uses a newer resource model; ensure correct sheet/tab and lookup mapping.

#### Node: Unlock password protected PDF
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — unlocks encrypted PDFs (PDF security operation).
- **Configuration (interpreted):**
  - Resource: **pdfSecurity**
  - Operation: **unlockPdf**
  - `unlock_url`: `user@example.com` (placeholder; likely intended to be a file URL or accessible document URL)
  - `unlock_password`: `Smartmini@90` (hard-coded example password)
  - Uses credentials: **htmlcsstopdfApi** (“pdf munk - deepanshi”)
- **Inputs/outputs:**
  - **Input:** From Vendor Vault node.
  - **Output:** To **Intelligence Engine**.
- **Key variables/expressions:**
  - No expression usage; password is hard-coded instead of coming dynamically from the vault lookup.
- **Failure/edge cases:**
  - If `unlock_url` is not a publicly reachable URL (or not the correct parameter type), unlock will fail.
  - Wrong password → API error.
  - Large PDFs/timeouts/rate limits depending on provider plan.
  - Security risk: password hard-coded in node parameters (should be sourced from Sheets result or n8n credentials/environment variables).
- **Version-specific notes:** This is a community node integration; ensure it’s installed and compatible with your n8n version.

---

### Block 2 — Phase 2: Intelligence & Extraction
**Overview:**  
Performs vendor identification and currency conversion logic (currently a placeholder) and enriches the invoice data for later steps and alerting.

**Nodes involved:**
- Intelligence Engine

#### Node: Intelligence Engine
- **Type / role:** `n8n-nodes-base.code` — custom JavaScript transformation/enrichment.
- **Configuration (interpreted):**
  - Reads first incoming item: `const item = $input.first().json;`
  - Sets `item.vendorName = "Amazon Web Services";` (hard-coded example)
  - Sets `item.convertedAmount = item.amount * 1.09; // USD to EUR` (assumes `amount` exists)
  - Returns the modified object as a single item.
- **Inputs/outputs:**
  - **Input:** From **Unlock password protected PDF**.
  - **Output:** To **Drive: Fetch PO & Receipt**.
- **Key expressions/variables:**
  - `$input.first().json`
  - Relies on `item.amount` existing; not produced by prior nodes in this workflow as provided.
- **Failure/edge cases:**
  - If upstream does not provide `amount`, `convertedAmount` becomes `NaN`.
  - Vendor name is hard-coded; no real fuzzy match implemented.
  - If multiple items come in, only the first is processed (`first()`).
- **Version-specific notes:** Code node v2 uses the modern sandbox and `$input` helpers; older n8n versions differ.

---

### Block 3 — Phase 3: 3-Way Matching & Archival
**Overview:**  
Downloads PO/receipt from Google Drive, merges PDFs into one “audit bundle”, and creates a Bill in Xero. Despite the “3-way matching” label, there is no explicit reconciliation/verification logic node in the JSON.

**Nodes involved:**
- Drive: Fetch PO & Receipt
- Merge multiple PDFS into one
- Xero: Create Bill

#### Node: Drive: Fetch PO & Receipt
- **Type / role:** `n8n-nodes-base.googleDrive` — downloads supporting documents (PO and receipt).
- **Configuration (interpreted):**
  - Operation: **download**
  - File ID: `PO_FILE_ID` (placeholder)
- **Inputs/outputs:**
  - **Input:** From **Intelligence Engine**.
  - **Output:** To **Merge multiple PDFS into one**.
- **Key variables/expressions:** None shown; `fileId` is hard-coded placeholder.
- **Failure/edge cases:**
  - Google Drive credentials missing/expired.
  - File ID not found / permission denied.
  - Only one file is downloaded; if you need both PO and receipt, you typically need multiple downloads or a file search + loop.
- **Version-specific notes:** Google Drive node v3 may differ from v2 in binary handling and options.

#### Node: Merge multiple PDFS into one
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — PDF manipulation (merge).
- **Configuration (interpreted):**
  - Resource: **pdfManipulation**
  - Operation not explicitly shown in JSON (but implied “merge” by node name and sticky note)
  - Uses credentials: **htmlcsstopdfApi** (“pdf munk - deepanshi”)
- **Inputs/outputs:**
  - **Input:** From **Drive: Fetch PO & Receipt**.
  - **Output:** To **Xero: Create Bill**.
- **Key variables/expressions:** None shown.
- **Failure/edge cases:**
  - If inputs are not provided as binaries/URLs in the format the API expects, merge fails.
  - If only one PDF is provided, merge may either no-op or error depending on API.
  - Size limits and timeouts.
- **Version-specific notes:** Community node; validate correct operation selection in UI.

#### Node: Xero: Create Bill
- **Type / role:** `n8n-nodes-base.xero` — creates an Accounts Payable Bill in Xero.
- **Configuration (interpreted):**
  - Resource: **bill**
  - The required bill fields (Contact, Line Items, Amounts, Account Codes, etc.) are not configured in the JSON snippet.
- **Inputs/outputs:**
  - **Input:** From merged PDF node.
  - **Output:** To **Switch: Alert Channel**.
- **Key variables/expressions:** None shown, but typically you would map from `Intelligence Engine` output (vendorName, amounts, etc.).
- **Failure/edge cases:**
  - Missing required Xero fields → validation errors.
  - Xero OAuth connection expired.
  - Duplicate bill submission (idempotency not implemented).
- **Version-specific notes:** Xero node v1 uses Xero’s API with tenant selection; ensure tenant is configured.

---

### Block 4 — Phase 4: ERP Sync & Multi-Channel Alerting
**Overview:**  
Chooses an alert route based on an invoice “type” field. “Urgent” goes to Slack; “Standard” goes to Teams (via webhook HTTP request).

**Nodes involved:**
- Switch: Alert Channel
- Slack: High Value Alert
- Teams: Dept Head Notify

#### Node: Switch: Alert Channel
- **Type / role:** `n8n-nodes-base.switch` — branching logic.
- **Configuration (interpreted):**
  - `value1` expression: `={{ $node["Intelligence Engine"].json.type }}`
  - Data type: string
  - Rules:
    - If equals **Urgent** → output 0 (Slack path)
    - If equals **Standard** → output 1 (Teams path)
- **Inputs/outputs:**
  - **Input:** From **Xero: Create Bill** (note: it references “Intelligence Engine” by name, not the immediate input).
  - **Output 0:** To **Slack: High Value Alert**
  - **Output 1:** To **Teams: Dept Head Notify**
- **Key expressions/variables:**
  - `$node["Intelligence Engine"].json.type` must exist.
- **Failure/edge cases:**
  - If `type` is undefined, no rule matches → execution may end without notifications (unless a “fallback” output is configured; not shown).
  - Tight coupling to node name “Intelligence Engine”; renaming node breaks expression.
- **Version-specific notes:** Switch node v1 rules behavior is stable but ensure the UI matches the “rules” model.

#### Node: Slack: High Value Alert
- **Type / role:** `n8n-nodes-base.slack` — sends an alert message to Slack channel.
- **Configuration (interpreted):**
  - Posts text: `🔴 *Urgent Invoice Processed:* {{ $node["Intelligence Engine"].json.vendorName }}`
  - Target: channel selector
  - Channel ID: `FINANCE_URGENT` (placeholder)
- **Inputs/outputs:**
  - **Input:** From Switch output “Urgent”.
  - **Output:** None downstream.
- **Key expressions/variables:**
  - References `$node["Intelligence Engine"].json.vendorName`.
- **Failure/edge cases:**
  - Slack auth/token missing or insufficient scopes.
  - Channel ID invalid or bot not in channel.
  - If vendorName is missing, message is incomplete.
- **Version-specific notes:** Slack node v2.1 uses Slack API; scopes differ based on operation.

#### Node: Teams: Dept Head Notify
- **Type / role:** `n8n-nodes-base.httpRequest` — posts to a Microsoft Teams incoming webhook.
- **Configuration (interpreted):**
  - URL: `TEAMS_WEBHOOK_URL` (placeholder)
  - `sendBody: true`
  - Body parameters list contains an empty object (effectively no meaningful payload configured yet).
- **Inputs/outputs:**
  - **Input:** From Switch output “Standard”.
  - **Output:** None downstream.
- **Key expressions/variables:** None currently.
- **Failure/edge cases:**
  - Webhook URL invalid/rotated → 4xx.
  - Teams expects a JSON payload with specific structure (e.g., `{ "text": "..." }`); current body likely results in a malformed request.
- **Version-specific notes:** HTTP Request node v4.1 supports multiple auth/body modes; ensure JSON body is enabled and correct headers are set.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| MAIN_GLOBAL_INFO | Sticky Note | Global context/setup info |  |  | ## 🚀 Intelligent Invoice Hub: Advanced Finance Automation / Core Capabilities / Setup steps (connect Gmail, Sheets, Xero, Slack; set threshold in IF; map Xero fields) |
| Sticky_Ingestion | Sticky Note | Phase label/commentary |  |  | ## 🛡️ PHASE 1: Multi-Channel Ingestion & Security … HTML to PDF (Unlock) |
| Gmail: Watch Invoices | Gmail | Ingest invoices from Gmail |  | Vault: Get Decryption Key | ## 🛡️ PHASE 1: Multi-Channel Ingestion & Security … HTML to PDF (Unlock) |
| Vault: Get Decryption Key | Google Sheets | Lookup vendor decryption key | Gmail: Watch Invoices | Unlock password protected PDF | ## 🛡️ PHASE 1: Multi-Channel Ingestion & Security … HTML to PDF (Unlock) |
| Unlock password protected PDF | HTMLCSS to PDF (community) | Unlock encrypted PDF | Vault: Get Decryption Key | Intelligence Engine | ## 🛡️ PHASE 1: Multi-Channel Ingestion & Security … HTML to PDF (Unlock) |
| Sticky_Intelligence | Sticky Note | Phase label/commentary |  |  | ## 🤖 PHASE 2: Intelligence & Extraction … Parse PDF to JSON … currency conversion … budget codes |
| Intelligence Engine | Code | Vendor matching + currency conversion (placeholder) | Unlock password protected PDF | Drive: Fetch PO & Receipt | ## 🤖 PHASE 2: Intelligence & Extraction … Parse PDF to JSON … currency conversion … budget codes |
| Sticky_Reconciliation | Sticky Note | Phase label/commentary |  |  | ## ⚖️ PHASE 3: 3-Way Matching & Archival … HTML to PDF (Merge) … approve for ERP sync |
| Drive: Fetch PO & Receipt | Google Drive | Download PO/receipt document | Intelligence Engine | Merge multiple PDFS into one | ## ⚖️ PHASE 3: 3-Way Matching & Archival … HTML to PDF (Merge) … approve for ERP sync |
| Merge multiple PDFS into one | HTMLCSS to PDF (community) | Merge PDFs into audit bundle | Drive: Fetch PO & Receipt | Xero: Create Bill | ## ⚖️ PHASE 3: 3-Way Matching & Archival … HTML to PDF (Merge) … approve for ERP sync |
| Xero: Create Bill | Xero | Create bill in Xero | Merge multiple PDFS into one | Switch: Alert Channel | ## 📢 PHASE 4: ERP Sync & Multi-Channel Alerting … Xero … Slack … Teams … Gmail digest |
| Sticky_Alerting | Sticky Note | Phase label/commentary |  |  | ## 📢 PHASE 4: ERP Sync & Multi-Channel Alerting … Slack … Teams … Gmail digest |
| Switch: Alert Channel | Switch | Route to Slack vs Teams | Xero: Create Bill | Slack: High Value Alert; Teams: Dept Head Notify | ## 📢 PHASE 4: ERP Sync & Multi-Channel Alerting … Slack … Teams … Gmail digest |
| Slack: High Value Alert | Slack | Send urgent invoice alert | Switch: Alert Channel |  | ## 📢 PHASE 4: ERP Sync & Multi-Channel Alerting … Slack … Teams … Gmail digest |
| Teams: Dept Head Notify | HTTP Request | Post standard notification to Teams webhook | Switch: Alert Channel |  | ## 📢 PHASE 4: ERP Sync & Multi-Channel Alerting … Slack … Teams … Gmail digest |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Gmail trigger**
   - Node: **Gmail**
   - Name: `Gmail: Watch Invoices`
   - Configure the trigger event (e.g., new email) and apply filters (label, query like `has:attachment filename:pdf`, etc.).
   - **Credentials:** Gmail OAuth2 in n8n.
3. **Add Google Sheets lookup (“Vendor Vault”)**
   - Node: **Google Sheets**
   - Name: `Vault: Get Decryption Key`
   - Operation: **Lookup**
   - Document ID: set to your Vendor Vault spreadsheet (replace `VENDOR_VAULT_ID`)
   - Configure lookup column/value (e.g., match `from email domain` or `vendor name`) and return password/key fields.
   - **Credentials:** Google Sheets OAuth2/service account.
   - Connect: `Gmail: Watch Invoices` → `Vault: Get Decryption Key`.
4. **Add PDF unlock node (HTMLCSS to PDF community node)**
   - Node: **HTMLCSS to PDF** (community)
   - Name: `Unlock password protected PDF`
   - Resource: **PDF Security**
   - Operation: **Unlock PDF**
   - Configure:
     - Input PDF reference: replace `unlock_url` with a real accessible URL or map binary file (depending on node capability).
     - Password: ideally map from the Sheets lookup result (instead of hard-coded `Smartmini@90`).
   - **Credentials:** htmlcsstopdf API key.
   - Connect: `Vault: Get Decryption Key` → `Unlock password protected PDF`.
5. **Add Code node for enrichment**
   - Node: **Code**
   - Name: `Intelligence Engine`
   - Implement vendor detection and currency conversion. At minimum:
     - Ensure `amount` exists (or parse it beforehand).
     - Populate fields used later: `vendorName`, `type` (Urgent/Standard), amounts, etc.
   - Connect: `Unlock password protected PDF` → `Intelligence Engine`.
6. **Add Google Drive download**
   - Node: **Google Drive**
   - Name: `Drive: Fetch PO & Receipt`
   - Operation: **Download**
   - File ID: replace `PO_FILE_ID` (or implement search + loop to retrieve PO and receipt separately).
   - **Credentials:** Google Drive OAuth2/service account.
   - Connect: `Intelligence Engine` → `Drive: Fetch PO & Receipt`.
7. **Add PDF merge node (HTMLCSS to PDF community node)**
   - Node: **HTMLCSS to PDF**
   - Name: `Merge multiple PDFS into one`
   - Resource: **PDF Manipulation**
   - Configure merge inputs (invoice PDF + PO + receipt). This often requires providing multiple file URLs/binaries in the API’s expected format.
   - **Credentials:** htmlcsstopdf API key.
   - Connect: `Drive: Fetch PO & Receipt` → `Merge multiple PDFS into one`.
8. **Add Xero bill creation**
   - Node: **Xero**
   - Name: `Xero: Create Bill`
   - Resource: **Bill**
   - Map required fields (Contact, Dates, Line Items, Account Codes, Tax, Total, etc.) using data from `Intelligence Engine` and/or parsed invoice content.
   - **Credentials:** Xero OAuth2 + select tenant.
   - Connect: `Merge multiple PDFS into one` → `Xero: Create Bill`.
9. **Add Switch for alert routing**
   - Node: **Switch**
   - Name: `Switch: Alert Channel`
   - Value to evaluate (expression): `{{$node["Intelligence Engine"].json.type}}`
   - Rule 1: equals `Urgent` → Output 0
   - Rule 2: equals `Standard` → Output 1
   - Connect: `Xero: Create Bill` → `Switch: Alert Channel`.
10. **Add Slack alert**
    - Node: **Slack**
    - Name: `Slack: High Value Alert`
    - Operation: send message to channel
    - Channel: set to your finance urgent channel (replace `FINANCE_URGENT`)
    - Text: reference vendor/amount fields (e.g., `{{$node["Intelligence Engine"].json.vendorName}}`)
    - **Credentials:** Slack OAuth/token with proper scopes.
    - Connect: `Switch: Alert Channel` (Urgent output) → `Slack: High Value Alert`.
11. **Add Teams webhook notification**
    - Node: **HTTP Request**
    - Name: `Teams: Dept Head Notify`
    - Method: POST
    - URL: your Teams incoming webhook (replace `TEAMS_WEBHOOK_URL`)
    - Send JSON body, e.g. `{ "text": "Standard invoice processed: <vendor>" }`
    - Connect: `Switch: Alert Channel` (Standard output) → `Teams: Dept Head Notify`.
12. (Optional but implied by sticky notes) **Add missing controls**
    - Add an **IF** node for “high value threshold” before Slack routing (mentioned in global note but not present).
    - Add actual **PDF parsing** to extract amounts/line items (not present).
    - Add explicit **3-way match validation** step (not present).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Intelligent Invoice Hub: Advanced Finance Automation… Setup: Connect Gmail, Sheets (Vendor Vault), Xero, Slack; set ‘High Value’ threshold in the IF node; map Xero fields to Chart of Accounts.” | From global sticky note (MAIN_GLOBAL_INFO). Note: the referenced **IF node** is not included in this workflow JSON. |
| “PHASE 1… consolidates data from Gmail, IMAP, and Webhooks… fetches vendor-specific decryption keys… uses HTML to PDF (Unlock).” | From Sticky_Ingestion. Note: IMAP/Webhooks nodes are not present; only Gmail is implemented. |
| “PHASE 2… Fuzzy Matching… Parse PDF to JSON… currency conversion… early payment discounts… budget codes.” | From Sticky_Intelligence. Note: PDF parsing/budget mapping nodes are not present; only a placeholder Code node exists. |
| “PHASE 3… fetches POs and Receipts… HTML to PDF (Merge)… reconciliation confirms 3-way match.” | From Sticky_Reconciliation. Note: reconciliation/matching confirmation logic is not implemented in this JSON. |
| “PHASE 4… Creates bills in Xero… forwards alerts to Slack… monthly budget summaries to Teams… success digest via Gmail.” | From Sticky_Alerting. Note: there is no Gmail “digest” sender node and no monthly summary scheduler in this JSON. |