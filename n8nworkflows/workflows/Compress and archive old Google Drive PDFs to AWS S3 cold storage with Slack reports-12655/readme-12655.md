Compress and archive old Google Drive PDFs to AWS S3 cold storage with Slack reports

https://n8nworkflows.xyz/workflows/compress-and-archive-old-google-drive-pdfs-to-aws-s3-cold-storage-with-slack-reports-12655


# Compress and archive old Google Drive PDFs to AWS S3 cold storage with Slack reports

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Compress and archive old Google Drive PDFs to AWS S3 cold storage with Slack reports

**Purpose:**  
This workflow runs weekly to scan a Google Drive folder, identify files older than 6 months (based on `modifiedTime`), download eligible files, compress them, upload them to an AWS S3 “cold storage” bucket, and send a Slack report confirming the migration.

**Target use cases:**
- Reduce Google Drive “hot storage” usage by compressing and offloading old PDFs.
- Maintain an archive structure by year/month.
- Provide weekly audit-style notifications to an IT/ops channel or user in Slack.

### Logical Blocks
**1.1 Scheduled Trigger & File Listing**
- Runs weekly and lists files from Google Drive (folder list operation).

**1.2 Age Gate & Archive Path Preparation**
- Computes whether each file is older than 6 months and builds a year/month archive path.

**1.3 Conditional Archive Execution**
- Only proceeds when the file meets the archive criteria.

**1.4 Download → Compress → Upload**
- Downloads the file binary from Drive, compresses it via an external PDF compression node, uploads to S3.

**1.5 Slack Reporting**
- Sends a message with the migrated file name and computed archive path.

---

## 2. Block-by-Block Analysis

### 1.1 Scheduled Trigger & File Listing
**Overview:** Starts the workflow weekly and retrieves a list of items from Google Drive using the Google Drive node.  
**Nodes involved:**  
- Trigger: Weekly Archive Scan  
- List Active Project Files

#### Node: Trigger: Weekly Archive Scan
- **Type / role:** Schedule Trigger — workflow entry point on a timed schedule.
- **Configuration choices:** Runs on an interval of **every 1 week**.
- **Inputs/outputs:** No input; outputs one trigger event into “List Active Project Files”.
- **Version notes:** `typeVersion 1.1` (standard schedule trigger behavior).
- **Edge cases / failures:**
  - n8n instance timezone affects when “weekly” fires.
  - If workflow is inactive, it will not run.

#### Node: List Active Project Files
- **Type / role:** Google Drive node — lists folder contents / folder resources.
- **Configuration choices:** `resource: folder`, `operation: list`.
  - Note: As configured, it does not explicitly specify a folder ID in parameters, so it will rely on node defaults/UI selection (or may list accessible folders depending on n8n node behavior and additional options).
- **Credentials:** Google Drive OAuth2 (`jitesh0dugar`).
- **Inputs/outputs:** Input from schedule trigger; outputs listed folder items to the Code node.
- **Version notes:** `typeVersion 3`.
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent.
  - Insufficient Drive permissions.
  - Listing may return folders (not only PDFs). The rest of the workflow assumes `modifiedTime` exists and later steps assume a downloadable file.

---

### 1.2 Age Gate & Archive Path Preparation
**Overview:** Adds two computed fields to each item: a boolean flag indicating archival eligibility and an archive path grouped by year/month.  
**Nodes involved:**  
- Calculate File Age (Luxon Logic)

#### Node: Calculate File Age (Luxon Logic)
- **Type / role:** Code node — transforms item JSON and adds derived fields.
- **Configuration choices (interpreted):**
  - Reads the **first input item only** (`$input.first().json`).
  - Parses `modifiedTime` into a JS `Date`.
  - Calculates a date threshold: “today minus 6 months”.
  - Sets:
    - `item.isReadyForArchive = (modifiedDate < sixMonthsAgo)`
    - `item.archivePath = archives/<YYYY>/<M>` using current year and month.
- **Key variables/expressions:**
  - `item.modifiedTime` (must exist and be parseable)
  - `$input.first()` means multi-item listing may not be processed as intended.
- **Inputs/outputs:** Input from “List Active Project Files”; output to IF node.
- **Version notes:** `typeVersion 2`.
- **Edge cases / failures:**
  - **Major logic issue:** Using `$input.first()` likely causes only the first listed file to be evaluated and processed. If Drive returns multiple items, the rest will be ignored unless the node is rewritten to iterate over all items (e.g., `return $input.all().map(...)`).
  - If `modifiedTime` is missing or invalid, `new Date(item.modifiedTime)` becomes `Invalid Date`, producing incorrect comparisons.
  - The node name references “Luxon” but the implementation uses native `Date`, not Luxon.

---

### 1.3 Conditional Archive Execution
**Overview:** Branches execution so only eligible (older than 6 months) files proceed to download/compress/upload.  
**Nodes involved:**  
- IF: Meets Archive Criteria?

#### Node: IF: Meets Archive Criteria?
- **Type / role:** IF node — boolean gate.
- **Configuration choices:**
  - Condition checks `={{ $json.isReadyForArchive }}` as a boolean.
- **Inputs/outputs:**
  - Input from “Calculate File Age (Luxon Logic)”.
  - **True branch (output 0):** connected to “Download Hot Storage Binary”.
  - **False branch:** not connected (no action taken for non-eligible files).
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:**
  - If `isReadyForArchive` is `undefined` or a string, boolean evaluation may behave unexpectedly.
  - Non-eligible files are silently ignored (no reporting).

---

### 1.4 Download → Compress → Upload
**Overview:** Retrieves the file content from Drive, compresses the PDF, then uploads the resulting binary to an S3 bucket.  
**Nodes involved:**  
- Download Hot Storage Binary  
- Compress PDF for Archival  
- Upload to Cold Storage (S3)

#### Node: Download Hot Storage Binary
- **Type / role:** Google Drive node — downloads a file as binary.
- **Configuration choices:**
  - `operation: download`
  - `fileId: ={{ $json.id }}` from the Drive listing output.
- **Credentials:** Google Drive OAuth2 (`jitesh0dugar`).
- **Inputs/outputs:** Input from IF true path; outputs binary to the compression node.
- **Version notes:** `typeVersion 3`.
- **Edge cases / failures:**
  - If the listed item is a folder or Google Doc (non-binary Drive native), download may fail or require export logic.
  - Missing `id` field will break the expression.
  - Large files can cause memory/timeouts depending on n8n execution settings.

#### Node: Compress PDF for Archival
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — PDF manipulation (compress).
- **Configuration choices:**
  - `resource: pdfManipulation`
  - `operation: compress`
  - Expects incoming binary PDF content from previous node.
- **Credentials:** htmlcsstopdf API (`pdf munk - deepanshi`).
- **Inputs/outputs:** Input binary from download; output compressed binary to S3 upload.
- **Version notes:** `typeVersion 1` (custom/community node behavior may differ by install).
- **Edge cases / failures:**
  - If input binary is not a PDF, compression may fail.
  - API rate limits, auth failures, payload size limits.
  - Output binary property name may differ depending on node implementation (important for S3 upload mapping).

#### Node: Upload to Cold Storage (S3)
- **Type / role:** S3 node — uploads binary to AWS S3.
- **Configuration choices:**
  - `bucketName: project-archives-cold`
  - `fileKey: "2"` (static)
    - This is likely incorrect for real archival; it will overwrite the same object key repeatedly unless S3 node interprets `fileKey` differently in this version/config. Typically you’d use a dynamic key like `={{$json.archivePath}}/{{$json.name}}`.
- **Credentials:** S3 (`deepanshi`).
- **Inputs/outputs:** Input from compression node; output to Slack node.
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:**
  - Invalid AWS credentials, missing IAM permissions (`s3:PutObject`).
  - Wrong region/bucket policy restrictions.
  - Static `fileKey` causes overwrites and defeats archiving.
  - If node expects a specific binary property name and it doesn’t match, upload may fail.

---

### 1.5 Slack Reporting
**Overview:** Posts a message confirming that a file was compressed and migrated, including the file name and archive path.  
**Nodes involved:**  
- Log Savings to Slack

#### Node: Log Savings to Slack
- **Type / role:** Slack node — sends a Slack message using OAuth2.
- **Configuration choices:**
  - Message text uses expressions referencing other nodes:
    - File name: `{{ $node["List Active Project Files"].json.name }}`
    - Archive path: `{{ $node["Calculate File Age (Luxon Logic)"].json.archivePath }}`
  - Authentication: OAuth2 (`Mediajade Slack`)
  - Target: “user” selection mode is enabled, but `value` is empty — meaning the actual destination may be misconfigured (it might fail or default depending on node behavior).
- **Inputs/outputs:** Input from S3 upload; no downstream nodes.
- **Version notes:** `typeVersion 2.1`.
- **Edge cases / failures:**
  - If Slack destination (channel/user) is not set, the node may error.
  - Expressions referencing `List Active Project Files`.json.name may not match the actually archived item if multiple items are processed (and currently only first item is processed anyway).
  - Slack OAuth token revoked/expired.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: Weekly Archive Scan | n8n-nodes-base.scheduleTrigger | Weekly entry point | — | List Active Project Files | ## How it works; This workflow manages document lifecycles... (Google Drive setup, threshold, compression API, S3, Slack auth) |
| List Active Project Files | n8n-nodes-base.googleDrive | List Drive folders/files to evaluate | Trigger: Weekly Archive Scan | Calculate File Age (Luxon Logic) | ## How it works; This workflow manages document lifecycles... |
| Calculate File Age (Luxon Logic) | n8n-nodes-base.code | Compute age gate + archive path | List Active Project Files | IF: Meets Archive Criteria? | ## How it works; This workflow manages document lifecycles... |
| IF: Meets Archive Criteria? | n8n-nodes-base.if | Proceed only if older than threshold | Calculate File Age (Luxon Logic) | Download Hot Storage Binary | ## How it works; This workflow manages document lifecycles... |
| Download Hot Storage Binary | n8n-nodes-base.googleDrive | Download eligible file as binary | IF: Meets Archive Criteria? | Compress PDF for Archival | ### 🛠️ Optimization & Migration Logic; Grouped nodes for data compression, integrity verification, and S3 cloud migration. |
| Compress PDF for Archival | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Compress PDF payload | Download Hot Storage Binary | Upload to Cold Storage (S3) | ### 🛠️ Optimization & Migration Logic; Grouped nodes for data compression, integrity verification, and S3 cloud migration. |
| Upload to Cold Storage (S3) | n8n-nodes-base.s3 | Upload compressed PDF to S3 bucket | Compress PDF for Archival | Log Savings to Slack | ### 🛠️ Optimization & Migration Logic; Grouped nodes for data compression, integrity verification, and S3 cloud migration. |
| Log Savings to Slack | n8n-nodes-base.slack | Notify team/user in Slack | Upload to Cold Storage (S3) | — | ### 🛠️ Optimization & Migration Logic; Grouped nodes for data compression, integrity verification, and S3 cloud migration. |
| Main Documentation | n8n-nodes-base.stickyNote | Documentation / operator notes | — | — | ## How it works… + ## Setup steps… |
| Processing Overview | n8n-nodes-base.stickyNote | Visual grouping note | — | — | ### 🛠️ Optimization & Migration Logic… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **“Compress and archive old Google Drive PDFs to AWS S3 cold storage with Slack reports”**

2. **Add node: “Trigger: Weekly Archive Scan”**
   - Node type: **Schedule Trigger**
   - Set schedule to **Interval → Every 1 week**.

3. **Add node: “List Active Project Files”**
   - Node type: **Google Drive**
   - Credentials: connect **Google Drive OAuth2** (select the account that owns/has access to the “hot storage” location).
   - Operation: **Folder → List**
   - In the UI, select the target folder to scan (ensure the node is listing the files you intend; depending on n8n UI, you may need to specify folder ID or options to list folder contents).

4. **Connect:** Trigger → List Active Project Files

5. **Add node: “Calculate File Age (Luxon Logic)”**
   - Node type: **Code**
   - Paste logic equivalent to:
     - Read item JSON
     - Compare `modifiedTime` to now-minus-6-months
     - Add:
       - `isReadyForArchive` boolean
       - `archivePath` like `archives/YYYY/M`
   - Important: If you want to process *all* listed items, implement iteration across all input items (the provided workflow uses only the first item).

6. **Connect:** List Active Project Files → Calculate File Age (Luxon Logic)

7. **Add node: “IF: Meets Archive Criteria?”**
   - Node type: **IF**
   - Condition: Boolean
   - `value1` expression: `{{ $json.isReadyForArchive }}`

8. **Connect:** Calculate File Age → IF

9. **Add node: “Download Hot Storage Binary”**
   - Node type: **Google Drive**
   - Credentials: same Google Drive OAuth2
   - Operation: **File → Download**
   - File ID: `{{ $json.id }}`

10. **Connect:** IF (true output) → Download Hot Storage Binary

11. **Add node: “Compress PDF for Archival”**
   - Node type: **HTMLCSS to PDF** (community/custom node: `n8n-nodes-htmlcsstopdf`)
   - Credentials: configure htmlcsstopdf API credential (API key/token).
   - Resource: **PDF Manipulation**
   - Operation: **Compress**
   - Ensure the node is configured to consume the incoming binary property from the Google Drive download.

12. **Connect:** Download Hot Storage Binary → Compress PDF for Archival

13. **Add node: “Upload to Cold Storage (S3)”**
   - Node type: **S3**
   - Credentials: configure AWS S3 access key/secret (or IAM role, depending on your n8n setup).
   - Bucket name: `project-archives-cold`
   - Object key / file key:
     - The provided workflow uses a static `"2"`; for a real rebuild, set a dynamic key (recommended) such as:
       - `{{ $json.archivePath }}/{{ $json.name }}`
   - Ensure the node uploads the compressed binary output.

14. **Connect:** Compress PDF for Archival → Upload to Cold Storage (S3)

15. **Add node: “Log Savings to Slack”**
   - Node type: **Slack**
   - Credentials: Slack OAuth2
   - Choose destination (channel or user) explicitly in the node UI (the provided workflow leaves user value empty, which may not work).
   - Message text (adapt from workflow):
     - Include file name and archive path using expressions:
       - `{{ $node["List Active Project Files"].json.name }}`
       - `{{ $node["Calculate File Age (Luxon Logic)"].json.archivePath }}`

16. **Connect:** Upload to Cold Storage (S3) → Log Savings to Slack

17. **(Optional) Add sticky notes** for operator context:
   - “Main Documentation” with setup steps.
   - “Processing Overview” to label the compression/migration block.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow manages document lifecycles… triggers weekly… scans Google Drive… files modified over 6 months ago… downloaded, compressed… uploaded to AWS S3… Slack report…” | Sticky note: **Main Documentation** |
| Setup steps: Google Drive connect + choose “Hot Storage” folder; adjust threshold; ensure compression API credentials; configure S3 bucket/credentials; authenticate Slack reports | Sticky note: **Main Documentation** |
| “### 🛠️ Optimization & Migration Logic — Grouped nodes for data compression, integrity verification, and S3 cloud migration.” | Sticky note: **Processing Overview** |