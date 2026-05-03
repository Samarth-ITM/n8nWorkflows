Consolidate and report monthly financial PDFs with Google Drive and Slack

https://n8nworkflows.xyz/workflows/consolidate-and-report-monthly-financial-pdfs-with-google-drive-and-slack-12544


# Consolidate and report monthly financial PDFs with Google Drive and Slack

## 1. Workflow Overview

**Purpose:**  
This workflow runs once per month to collect financial PDF files from a Google Drive folder, validate they are PDFs, download them as binaries, merge them into a single consolidated “Master Report” PDF using an external PDF manipulation service (HTML/CSS to PDF), upload the merged report back to Google Drive, and notify the finance team via Slack with a month-stamped message.

**Primary use cases:**
- Monthly finance close: consolidate statements/invoices/reports into one bundle
- Automated, scheduled reporting with traceable output in Drive + team notification in Slack

### 1.1 Scheduling / Entry
Runs on a monthly schedule (intended “on the 1st”, though the current schedule config is “every month” and may need a day-of-month specification depending on desired behavior).

### 1.2 Drive Discovery + Validation
Lists files in a Drive folder, then filters to PDFs only (by MIME type) to avoid merge failures.

### 1.3 Processing & Consolidation
Downloads validated PDFs (binary), aggregates them into a batch payload, and merges them into one PDF.

### 1.4 Storage + Notification
Uploads the consolidated PDF to Drive, then posts a Slack notification including the report month and merged file name.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling / Entry

**Overview:**  
Starts the workflow on a repeating monthly cadence.

**Nodes involved:**
- Trigger: Monthly on the 1st1

#### Node: Trigger: Monthly on the 1st1
- **Type / role:** Schedule Trigger — workflow entry point.
- **Configuration (interpreted):** Runs on an interval of **months**. (No explicit day-of-month is shown in the provided configuration.)
- **Key variables/expressions:** None.
- **Connections:**
  - **Output →** List Financial PDFs from Drive1
- **Version notes:** `typeVersion 1.1` (n8n Schedule Trigger behavior can vary slightly across versions; confirm UI options match).
- **Edge cases / failures:**
  - Misconfiguration: if you truly need “1st of every month”, ensure the trigger rule includes **day = 1** (and optionally time + timezone), otherwise it may run monthly relative to activation time/settings.

---

### Block 2 — Drive Discovery + Validation

**Overview:**  
Fetches a list of items from a Google Drive folder and filters out anything that is not a PDF based on MIME type.

**Nodes involved:**
- List Financial PDFs from Drive1
- Validate File Format (PDF only)1

#### Node: List Financial PDFs from Drive1
- **Type / role:** Google Drive node — lists folder contents.
- **Configuration (interpreted):**
  - **Resource:** Folder
  - **Operation:** List
  - (Folder selection details are not included in the exported parameters snippet; you must select the target folder in the node UI.)
- **Credentials:** Google Drive OAuth2 (`jitesh0dugar`)
- **Key variables/expressions:** None.
- **Connections:**
  - **Input ←** Trigger: Monthly on the 1st1
  - **Output →** Validate File Format (PDF only)1
- **Version notes:** `typeVersion 3`
- **Edge cases / failures:**
  - OAuth permission issues (missing scopes, revoked consent)
  - Folder not selected / wrong folder → empty results
  - Large folders may require pagination/limits (depending on node settings)

#### Node: Validate File Format (PDF only)1
- **Type / role:** IF node — filters items to PDFs only.
- **Configuration (interpreted):**
  - Condition: `mimeType` must equal `application/pdf`
  - Expression: `{{ $json.mimeType }} == "application/pdf"`
- **Connections:**
  - **Input ←** List Financial PDFs from Drive1
  - **True output →** Download Documents for Processing1
  - **False output:** not connected (non-PDFs are dropped)
- **Version notes:** `typeVersion 1`
- **Edge cases / failures:**
  - Some Drive files may have unexpected MIME types (e.g., Google Docs PDFs, shortcuts, or exported files)
  - If Drive list output doesn’t include `mimeType` for some reason, the expression can evaluate unexpectedly and route to “false” (dropping valid files)

---

### Block 3 — Processing & Consolidation

**Overview:**  
Downloads each validated PDF as binary data, aggregates all binaries into a single payload, and merges them into one PDF using the HTML/CSS to PDF service.

**Nodes involved:**
- Download Documents for Processing1
- Batch Files for Merging1
- Merge multiple PDFS into one

#### Node: Download Documents for Processing1
- **Type / role:** Google Drive node — downloads a file (binary).
- **Configuration (interpreted):**
  - **Operation:** Download
  - **File ID:** from current item: `{{ $json.id }}`
- **Credentials:** Google Drive OAuth2 (`jitesh0dugar`)
- **Connections:**
  - **Input ←** Validate File Format (PDF only)1 (true branch)
  - **Output →** Batch Files for Merging1
- **Version notes:** `typeVersion 3`
- **Edge cases / failures:**
  - File permissions / access denied
  - Download size/timeouts for large PDFs
  - Binary property naming: Google Drive download typically outputs a binary property (often `data`). Downstream nodes assume a binary field named `data` exists.

#### Node: Batch Files for Merging1
- **Type / role:** Aggregate node — collects multiple items into one item containing an array of binaries for merging.
- **Configuration (interpreted):**
  - Aggregates the field `data` (binary) across incoming items into a single aggregated structure.
  - No extra options configured.
- **Connections:**
  - **Input ←** Download Documents for Processing1
  - **Output →** Merge multiple PDFS into one
- **Version notes:** `typeVersion 1`
- **Edge cases / failures:**
  - If the incoming binary field is not named `data`, aggregation will produce an empty/invalid payload.
  - If zero PDFs pass the IF filter, this node may receive no items; merge will fail or produce no output (consider adding a guard/IF for “no files found”).

#### Node: Merge multiple PDFS into one
- **Type / role:** HTML/CSS to PDF node (`n8n-nodes-htmlcsstopdf.htmlcsstopdf`) — performs PDF manipulation (merge).
- **Configuration (interpreted):**
  - **Resource:** PDF Manipulation
  - (Specific merge mapping isn’t visible in the snippet; logically it consumes the aggregated binary list produced by the Aggregate node.)
- **Credentials:** htmlcsstopdf API (`pdf munk - deepanshi`)
- **Connections:**
  - **Input ←** Batch Files for Merging1
  - **Output →** Upload Consolidated Master Report1
- **Version notes:** `typeVersion 1` (community/third-party node; behavior depends on node package version).
- **Edge cases / failures:**
  - API key invalid / quota exceeded
  - Payload format mismatch (expects specific binary array field naming/structure)
  - Merging very large PDFs can hit API limits (file size, page count, request timeout)
  - Output fields (e.g., `fileName`) are assumed later by Slack; if absent, Slack message will render blank or error depending on expression strictness

**Sticky note context for this block:**
- **“⚙️ Processing & Consolidation”**  
  “Nodes grouped here handle the downloading, batching, and merging of validated binary data.”

---

### Block 4 — Storage + Notification

**Overview:**  
Uploads the merged “Master Report” to Drive and posts a Slack message to the finance team with a month label and the merged file name.

**Nodes involved:**
- Upload Consolidated Master Report1
- Notify Finance Team (Slack)1

#### Node: Upload Consolidated Master Report1
- **Type / role:** Google Drive node — uploads the merged PDF output.
- **Configuration (interpreted):**
  - **Drive:** “My Drive”
  - **Folder:** `root` (top-level My Drive)
  - Upload specifics (binary property, filename) are not shown; in practice you must map:
    - the merged PDF binary property from the merge node
    - a filename (ideally month-stamped)
- **Credentials:** Google Drive OAuth2 (`jitesh0dugar`)
- **Connections:**
  - **Input ←** Merge multiple PDFS into one
  - **Output →** Notify Finance Team (Slack)1
- **Version notes:** `typeVersion 3`
- **Edge cases / failures:**
  - Missing binary mapping: upload will fail if no binary property is provided
  - Permissions: cannot write to target folder
  - Filename collisions if you always upload with the same name (Drive may create duplicates or overwrite depending on settings)

#### Node: Notify Finance Team (Slack)1
- **Type / role:** Slack node — sends a message using OAuth2 auth.
- **Configuration (interpreted):**
  - Authentication: OAuth2 (`Mediajade Slack`)
  - Message text (expression):
    - Uses Luxon to format month/year in a specific timezone:
      - `{{$now.setZone('America/New_York').toFormat('MMMM yyyy')}}`
    - References merged filename from the merge node:
      - `{{ $node["Merge multiple PDFS into one"].json.fileName }}`
  - Target selection: “user” select mode is present, but the `user` value is empty in the export (likely needs to be set to a channel or user depending on node operation/config).
- **Connections:**
  - **Input ←** Upload Consolidated Master Report1
  - **Output:** none (end)
- **Version notes:** `typeVersion 2.1`
- **Edge cases / failures:**
  - Slack OAuth token revoked / missing scopes (e.g., chat:write)
  - Target misconfigured (no channel/user selected) → message send failure
  - If `fileName` doesn’t exist on the merge node output, the message will contain an empty value or error depending on evaluation

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Documentation1 | Sticky Note | Embedded documentation / setup notes | — | — | ## How it works; This workflow automates the gathering and merging of financial records… (includes setup steps and Luxon mention) |
| Trigger: Monthly on the 1st1 | Schedule Trigger | Monthly workflow start | — | List Financial PDFs from Drive1 |  |
| List Financial PDFs from Drive1 | Google Drive | List files in target folder | Trigger: Monthly on the 1st1 | Validate File Format (PDF only)1 |  |
| Validate File Format (PDF only)1 | IF | Filter to PDFs by MIME type | List Financial PDFs from Drive1 | Download Documents for Processing1 (true) |  |
| Download Documents for Processing1 | Google Drive | Download each PDF as binary | Validate File Format (PDF only)1 | Batch Files for Merging1 | ### ⚙️ Processing & Consolidation; Nodes grouped here handle the downloading, batching, and merging of validated binary data. |
| Batch Files for Merging1 | Aggregate | Collect binaries into a single batch payload | Download Documents for Processing1 | Merge multiple PDFS into one | ### ⚙️ Processing & Consolidation; Nodes grouped here handle the downloading, batching, and merging of validated binary data. |
| Merge multiple PDFS into one | HTML/CSS to PDF (htmlcsstopdf) | Merge multiple PDFs into one | Batch Files for Merging1 | Upload Consolidated Master Report1 | ### ⚙️ Processing & Consolidation; Nodes grouped here handle the downloading, batching, and merging of validated binary data. |
| Upload Consolidated Master Report1 | Google Drive | Upload merged PDF to Drive | Merge multiple PDFS into one | Notify Finance Team (Slack)1 |  |
| Notify Finance Team (Slack)1 | Slack | Notify finance team with month-stamped message | Upload Consolidated Master Report1 | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set the name to:  
   **“Consolidate and report monthly financial PDFs with Google Drive and Slack”**

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Configure it to run **monthly**.
   - If you need “on the 1st”, set:
     - **Repeat:** Monthly
     - **Day of month:** 1
     - Set a time and timezone as needed.

3. **Add node: Google Drive (List folder contents)**
   - Node type: **Google Drive**
   - **Resource:** Folder
   - **Operation:** List
   - Select the **folder containing the financial PDFs**.
   - Credentials:
     - Create/choose **Google Drive OAuth2** credentials.
     - Ensure it has permission to read the folder.
   - Connect: **Schedule Trigger → Google Drive (List)**

4. **Add node: IF (Validate PDF MIME type)**
   - Node type: **IF**
   - Add condition (String):
     - **Value 1:** `={{ $json.mimeType }}`
     - **Operation:** equals
     - **Value 2:** `application/pdf`
   - Connect: **Drive List → IF**
   - Use the **true** output for the next step.

5. **Add node: Google Drive (Download)**
   - Node type: **Google Drive**
   - **Operation:** Download
   - **File ID:** `={{ $json.id }}`
   - Use the same Google Drive OAuth2 credentials.
   - Connect: **IF (true) → Drive Download**

6. **Add node: Aggregate (Batch files for merging)**
   - Node type: **Aggregate**
   - In “Fields to Aggregate”, add:
     - **Field to aggregate:** `data`
   - This assumes the download node outputs the binary as `data`. If your Drive download outputs a different binary property name, use that instead.
   - Connect: **Drive Download → Aggregate**

7. **Add node: HTML/CSS to PDF (Merge PDFs)**
   - Node type: **HTML/CSS to PDF** (community/installed node: `n8n-nodes-htmlcsstopdf`)
   - **Resource:** PDF Manipulation
   - Configure the operation to **merge PDFs** using the aggregated binaries from the previous node (exact mapping depends on this node’s UI; ensure it receives the full list/array).
   - Credentials:
     - Create/choose **htmlcsstopdf API** credentials (API key).
   - Connect: **Aggregate → Merge PDFs**

8. **Add node: Google Drive (Upload)**
   - Node type: **Google Drive**
   - Configure for **Upload** (in the UI, choose upload operation).
   - Set:
     - **Drive:** My Drive (or shared drive as needed)
     - **Folder:** root (or your desired output folder)
   - Map:
     - **Binary property:** the merged PDF binary from the merge node
     - **File name:** recommended to include month/year (e.g., `Master Report - {{$now.toFormat('yyyy-MM')}}.pdf`)
   - Connect: **Merge PDFs → Drive Upload**

9. **Add node: Slack (Send message)**
   - Node type: **Slack**
   - Authentication: **OAuth2**
   - Select the target (channel like `#finance-dept` or a user), ensuring it’s not left blank.
   - Message text (example matching the workflow):
     - ```
       =✅ *Monthly Finance Bundle Generated* for {{$now.setZone('America/New_York').toFormat('MMMM yyyy')}}
       
       Your consolidated report is now available in Drive.
       File: `{{ $node["Merge multiple PDFS into one"].json.fileName }}`
       ```
   - Credentials:
     - Create/choose **Slack OAuth2** credentials with `chat:write` (and channel access as needed).
   - Connect: **Drive Upload → Slack**

10. **(Recommended hardening) Add guards**
   - Add an IF after listing files to check “any PDFs found” before attempting merge.
   - Optionally log or notify if no PDFs were found.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates the gathering and merging of financial records… triggers monthly… scans a specific Google Drive folder for PDF files… merged into a single ‘Master Report’… uploaded back to Drive… sends a formatted notification… via Slack.” | Sticky note: **Main Documentation1** |
| Setup steps: Google Drive connection + folder selection; HTML/CSS to PDF API credentials; Slack channel selection; Luxon expressions for date labeling. | Sticky note: **Main Documentation1** |
| Luxon formatting mentioned: `{{$now.format('MMMM')}}` and used in Slack message as `{{$now.setZone('America/New_York').toFormat('MMMM yyyy')}}`. | Sticky note + Slack node expression |
| “Nodes grouped here handle the downloading, batching, and merging of validated binary data.” | Sticky note: **Processing Sticky** |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.