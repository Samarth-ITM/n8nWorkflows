Back up databases and files to Box with Mailgun email notifications

https://n8nworkflows.xyz/workflows/back-up-databases-and-files-to-box-with-mailgun-email-notifications-13260


# Back up databases and files to Box with Mailgun email notifications

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Back up databases and files to Box with Mailgun email notifications  
**Workflow name (JSON):** Scheduled Backup Automation – Mailgun & Box  
**Purpose:** Run an on-demand (or externally scheduled) multi-asset backup by fetching export files from external endpoints, compressing them to GZIP, uploading them into a date-organized Box folder, and sending per-asset success/failure notifications via Mailgun.

### 1.1 Trigger & Input Reception
A **Webhook** receives a `POST` call (from cron/CI/CD/etc.). The workflow then applies defaults (and/or merges user input, depending on how the Set node is configured).

### 1.2 Box Folder Provisioning
A Box folder is created to store the current run’s backup artifacts (typically named by date/time in a real setup).

### 1.3 Per-Backup Iteration (Batch Loop)
The workflow expands the backup definitions into individual items and processes them one-by-one through:
- export download (HTTP)
- export validation (status code check)
- compression (GZIP)
- binary preparation for Box upload
- upload + upload validation

### 1.4 Notification & Reporting
Each backup item generates either a success email or a failure email through Mailgun.

---

## 2. Block-by-Block Analysis

### Block A — Trigger & Preparation
**Overview:** Accepts a webhook call to start a backup run, applies default backup configuration, creates a destination folder in Box, and transforms the backup list into one item per backup target (each annotated with the folderId).

**Nodes involved:**
- Backup Webhook Trigger
- Set Default Backup Config
- Create Backup Folder
- Prepare Backup Items

#### Node: Backup Webhook Trigger
- **Type / role:** `Webhook` (entry point). Receives external `POST` requests to initiate the workflow.
- **Key configuration:**
  - **HTTP Method:** `POST`
  - **Path:** `scheduled-backup`
  - **Response mode:** `lastNode` (the HTTP response is the output of the last executed node in the path)
- **Inputs / outputs:** No input; outputs to **Set Default Backup Config**.
- **Edge cases / failures:**
  - If called with unexpected payload shape, later expressions referencing missing fields may fail (depending on how defaults are applied).
  - If multiple branches execute, “lastNode” can be confusing to callers (the returned payload may differ based on success/failure path and which item finishes last).

#### Node: Set Default Backup Config
- **Type / role:** `Set` (data shaping). Intended to ensure a default list of backups exists and optionally merge incoming webhook payload.
- **Key configuration:**
  - **Mode:** `keep` (keeps existing fields; typically used to avoid wiping input)
  - The JSON shown does **not** list explicit fields; however downstream code expects a structure like:
    - `backups`: array of backup definitions, each containing at least `name` and `exportUrl`.
- **Inputs / outputs:** Input from webhook; output to **Create Backup Folder**.
- **Edge cases / failures:**
  - If `backups` is not actually set/merged here, **Prepare Backup Items** will fail when reading `$node["Set Default Backup Config"].json["backups"]`.
  - If backups exist but are not an array, the Code node mapping will throw.

#### Node: Create Backup Folder
- **Type / role:** `Box` node (provisioning storage container).
- **Operation:** `createFolder`
- **Key configuration (implied):**
  - Requires Box credentials (OAuth2 / JWT depending on your n8n setup and Box app type).
  - Needs a **folder name** and **parent folder** in typical Box API usage. These are not visible in the JSON snippet, so they must be configured in the node UI for a working workflow.
- **Inputs / outputs:** Input from Set node; outputs to **Prepare Backup Items**.
- **Edge cases / failures:**
  - Auth/permission issues (cannot create folder in target parent).
  - Name conflicts if using deterministic names (Box may reject duplicates depending on settings).
  - Missing required parameters (folder name/parent) will fail at runtime.

#### Node: Prepare Backup Items
- **Type / role:** `Code` (item fan-out). Creates one n8n item per backup definition and injects the Box `folderId` into each.
- **Key code logic:**
  - Reads `folderId` from: `$node["Create Backup Folder"].json["id"]`
  - Reads backups from: `$node["Set Default Backup Config"].json["backups"]`
  - Returns: `backups.map(b => ({ json: { ...b, folderId } }))`
- **Inputs / outputs:** Input from Create Backup Folder; output to **Split Backups**.
- **Edge cases / failures:**
  - If folder creation failed or returned no `id`, folderId will be `undefined`, causing later upload destination errors.
  - If backups contain unexpected keys, they’ll pass through—good for flexibility, but can hide typos (e.g., `exportURL` vs `exportUrl`).

---

### Block B — Export Retrieval & Validation (Per Backup Item)
**Overview:** Iterates through backups, downloads each export from its target URL, and gates the flow based on HTTP status.

**Nodes involved:**
- Split Backups
- Export Backup Data
- Was Export Successful?

#### Node: Split Backups
- **Type / role:** `Split In Batches` (loop controller). Feeds backup items in controlled batches (here, default behavior).
- **Key configuration:**
  - Options empty; default batch size is commonly `1` unless set otherwise in UI.
- **Inputs / outputs:** Input from Prepare Backup Items; outputs to **Export Backup Data**.
- **Edge cases / failures:**
  - In many workflows, Split In Batches requires a “continue” connection back into itself to process subsequent batches. This workflow has **no loop-back connection**, so it may only process the first batch/item depending on n8n’s execution behavior and the node configuration at runtime.
  - If you intended “one by one for all backups”, you usually need a loop-back from the last node in the per-item chain to Split In Batches.

#### Node: Export Backup Data
- **Type / role:** `HTTP Request` (fetch export payload).
- **Key configuration:**
  - **URL:** `={{ $json.exportUrl }}`
  - Other options not specified (timeouts, response format, binary download, auth).
- **Inputs / outputs:** Input from Split Backups; output to **Was Export Successful?**
- **Critical integration requirement:**
  - For compression/upload to work, this request should return **binary data** (download file) and/or provide binary in `item.binary`. If it returns only JSON, the compression node will fail when it tries `Object.keys(item.binary)[0]`.
- **Edge cases / failures:**
  - Non-200 responses, redirects, timeouts.
  - If export endpoint requires auth/IP allowlisting, it will fail until configured.
  - If response is large, memory usage may be significant (especially if fully buffered before gzip).

#### Node: Was Export Successful?
- **Type / role:** `IF` (quality gate).
- **Condition:**
  - Number: `{{ $json.statusCode }}` equals `200`
- **Outputs:**
  - **True →** Compress Export
  - **False →** Compose Failure Email
- **Edge cases / failures:**
  - Depending on HTTP Request node settings, `statusCode` may not be present in `$json` (it might be in a different field or not included). If missing, the condition may evaluate unexpectedly.
  - Some export endpoints return `204`, `201`, or `302` for valid flows; this strict check would mark them as failures.

---

### Block C — Compression, Box Upload & Upload Validation (Per Backup Item)
**Overview:** Compresses the exported binary to `.gz`, ensures the binary field matches what the Box node expects, uploads to Box, and validates upload success.

**Nodes involved:**
- Compress Export
- Prepare File for Box
- Upload to Box
- Was Upload Successful?

#### Node: Compress Export
- **Type / role:** `Code` (binary transformation). GZIP-compresses the downloaded export and creates a timestamped `.gz` binary for upload.
- **Key code logic:**
  - Uses `zlib.gzipSync(...)` on the first binary property found:
    - `const binKey = Object.keys(item.binary)[0];`
    - `Buffer.from(item.binary[binKey].data, 'base64')`
  - Outputs:
    - `json`: `{ name, folderId }`
    - `binary.data`: `{ data: <base64 gzip>, mimeType: 'application/gzip', fileName: `${name}-${ISO}.gz` }`
- **Inputs / outputs:** Input from IF (true branch); output to **Prepare File for Box**
- **Edge cases / failures:**
  - If there is no `item.binary`, or it’s empty → `Object.keys(item.binary)[0]` throws.
  - Very large exports may cause memory pressure because `gzipSync` is synchronous and buffers entire content in memory.
  - Filenames include `:` from ISO timestamps; some systems dislike that (Box generally tolerates, but it can be a portability concern).

#### Node: Prepare File for Box
- **Type / role:** `Move Binary Data` (binary key normalization).
- **Key configuration:** Options empty; commonly used to rename/move `binary.data` to the property name required by the next node.
- **Inputs / outputs:** Input from Compress Export; output to **Upload to Box**
- **Edge cases / failures:**
  - If the Box node expects a specific binary property (e.g., `data`), misconfiguration here can cause “No binary data found” errors.
  - With empty options, it may not actually rename anything; verify in UI which property Box upload is configured to read.

#### Node: Upload to Box
- **Type / role:** `Box` node (file upload).
- **Operation:** Not shown in parameters; must be configured in UI (typically “Upload” / “Upload File”).
- **Key configuration (implied):**
  - Destination folder should use `={{ $json.folderId }}` from upstream.
  - File content should come from the prepared binary property.
- **Inputs / outputs:** Input from Prepare File for Box; output to **Was Upload Successful?**
- **Edge cases / failures:**
  - Auth/permission issues to upload into the created folder.
  - Missing/incorrect folderId.
  - Upload size limits/timeouts.
  - If operation is not set, node will fail immediately.

#### Node: Was Upload Successful?
- **Type / role:** `IF` (upload verification).
- **Condition:**
  - String check: `{{ $json.id }}` **isEmpty**
- **Branch meaning (important):**
  - If `id isEmpty` = **True**, that indicates **failure** (no file id returned).
  - In this workflow’s wiring:
    - **True → Compose Success Email**
    - **False → Compose Failure Email**
  - This appears **reversed** logically. Usually you want:
    - If `id isEmpty` → Failure path
    - Else → Success path
- **Inputs / outputs:** Input from Upload to Box; outputs to Compose Success/Failure accordingly.
- **Edge cases / failures:**
  - Box upload responses may store the ID under another field depending on node version/operation (e.g., `entries[0].id`); checking `$json.id` may be wrong.
  - If Box node returns a complex object, “isEmpty” might behave unexpectedly.

---

### Block D — Notifications (Per Backup Item)
**Overview:** Builds an email payload for success or failure and sends it via Mailgun. Because this is inside the per-item path, it can generate one email per backup target.

**Nodes involved:**
- Compose Success Email
- Send Success Notification
- Compose Failure Email
- Send Failure Notification

#### Node: Compose Success Email
- **Type / role:** `Set` (compose message fields).
- **Key configuration:**
  - Mode: `add`
  - Actual fields aren’t shown, but Mailgun node expects at least:
    - `emailSubject` (used as expression in Mailgun subject)
    - typically `text` / `html` body fields (depending on Mailgun node configuration)
- **Inputs / outputs:** Input from Was Upload Successful?; output to Send Success Notification.
- **Edge cases / failures:**
  - If `emailSubject` isn’t created here, Mailgun node subject expression resolves to empty.
  - If you want per-asset details, ensure you carry `name`, `exportUrl`, and timestamps into the email body.

#### Node: Send Success Notification
- **Type / role:** `Mailgun` (send email).
- **Key configuration:**
  - Subject: `={{ $json.emailSubject }}`
  - To: `admin@example.com`
  - From: `user@example.com`
  - Credentials required (Mailgun API key + domain, depending on n8n credential type).
- **Inputs / outputs:** Input from Compose Success Email; terminal in that branch.
- **Edge cases / failures:**
  - Unverified sender domain / invalid From.
  - Mailgun sandbox restrictions.
  - Rate limits.

#### Node: Compose Failure Email
- **Type / role:** `Set` (compose failure message fields).
- **Key configuration:**
  - Mode: `add`
  - Should set `emailSubject` and body describing whether export failed (non-200) or upload failed (no file id).
- **Inputs / outputs:** Input from Was Export Successful? (false branch) and Was Upload Successful? (failure branch); output to Send Failure Notification.
- **Edge cases / failures:**
  - If called from export failure, Box-related fields may be missing; build messages defensively.

#### Node: Send Failure Notification
- **Type / role:** `Mailgun` (send email).
- **Key configuration:**
  - Subject: `={{ $json.emailSubject }}`
  - To/From same placeholders as success node.
- **Inputs / outputs:** Input from Compose Failure Email; terminal in that branch.
- **Edge cases / failures:** Same as success email.

---

### Block E — Documentation Notes (Sticky Notes)
**Overview:** Four sticky notes document intent, setup steps, and the rationale for each cluster.

**Nodes involved:**
- Workflow Overview (sticky note)
- Trigger & Prep Note (sticky note)
- Compression & Storage Note (sticky note)
- Notification & Reporting Note (sticky note)

Sticky notes do not affect execution but provide important operational guidance.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Backup Webhook Trigger | Webhook | Entry point (POST trigger) | — | Set Default Backup Config | Trigger & Preparation: This cluster receives the incoming request that kicks off each backup cycle… |
| Set Default Backup Config | Set | Apply/merge default backup definitions | Backup Webhook Trigger | Create Backup Folder | Trigger & Preparation: This cluster receives the incoming request that kicks off each backup cycle… |
| Create Backup Folder | Box | Create a dated folder for this run | Set Default Backup Config | Prepare Backup Items | Trigger & Preparation: This cluster receives the incoming request that kicks off each backup cycle… |
| Prepare Backup Items | Code | Fan-out backups; inject folderId | Create Backup Folder | Split Backups | Trigger & Preparation: This cluster receives the incoming request that kicks off each backup cycle… |
| Split Backups | Split In Batches | Iterate through backup items | Prepare Backup Items | Export Backup Data | Compression & Storage: After each backup definition reaches the Split node… |
| Export Backup Data | HTTP Request | Download export payload from `exportUrl` | Split Backups | Was Export Successful? | Compression & Storage: After each backup definition reaches the Split node… |
| Was Export Successful? | IF | Gate on HTTP status code 200 | Export Backup Data | Compress Export; Compose Failure Email | Compression & Storage: After each backup definition reaches the Split node… |
| Compress Export | Code | GZIP compress binary export, set filename | Was Export Successful? (true) | Prepare File for Box | Compression & Storage: After each backup definition reaches the Split node… |
| Prepare File for Box | Move Binary Data | Align binary property for Box upload | Compress Export | Upload to Box | Compression & Storage: After each backup definition reaches the Split node… |
| Upload to Box | Box | Upload `.gz` file into run folder | Prepare File for Box | Was Upload Successful? | Compression & Storage: After each backup definition reaches the Split node… |
| Was Upload Successful? | IF | Validate Box upload returned a file ID | Upload to Box | Compose Success Email; Compose Failure Email | Compression & Storage: After each backup definition reaches the Split node… |
| Compose Success Email | Set | Create success email subject/body fields | Was Upload Successful? | Send Success Notification | Notification & Reporting: Clear visibility is critical for any backup strategy… |
| Send Success Notification | Mailgun | Send success email | Compose Success Email | — | Notification & Reporting: Clear visibility is critical for any backup strategy… |
| Compose Failure Email | Set | Create failure email subject/body fields | Was Export Successful? (false); Was Upload Successful? (failure) | Send Failure Notification | Notification & Reporting: Clear visibility is critical for any backup strategy… |
| Send Failure Notification | Mailgun | Send failure email | Compose Failure Email | — | Notification & Reporting: Clear visibility is critical for any backup strategy… |
| Workflow Overview | Sticky Note | Embedded operational description and setup checklist | — | — | ## How it works … / ## Setup steps … |
| Trigger & Prep Note | Sticky Note | Explains trigger + defaults + folder creation | — | — | ## Trigger & Preparation … |
| Compression & Storage Note | Sticky Note | Explains export, gzip, upload checks | — | — | ## Compression & Storage … |
| Notification & Reporting Note | Sticky Note | Explains per-asset emails and customization | — | — | ## Notification & Reporting … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Webhook node**
   - Name: `Backup Webhook Trigger`
   - Method: `POST`
   - Path: `scheduled-backup`
   - Response Mode: `Last Node`
3. **Add a Set node**
   - Name: `Set Default Backup Config`
   - Mode: `Keep` (so incoming payload can remain available)
   - Add (or ensure) a field `backups` as an **array of objects**, e.g. each object:
     - `name` (string)
     - `exportUrl` (string URL to download the export)
   - If you want overrides from webhook payload, design the Set node to merge when provided (commonly done with expressions).
4. **Add a Box node for folder creation**
   - Name: `Create Backup Folder`
   - Operation: `Create Folder`
   - Configure:
     - Parent folder (Box folder ID)
     - Folder name (typically include date, e.g. `backups-{{ $now.toFormat('yyyy-LL-dd') }}`)
   - **Credentials:** Create and select Box credentials (OAuth2/JWT per your environment).
5. **Add a Code node**
   - Name: `Prepare Backup Items`
   - Paste logic to fan-out items and inject folderId (as in the workflow):
     - Read folder id from `Create Backup Folder`
     - Read backups from `Set Default Backup Config`
     - Return one item per backup with `folderId`
6. **Add Split In Batches**
   - Name: `Split Backups`
   - Set batch size (commonly `1`) if you want sequential processing.
   - (Recommended) Ensure you build a complete loop if you truly need all items processed; typically you add a loop-back from the end of the per-item chain to this node.
7. **Add HTTP Request**
   - Name: `Export Backup Data`
   - URL: `{{ $json.exportUrl }}`
   - Configure it to **download as file/binary** (so output includes `item.binary`).
   - Configure auth headers if your export endpoints require them.
8. **Add an IF node**
   - Name: `Was Export Successful?`
   - Condition: numeric `statusCode == 200` (or broaden to 2xx if needed)
9. **Add a Code node (compression)**
   - Name: `Compress Export`
   - Use Node.js `zlib` to gzip the binary content and output it as `binary.data` with `.gz` filename.
10. **Add Move Binary Data**
    - Name: `Prepare File for Box`
    - Configure to ensure the binary property name matches what the Box upload node expects (verify in Box node’s “Binary Property” setting).
11. **Add a Box node for upload**
    - Name: `Upload to Box`
    - Operation: `Upload` (or “Upload File”)
    - Folder ID: `{{ $json.folderId }}`
    - Binary property: the one produced by the previous node (commonly `data`)
12. **Add an IF node**
    - Name: `Was Upload Successful?`
    - Check for presence of the uploaded file ID in the upload response.
    - Important: wire branches correctly (usually “ID exists” → success; “ID missing” → failure).
13. **Add Set node for success email**
    - Name: `Compose Success Email`
    - Mode: `Add`
    - Add `emailSubject` and body fields you want to send (text/html).
14. **Add Mailgun node for success**
    - Name: `Send Success Notification`
    - To: `admin@example.com` (replace)
    - From: `user@example.com` (replace with a verified sender)
    - Subject: `{{ $json.emailSubject }}`
    - **Credentials:** configure Mailgun credentials (API key/domain) and select them.
15. **Add Set node for failure email**
    - Name: `Compose Failure Email`
    - Mode: `Add`
    - Add `emailSubject` and error details (include whether export or upload failed).
16. **Add Mailgun node for failure**
    - Name: `Send Failure Notification`
    - Configure To/From/Subject similarly.
17. **Connect nodes in this order**
    - Webhook → Set → Box(Create folder) → Code(fan-out) → SplitInBatches → HTTP → IF(export ok)
    - IF true → Code(gzip) → MoveBinary → Box(upload) → IF(upload ok) → success Set → success Mailgun
    - IF false (export) → failure Set → failure Mailgun
    - IF upload failure → failure Set → failure Mailgun
18. **Credentials & security**
    - Box: ensure the app has permission to create folders and upload files to the chosen parent folder.
    - Mailgun: verify domain/sender; ensure API key is stored in n8n credentials.
    - Export endpoints: whitelist n8n IPs if required; add auth headers/tokens.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates multi-asset backups on demand or from external schedulers…” | Sticky note: **Workflow Overview** |
| Setup steps: create Box credentials, create Mailgun credentials, replace export URLs, whitelist IPs, schedule POST to webhook, test small backup, monitor Mailgun logs | Sticky note: **Workflow Overview** |
| Trigger & Preparation cluster description (webhook + defaults + Box folder + per-item expansion) | Sticky note: **Trigger & Prep Note** |
| Compression & Storage cluster description (HTTP export → 200 check → gzip → Move Binary Data → Box upload → file id check) | Sticky note: **Compression & Storage Note** |
| Notification & Reporting cluster description (one email per asset; customize for aggregation/logs/stakeholders) | Sticky note: **Notification & Reporting Note** |