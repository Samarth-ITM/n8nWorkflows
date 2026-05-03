Manage enterprise contract approvals and audit PDFs with HubSpot, Google Drive and Slack

https://n8nworkflows.xyz/workflows/manage-enterprise-contract-approvals-and-audit-pdfs-with-hubspot--google-drive-and-slack-12659


# Manage enterprise contract approvals and audit PDFs with HubSpot, Google Drive and Slack

## 1. Workflow Overview

**Purpose:** Automate enterprise contract execution steps: determine approval depth from contract value, predict PDF compression needs, compress a contract PDF, validate size against delivery/provider limits, generate audit metadata, re-compress if needed, archive the audit PDF to Google Drive, update the HubSpot deal, and notify Finance via Slack.

**Primary use cases:**
- Standardizing contract governance for high-value deals (approval depth based on value tiers).
- Enforcing PDF size constraints (email/provider limits) via compression and validation.
- Producing an audit trail and archiving artifacts for compliance.
- Closing the loop in CRM (HubSpot) and internal notifications (Slack).

### 1.1 Logical Blocks
1. **Workflow Context & Documentation (Sticky notes)**
2. **Phase 1: Intake & Approval Routing**
3. **Phase 2: Assembly/Compression Planning**
4. **Phase 3: PDF Compression + Size Validation + Audit Metadata**
5. **Phase 4: Archival & Closure (Drive → HubSpot → Slack)**

> Important: The sticky notes describe additional systems (Salesforce trigger, Gmail/SMS/DocuSign, SendGrid, Postgres/Sheets) that are **not present as nodes** in this JSON. The implemented workflow starts at the first Code node and assumes upstream data and a PDF binary are already available.

---

## 2. Block-by-Block Analysis

### Block A — Workflow Context & Documentation
**Overview:** Provides human-readable phase descriptions and setup notes. No execution impact.  
**Nodes Involved:** `Sticky Note`, `Sticky_Approval`, `Sticky_Assembly`, `Sticky_Audit`, `Sticky_Closure`

#### Node: Sticky Note
- **Type / Role:** Sticky Note (documentation)
- **Configuration choices:** Describes intended end-to-end “Enterprise CLM” concept, core logic, quick setup, and key metrics.
- **Connections:** None (notes do not connect).
- **Edge cases:** None.

#### Node: Sticky_Approval
- **Type / Role:** Sticky Note
- **Configuration choices:** Mentions Salesforce trigger, ERP/Legal fetch, 2–6 approval levels, and manual approvals via webhook wait nodes.
- **Reality check:** No Salesforce/Webhook Wait nodes exist in the provided workflow.
- **Edge cases:** None.

#### Node: Sticky_Assembly
- **Type / Role:** Sticky Note
- **Configuration choices:** Mentions HTML assembly with pricing/inventory/charts and re-compression if PDF > 25MB.
- **Reality check:** HTML assembly nodes are not present; re-compression exists but is unconditional in the current graph (see Block D/E).

#### Node: Sticky_Audit
- **Type / Role:** Sticky Note
- **Configuration choices:** Mentions Gmail/SMS/DocuSign delivery, SendGrid fallback, Postgres/Sheets metrics logging, and generating compliance audit PDF.
- **Reality check:** Delivery/logging nodes are not present. Audit metadata generation exists.

#### Node: Sticky_Closure
- **Type / Role:** Sticky Note
- **Configuration choices:** Mentions uploading signed contract + audit PDF, updating CRM status, notifying Finance/Legal via Slack.
- **Reality check:** Drive upload + HubSpot update + Slack alert are present; “signed contract” handling is not implemented as a separate artifact.

---

### Block B — Phase 1: Intake & Approval Routing
**Overview:** Determines approval chain depth from `contractValue` and sets a board notification flag. This is the governance gate logic the rest of the workflow depends on.  
**Nodes Involved:** `Code: Approval Router`

#### Node: Code: Approval Router
- **Type / Role:** Code node (JavaScript) to compute routing metadata.
- **Configuration choices (interpreted):**
  - Reads the first incoming item (`$input.first().json`).
  - Uses `contractValue` to set:
    - `approvalLevels`: 2 / 4 / 6
    - `boardNotify`: boolean
  - Thresholds:
    - `> 500000` ⇒ 6 levels, board notify
    - `> 50000` ⇒ 4 levels
    - else ⇒ 2 levels
- **Key variables/fields produced:** `approvalLevels`, `boardNotify` (added to the existing JSON).
- **Input/Output connections:**
  - **Input:** (Not defined in JSON; this node has no incoming connection, so the workflow as provided has no explicit trigger.)
  - **Output:** to `Code: Complexity Engine`.
- **Version specifics:** TypeVersion **2** (standard n8n Code node).
- **Edge cases / failure types:**
  - If `contractValue` is missing or not numeric, comparisons may yield unexpected results (e.g., `undefined > 50000` is `false`) causing fallback to 2 levels.
  - If upstream supplies multiple items, only the first is used.
- **Sub-workflow reference:** None.

---

### Block C — Phase 2: Assembly/Compression Planning
**Overview:** Predicts compression ratio and size limits based on estimated pages and recipient domain, and chooses a delivery method hint.  
**Nodes Involved:** `Code: Complexity Engine`

#### Node: Code: Complexity Engine
- **Type / Role:** Code node (JavaScript) for PDF compression planning.
- **Configuration choices (interpreted):**
  - Reads:
    - `pageEstimate` (defaults to 10 if missing)
    - `clientEmail` (splits on `@` to infer domain)
  - Decision matrix:
    - If `pages > 30`:
      - `compressionRatio = 0.65`
      - `deliveryMethod = 'drive_link'`
    - Else if domain is `gmail.com`:
      - `compressionRatio = 0.85`
      - `limit = 25` (MB)
    - Else:
      - `compressionRatio = 0.90`
      - `limit = 20` (MB)
- **Key variables/fields produced:** `compressionRatio`, `deliveryMethod` (sometimes), `limit` (sometimes).
- **Input/Output connections:**
  - **Input:** from `Code: Approval Router`
  - **Output:** to `Compress PDF`
- **Version specifics:** TypeVersion **2**
- **Edge cases / failure types:**
  - If `clientEmail` is missing/invalid, `split('@')[1]` can be `undefined`, which may throw if `clientEmail` is not a string (e.g., null). This would fail the node.
  - `deliveryMethod` is only set for `pages > 30`; downstream nodes may assume it exists.
- **Sub-workflow reference:** None.

---

### Block D — Phase 3: PDF Compression + Size Validation
**Overview:** Compresses a PDF via the HTMLCSSToPDF “compressPdf” operation and checks whether the resulting binary exceeds the configured size limit.  
**Nodes Involved:** `Compress PDF`, `Code: Size Validator`

#### Node: Compress PDF
- **Type / Role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` (PDF manipulation) to compress a PDF binary.
- **Configuration choices (interpreted):**
  - Resource: `pdfManipulation`
  - Operation: `compressPdf`
  - Uses credential: **htmlcsstopdfApi** (“pdf munk - deepanshi”)
  - No explicit mapping shown for `compressionRatio` input; depending on node implementation, it may use defaults unless additional parameters are configured (not present here).
- **Input/Output connections:**
  - **Input:** from `Code: Complexity Engine`
  - **Output:** to `Code: Size Validator`
- **Version specifics:** TypeVersion **1** (this is a community/third-party node; behavior may differ by package version).
- **Edge cases / failure types:**
  - Missing/invalid API credentials.
  - If the incoming item does not contain an expected PDF binary field (commonly `binary.data`), compression will fail.
  - Provider/API size limits or timeouts for very large PDFs.

#### Node: Code: Size Validator
- **Type / Role:** Code node (JavaScript) to compute size and threshold flag.
- **Configuration choices (interpreted):**
  - Reads `binary.data.fileSize` and computes MB.
  - Reads limit from `json.limit` (defaults to 25 if missing).
  - Outputs **only**:
    - `sizeExceeded` (boolean)
    - `finalSize` (string like `"12.34MB"`)
- **Input/Output connections:**
  - **Input:** from `Compress PDF`
  - **Output:** to `Code: Audit Metadata`
- **Version specifics:** TypeVersion **2**
- **Edge cases / failure types:**
  - Assumes `binary.data` exists and has `fileSize`. If missing, node will throw (cannot read properties of undefined).
  - The computed result discards prior JSON fields unless merged upstream; downstream nodes expecting original fields may break.

> Design gap: Although it computes `sizeExceeded`, there is **no IF/Switch** node to branch into a “re-compress” path only when exceeded.

---

### Block E — Phase 3 (continued): Audit Metadata + Secondary Compression
**Overview:** Builds a textual audit log string using approval and delivery fields, then performs a second compression pass (currently always).  
**Nodes Involved:** `Code: Audit Metadata`, `Compress PDF1`

#### Node: Code: Audit Metadata
- **Type / Role:** Code node (JavaScript) generating an audit log snippet.
- **Configuration choices (interpreted):**
  - Appends `auditLog` string containing:
    - `approvers.join(', ')`
    - generation timestamp
    - `finalSize`
    - `deliveryTimestamp`
- **Input/Output connections:**
  - **Input:** from `Code: Size Validator`
  - **Output:** to `Compress PDF1`
- **Version specifics:** TypeVersion **2**
- **Edge cases / failure types:**
  - **High risk of runtime failure:** `item.approvers.join(...)` will throw if `approvers` is missing or not an array.
  - References `item.deliveryTimestamp` which is never created in this workflow; will become `undefined` in the log unless provided upstream.
  - References `item.finalSize`—but `finalSize` is produced by `Code: Size Validator`; however, that node outputs a new JSON object and does not preserve earlier fields, meaning `approvers` is likely lost unless it was reattached upstream.
  - Slack later references `item.fingerprint`, but this node does not set `fingerprint`.
- **Sub-workflow reference:** None.

#### Node: Compress PDF1
- **Type / Role:** HTMLCSSToPDF node compressing again (secondary pass).
- **Configuration choices (interpreted):**
  - Same resource/operation as first compression: `pdfManipulation` → `compressPdf`
  - Same credential: **htmlcsstopdfApi** (“pdf munk - deepanshi”)
  - No conditional logic; it always runs.
- **Input/Output connections:**
  - **Input:** from `Code: Audit Metadata`
  - **Output:** to `Drive: Archive Audit PDF`
- **Version specifics:** TypeVersion **1**
- **Edge cases / failure types:**
  - If the input item lacks binary PDF data at this point (possible due to code node reshaping), compression fails.
  - Unnecessary extra compression can degrade quality.

---

### Block F — Phase 4: Archival & Closure (Drive → HubSpot → Slack)
**Overview:** Uploads the PDF to Google Drive, updates a HubSpot deal, then posts a Slack message to Finance.  
**Nodes Involved:** `Drive: Archive Audit PDF`, `Update a deal`, `Slack: Finance Alert`

#### Node: Drive: Archive Audit PDF
- **Type / Role:** Google Drive node to upload/store the (audit) PDF.
- **Configuration choices (interpreted):**
  - Drive: “My Drive”
  - Folder: `root` (top-level)
  - Operation is not explicitly visible in parameters snippet; by node type it is likely **Upload** or **Create** with binary input, but the JSON shown only includes drive/folder selection.
  - Credential: **googleDriveOAuth2Api** (“PDFMunk - Jitesh”)
- **Input/Output connections:**
  - **Input:** from `Compress PDF1`
  - **Output:** to `Update a deal`
- **Version specifics:** TypeVersion **3**
- **Edge cases / failure types:**
  - OAuth token expiration/insufficient permissions.
  - Missing binary data to upload.
  - Uploading to root can cause clutter; may require folder routing logic.

#### Node: Update a deal
- **Type / Role:** HubSpot node to update a deal record.
- **Configuration choices (interpreted):**
  - Resource: `deal`
  - Operation: `update`
  - Authentication: `appToken` (HubSpot private app token)
  - `dealId` is empty in the provided JSON (must be set for successful updates).
  - `updateFields` is empty (no properties are actually updated unless added).
- **Input/Output connections:**
  - **Input:** from `Drive: Archive Audit PDF`
  - **Output:** to `Slack: Finance Alert`
- **Version specifics:** TypeVersion **2.2**
- **Edge cases / failure types:**
  - Will fail if `dealId` is not provided (as currently configured).
  - Will do nothing meaningful if `updateFields` remains empty.
  - HubSpot rate limits or invalid app token.

#### Node: Slack: Finance Alert
- **Type / Role:** Slack node sending a message to a channel.
- **Configuration choices (interpreted):**
  - Authentication: OAuth2 credential “Mediajade Slack”
  - Posts to channel `FINANCE_DEPT` (selected via `channelId`)
  - Message uses n8n expressions referencing earlier nodes:
    - `clientName` and `contractValue` from `Code: Approval Router`
    - `fingerprint` from `Code: Audit Metadata` (but fingerprint is never set)
- **Input/Output connections:**
  - **Input:** from `Update a deal`
  - **Output:** none (end)
- **Version specifics:** TypeVersion **2.1**
- **Edge cases / failure types:**
  - If `fingerprint` is undefined, Slack message will show blank or “undefined”.
  - Slack OAuth scopes may be missing (e.g., `chat:write`).
  - Expression evaluation failures if referenced node outputs are missing due to upstream failures.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | High-level workflow description and setup notes |  |  | ## Enterprise CLM: AI-Optimized Assembly, Multi-Stage Approvals & Audit Governance… (includes “Quick Setup” and “Key Metrics”) |
| Sticky_Approval | Sticky Note | Phase annotation (approval gating) |  |  | ## 🛡️ PHASE 1: Intake & Approval Gating… |
| Code: Approval Router | Code | Determine approval levels and board notification flag | *(none / missing trigger)* | Code: Complexity Engine | ## 🛡️ PHASE 1: Intake & Approval Gating… |
| Sticky_Assembly | Sticky Note | Phase annotation (dynamic assembly & compression) |  |  | ## 🏗️ PHASE 2: Dynamic Assembly & Compression… |
| Code: Complexity Engine | Code | Predict compression ratio, delivery method, size limit | Code: Approval Router | Compress PDF | ## 🏗️ PHASE 2: Dynamic Assembly & Compression… |
| Compress PDF | HTMLCSSToPDF | Compress the contract PDF | Code: Complexity Engine | Code: Size Validator | ## 🏗️ PHASE 2: Dynamic Assembly & Compression… |
| Code: Size Validator | Code | Compute final size and “exceeded limit” flag | Compress PDF | Code: Audit Metadata | ## 🚀 PHASE 3: Delivery & Audit Archival… |
| Sticky_Audit | Sticky Note | Phase annotation (delivery & audit archival) |  |  | ## 🚀 PHASE 3: Delivery & Audit Archival… |
| Code: Audit Metadata | Code | Build audit log text (approvers, timestamp, size, proof) | Code: Size Validator | Compress PDF1 | ## 🚀 PHASE 3: Delivery & Audit Archival… |
| Compress PDF1 | HTMLCSSToPDF | Secondary compression pass (always executed) | Code: Audit Metadata | Drive: Archive Audit PDF | ## 🏁 PHASE 4: Archival & Closure… |
| Drive: Archive Audit PDF | Google Drive | Upload/archive the PDF to Drive | Compress PDF1 | Update a deal | ## 🏁 PHASE 4: Archival & Closure… |
| Update a deal | HubSpot | Update CRM deal status/fields | Drive: Archive Audit PDF | Slack: Finance Alert | ## 🏁 PHASE 4: Archival & Closure… |
| Slack: Finance Alert | Slack | Notify Finance channel that contract is executed | Update a deal |  | ## 🏁 PHASE 4: Archival & Closure… |
| Sticky_Closure | Sticky Note | Phase annotation (archival & closure) |  |  | ## 🏁 PHASE 4: Archival & Closure… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add documentation sticky notes** (optional but recommended):
   - Add 5 Sticky Note nodes named:
     - `Sticky Note` (place general description + quick setup + key metrics)
     - `Sticky_Approval`, `Sticky_Assembly`, `Sticky_Audit`, `Sticky_Closure` (place phase descriptions)
3. **Add node: `Code: Approval Router`**
   - Node type: **Code**
   - Paste logic to compute `approvalLevels` and `boardNotify` from `contractValue`.
   - Ensure your incoming JSON includes at least: `contractValue`, `clientName` (used later in Slack).
4. **Add node: `Code: Complexity Engine`**
   - Node type: **Code**
   - Configure to read `pageEstimate` and `clientEmail` and output:
     - `compressionRatio`
     - `limit` (MB threshold)
     - optionally `deliveryMethod`
   - Connect: `Code: Approval Router` → `Code: Complexity Engine`.
5. **Add node: `Compress PDF`**
   - Node type: **HTMLCSSToPDF** (community node `n8n-nodes-htmlcsstopdf`)
   - Operation: `pdfManipulation` → `compressPdf`
   - Credentials: create/select **htmlcsstopdfApi** credential (API key for your provider).
   - Connect: `Code: Complexity Engine` → `Compress PDF`.
   - Upstream requirement: the incoming item must carry the PDF as **binary** (commonly `binary.data`). If your source produces base64/string, add a conversion step before this node.
6. **Add node: `Code: Size Validator`**
   - Node type: **Code**
   - Compute `sizeExceeded` and `finalSize` based on `binary.data.fileSize` and `json.limit`.
   - Connect: `Compress PDF` → `Code: Size Validator`.
7. **Add node: `Code: Audit Metadata`**
   - Node type: **Code**
   - Build an `auditLog` string.
   - Critical: ensure the item has:
     - `approvers` as an array (or add defensive code/defaults)
     - `deliveryTimestamp` (or expect it to be optional)
     - `finalSize` from prior step
   - Connect: `Code: Size Validator` → `Code: Audit Metadata`.
8. **Add node: `Compress PDF1`**
   - Node type: **HTMLCSSToPDF**
   - Operation: `pdfManipulation` → `compressPdf`
   - Credentials: same **htmlcsstopdfApi**
   - Connect: `Code: Audit Metadata` → `Compress PDF1`.
   - (If you want the behavior described in sticky notes, insert an **IF** node before this step using `sizeExceeded` to decide whether to re-compress.)
9. **Add node: `Drive: Archive Audit PDF`**
   - Node type: **Google Drive**
   - Authenticate with **Google Drive OAuth2** credential.
   - Select Drive: “My Drive”
   - Select Folder: `/ (Root folder)` or a dedicated compliance folder.
   - Configure the operation to **upload** the incoming binary PDF (ensure binary property name matches what the node expects).
   - Connect: `Compress PDF1` → `Drive: Archive Audit PDF`.
10. **Add node: `Update a deal`**
   - Node type: **HubSpot**
   - Authentication: **Private App Token** (HubSpot App Token credential).
   - Resource: `deal`, Operation: `update`
   - Set **Deal ID** (map from an upstream field like `dealId`, or choose manually for testing).
   - Populate `updateFields` (e.g., set stage to “Executed”, store Drive file link/id, store `finalSize`, etc.).
   - Connect: `Drive: Archive Audit PDF` → `Update a deal`.
11. **Add node: `Slack: Finance Alert`**
   - Node type: **Slack**
   - Authentication: OAuth2 (Slack app with `chat:write`)
   - Select channel: `FINANCE_DEPT`
   - Message text expressions:
     - `clientName` and `contractValue` from `Code: Approval Router`
     - Adjust/remove `fingerprint` unless you implement it (currently not generated).
   - Connect: `Update a deal` → `Slack: Finance Alert`.
12. **Add an actual trigger (required to run)**
   - The provided workflow has no trigger node. Add one based on your intake source, e.g.:
     - HubSpot Trigger (deal updated), Salesforce Trigger, Webhook, or Manual Trigger for testing.
   - Ensure the trigger outputs:
     - `contractValue`, `clientName`, `clientEmail`, `pageEstimate`
     - `approvers` array and `deliveryTimestamp` (or adapt audit code)
     - A PDF binary in `binary.data` before the first compression node (or add a PDF-generation node prior to compression).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Connect: HubSpot/Salesforce, Drive, Gmail, and Slack.” | Mentioned in the general sticky note; Salesforce/Gmail are not implemented as nodes in this JSON. |
| “Rules: Link Airtable/Sheets for approval levels and folder IDs.” | Suggested configuration source; Airtable/Sheets nodes are not present. |
| “Vault: Store encryption passwords in a secure Sheet or Env Variable.” | Suggested security practice; no encryption node exists here. |
| Key Metrics: `ContractValue`, `AuditHash`, `CompressionRatio` | `AuditHash` is referenced conceptually; no hash/fingerprint is actually generated in nodes (Slack references `fingerprint` but it is not set). |
| Re-compression when PDF exceeds provider limits | The workflow computes `sizeExceeded` but does not branch; add an IF node to match the described behavior. |