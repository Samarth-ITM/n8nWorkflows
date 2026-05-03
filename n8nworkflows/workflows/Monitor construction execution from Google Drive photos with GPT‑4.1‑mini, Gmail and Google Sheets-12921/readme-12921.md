Monitor construction execution from Google Drive photos with GPT‑4.1‑mini, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-construction-execution-from-google-drive-photos-with-gpt-4-1-mini--gmail-and-google-sheets-12921


# Monitor construction execution from Google Drive photos with GPT‑4.1‑mini, Gmail and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow monitors a Google Drive folder for new construction-site photos, uses **GPT‑4.1‑mini** to generate a professional civil-engineering style inspection report from each image, emails the report (with the photo attached) via **Gmail**, logs the result into **Google Sheets** (including an inline image preview), and then moves processed images to a “Processed” Drive folder.

**Target use cases:**
- Daily/periodic site progress monitoring for clients, contractors, or internal QA/QC.
- Quick detection of apparent quality issues, safety risks, and recommended actions based on photos.

### 1.1 Scheduling & Configuration
A schedule trigger starts the run and a Config node defines timestamps, folder IDs, email target, and Sheets destinations.

### 1.2 Retrieve & Filter Candidate Files (Drive)
The workflow lists all files in the input folder and filters to keep only image MIME types.

### 1.3 Per-Image AI Analysis (Drive download + LLM Agent)
Each image is downloaded (binary), passed to an AI Agent backed by the OpenAI Chat Model, producing a structured technical assessment.

### 1.4 Reporting, Logging, and Archiving
The workflow prepares a consolidated payload, emails the report with attachment, shares the image via a public reader link, appends a log row to Google Sheets, and moves the image into a processed folder.

### 1.5 No-Data / Completion Endpoints
If no images are found, the workflow stops cleanly; after processing, it ends with a completion node.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Run Configuration
**Overview:** Starts the workflow on a configurable interval and sets all runtime constants (folder IDs, email, spreadsheet, timestamp) used downstream.

**Nodes involved:**
- Schedule Trigger (Configurable Interval)
- Config

#### Node: Schedule Trigger (Configurable Interval)
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entrypoint that runs periodically.
- **Key configuration (interpreted):**
  - Interval is set with `rule.interval` and currently uses **seconds** (field: `seconds`). (No explicit value is shown in the JSON snippet, so it may default or be incomplete depending on UI state.)
- **Connections:**
  - **Output →** Config
- **Version notes:** TypeVersion `1.3`.
- **Edge cases / failures:**
  - Too-frequent schedules may cause overlapping runs or API throttling (Drive/Gmail/OpenAI).
  - If the interval is misconfigured (e.g., missing quantity), it may not trigger as expected.

#### Node: Config
- **Type / role:** `n8n-nodes-base.set` — central configuration and runtime variables.
- **Key configuration:**
  - Sets:
    - `runTimestamp` = current time in `America/Sao_Paulo`, formatted `dd/MM/yyyy HH:mm:ss` (Luxon `DateTime.now()`).
    - `inputFolderId` = placeholder `REPLACE_WITH_INPUT_FOLDER_ID`
    - `processedFolderId` = placeholder `REPLACE_WITH_PROCESSED_FOLDER_ID`
    - `logSpreadsheetId` = placeholder `REPLACE_WITH_SPREADSHEET_ID`
    - `logSheetName` = `Página1`
    - `reportLanguage` = `English` (note: not referenced later in the provided nodes)
    - `emailTo` = `user@example.com`
    - `emailSubjectPrefix` = `Construction Photo Report`
- **Key expressions / variables:**
  - `{{ DateTime.now().setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm:ss') }}`
- **Connections:**
  - **Input ←** Schedule Trigger
  - **Output →** Drive: List files (Input folder)
- **Version notes:** TypeVersion `3.4`.
- **Edge cases / failures:**
  - If any placeholder IDs are not replaced, Drive/Sheets/Move operations will fail (404 / invalid ID).
  - Timezone string must be valid; otherwise Luxon may fall back or error depending on environment.

---

### Block 2 — Retrieve & Filter Construction Images
**Overview:** Lists files in the configured Drive folder and filters the stream to only image files.

**Nodes involved:**
- Drive: List files (Input folder)
- Filter: Keep images only
- IF: Images Found?
- End: No images found

#### Node: Drive: List files (Input folder)
- **Type / role:** `n8n-nodes-base.googleDrive` — enumerates files inside the input folder.
- **Key configuration:**
  - Resource: `fileFolder`
  - Return all: enabled
  - Filter: `folderId` = `{{$json.inputFolderId}}` (from Config)
  - Fields requested: `id`, `name`, `mimeType` (reduces payload; ensures downstream filter works)
- **Credentials:** Google Drive OAuth2.
- **Connections:**
  - **Input ←** Config
  - **Output →** Filter: Keep images only
- **Version notes:** TypeVersion `3`.
- **Edge cases / failures:**
  - Missing/invalid Drive OAuth2 token → 401/403.
  - Folder ID wrong or permission missing → 404/403.
  - Very large folders: potential execution time/memory pressure even with “Return all”.

#### Node: Filter: Keep images only
- **Type / role:** `n8n-nodes-base.filter` — keeps only items whose MIME type indicates an image.
- **Key configuration:**
  - Condition: `mimeType` **startsWith** `image/`
  - `alwaysOutputData: true` (important: workflow continues even if nothing matches; the IF node decides)
- **Connections:**
  - **Input ←** Drive: List files
  - **Output →** IF: Images Found?
- **Version notes:** TypeVersion `2.3`.
- **Edge cases / failures:**
  - Non-standard image MIME types or missing `mimeType` field could be dropped unexpectedly.
  - With `alwaysOutputData`, you may still get empty/placeholder items depending on upstream behavior.

#### Node: IF: Images Found?
- **Type / role:** `n8n-nodes-base.if` — branching logic; proceeds to processing if there is at least one non-empty item.
- **Key configuration (interpreted):**
  - Checks: `{{ $items().some(item => Object.keys(item.json).length > 0) }}` is **true**
  - This is a “global” check across items in the current execution context.
- **Connections:**
  - **True →** Drive: Download image
  - **False →** End: No images found
- **Version notes:** TypeVersion `2.3`.
- **Edge cases / failures:**
  - If the filter outputs items with minimal JSON structure, this condition might behave unexpectedly (it only checks “non-empty JSON”, not “count of images”).
  - If the workflow is executed with multiple runs/branches, `$items()` scope nuances can affect correctness.

#### Node: End: No images found
- **Type / role:** `n8n-nodes-base.noOp` — graceful stop when no images are available.
- **Connections:**
  - **Input ←** IF: Images Found? (False branch)
- **Version notes:** TypeVersion `1`.
- **Edge cases / failures:** None (acts as terminator).

---

### Block 3 — AI-Powered Construction Image Analysis
**Overview:** Downloads each image binary from Drive and uses an AI Agent (backed by OpenAI Chat Model GPT‑4.1‑mini) to generate a structured civil-engineering assessment.

**Nodes involved:**
- Drive: Download image
- OpenAI Chat Model
- AI: Analyze image

#### Node: Drive: Download image
- **Type / role:** `n8n-nodes-base.googleDrive` — downloads the file content as binary.
- **Key configuration:**
  - Operation: `download`
  - `fileId` = `{{$json.id}}` (from the listed file item)
- **Credentials:** Google Drive OAuth2.
- **Connections:**
  - **Input ←** IF: Images Found? (True)
  - **Output →** AI: Analyze image
- **Version notes:** TypeVersion `3`.
- **Edge cases / failures:**
  - If the file was deleted/moved between list and download → 404.
  - Large images may increase memory usage and LLM request size constraints (depending on how the agent attaches/uses the binary).

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM backend to the agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - No special options/tools configured.
- **Credentials:** OpenAI API credential.
- **Connections:**
  - Connected via **AI language model** output to **AI: Analyze image** (LangChain model binding).
- **Version notes:** TypeVersion `1.3`.
- **Edge cases / failures:**
  - Invalid API key or insufficient quota → 401/429.
  - Model availability changes or org restrictions could block `gpt-4.1-mini`.

#### Node: AI: Analyze image
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — runs an agent prompt to analyze the construction photo.
- **Key configuration:**
  - Prompt type: `define`
  - Prompt content: “PROMPT FOR CONSTRUCTION SITE PHOTO ANALYSIS – CIVIL ENGINEERING” with required structured sections:
    1) Stage identification  
    2) Execution quality  
    3) Non-conformities/risks  
    4) Occupational safety  
    5) Recommendations  
    Plus constraints: no speculation beyond visible evidence.
- **Inputs/Outputs:**
  - **Main input:** from Drive: Download image (includes binary data).
  - **AI model input:** from OpenAI Chat Model (via `ai_languageModel` connection).
  - **Main output:** provides `output` (used later as `imageAnalysis`).
- **Connections:**
  - **Input ←** Drive: Download image
  - **Model binding ←** OpenAI Chat Model
  - **Output →** Prepare: Image data
- **Version notes:** TypeVersion `3.1`.
- **Edge cases / failures:**
  - If the agent node is not configured to correctly read image binary as vision input, it may ignore the image and respond generically. (This depends on the node’s internal “vision” handling and how n8n passes binary to the agent.)
  - Token/context limits if prompt + image encoding is too large.
  - If the model/account doesn’t support vision, results will be poor or requests may fail.

---

### Block 4 — Distribute Report, Log to Sheets, and Archive
**Overview:** Consolidates image metadata + AI output, emails a report with the image attached, shares the image publicly for embedding, logs a row to Google Sheets (with an `IMAGE()` formula), then moves the file to the processed folder.

**Nodes involved:**
- Prepare: Image data
- Gmail: Send report
- Drive: Share image (anyone reader)
- Sheets: Append log row
- Drive: Move image (Processed folder)
- End: Processing Complete

#### Node: Prepare: Image data
- **Type / role:** `n8n-nodes-base.set` — creates a clean payload for downstream nodes.
- **Key configuration (fields created):**
  - `imageId` = `{{ $('Drive: Download image').item.json.id }}`
  - `imageName` = `{{ $('Drive: Download image').item.json.name }}`
  - `binaryImage` (binary) = `{{ $('Drive: Download image').item.binary.data }}`
  - `imageAnalysis` = `{{ $json.output }}` (the agent’s textual result)
- **Connections:**
  - **Input ←** AI: Analyze image
  - **Output →** Gmail: Send report
- **Version notes:** TypeVersion `3.4`.
- **Edge cases / failures:**
  - If the download node’s binary property name differs from expectations, `binary.data` may not exist (attachment breaks).
  - If the agent output key is not exactly `output`, `imageAnalysis` will be empty.

#### Node: Gmail: Send report
- **Type / role:** `n8n-nodes-base.gmail` — emails the AI report and attaches the image.
- **Key configuration:**
  - To: `{{ $('Config').item.json.emailTo }}`
  - Subject: `{{ $('Config').item.json.emailSubjectPrefix }} - {{ $('Config').item.json.runTimestamp }}`
  - Body message: `{{ $json.imageAnalysis }}`
  - Attachments: binary property `binaryImage`
  - Email type: text
- **Credentials:** Gmail OAuth2.
- **Connections:**
  - **Input ←** Prepare: Image data
  - **Output →** Drive: Share image (anyone reader)
- **Version notes:** TypeVersion `2.2`.
- **Edge cases / failures:**
  - Gmail API scopes/authorization missing → 401/403.
  - “From” address restrictions if using a non-matching authenticated mailbox.
  - Attachment too large (Gmail limits) → send failure.

#### Node: Drive: Share image (anyone reader)
- **Type / role:** `n8n-nodes-base.googleDrive` — sets public “anyone with link can read” permission.
- **Key configuration:**
  - Operation: `share`
  - File ID: `{{ $('Prepare: Image data').item.json.imageId }}`
  - Permission: `type=anyone`, `role=reader`
- **Credentials:** Google Drive OAuth2.
- **Connections:**
  - **Input ←** Gmail: Send report
  - **Output →** Sheets: Append log row
- **Version notes:** TypeVersion `3`.
- **Edge cases / failures:**
  - Domain/admin policies may block public sharing → 403.
  - If file already has restricted permissions, share may still fail depending on ownership.

#### Node: Sheets: Append log row
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row to a log sheet with timestamp, filename, embedded image preview, and report.
- **Key configuration:**
  - Operation: `append`
  - Document ID: `{{ $('Config').item.json.logSpreadsheetId }}`
  - Sheet name: `{{ $('Config').item.json.logSheetName }}`
  - Mapped columns:
    - **Foto**: `{{ $('Prepare: Image data').item.json.imageName }}`
    - **Imagem**: `=IMAGE("https://drive.google.com/uc?export=view&id={{ $('Prepare: Image data').item.json.imageId }}")`
      - Note: In the provided workflow text, the value begins with `==IMAGE(...)` (double `=`). This likely should be a **single** leading `=` to be recognized as a formula in Sheets.
    - **Relato**: `{{ $('Prepare: Image data').item.json.imageAnalysis }}`
    - **Data/hora**: `{{ $('Config').item.json.runTimestamp }}`
- **Credentials:** Google Sheets OAuth2.
- **Connections:**
  - **Input ←** Drive: Share image
  - **Output →** Drive: Move image (Processed folder)
- **Version notes:** TypeVersion `4.7`.
- **Edge cases / failures:**
  - Spreadsheet/sheet not found → 404.
  - Header mismatch: if the target sheet doesn’t have these columns or mapping mode expectations differ, data may go into wrong columns.
  - Formula handling: depending on node settings, formulas might be written as plain text; verify with Sheets node options.
  - The image may not render if sharing failed or Drive blocks unauthenticated access.

#### Node: Drive: Move image (Processed folder)
- **Type / role:** `n8n-nodes-base.googleDrive` — moves processed file out of the input folder to prevent reprocessing.
- **Key configuration:**
  - Operation: `move`
  - File ID: `{{ $('Prepare: Image data').item.json.imageId }}`
  - Destination folder ID: `{{ $('Config').item.json.processedFolderId }}`
  - Drive: `My Drive`
- **Credentials:** Google Drive OAuth2.
- **Connections:**
  - **Input ←** Sheets: Append log row
  - **Output →** End: Processing Complete
- **Version notes:** TypeVersion `3`.
- **Edge cases / failures:**
  - If destination folder ID is wrong or permission missing → 404/403.
  - If file is on a Shared Drive but “My Drive” is selected, move can fail or behave unexpectedly.

#### Node: End: Processing Complete
- **Type / role:** `n8n-nodes-base.noOp` — indicates successful completion path.
- **Connections:**
  - **Input ←** Drive: Move image (Processed folder)
- **Version notes:** TypeVersion `1`.
- **Edge cases / failures:** None.

---

### Block 5 — Documentation / Visual Grouping (Sticky Notes)
**Overview:** Sticky notes describe the intent, setup checklist, and visually label the three major phases.

**Nodes involved:**
- Sticky Note
- Sticky Note3
- Sticky Note4
- Sticky Note5

#### Node: Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — documentation and setup checklist.
- **Content highlights:**
  - Explains the 4-step flow (schedule → filter/download → AI analysis → email/log/move).
  - Setup checklist for Drive, Gmail, OpenAI, Sheets IDs, and schedule interval.
- **Connections:** None (informational only).

#### Node: Sticky Note3
- **Role:** Visual label: “## 1. Retrieve and Filter Construction Images”
- **Connections:** None.

#### Node: Sticky Note4
- **Role:** Visual label: “## 2. AI-Powered Construction Image Analysis”
- **Connections:** None.

#### Node: Sticky Note5
- **Role:** Visual label: “## 3. Distribute Report and Log Work Performance Data”
- **Connections:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger (Configurable Interval) | scheduleTrigger | Periodic workflow entrypoint | — | Config | ## AI-Driven construction execution monitoring / How it works / Setup checklist |
| Config | set | Defines runtime variables (folder IDs, email, timestamp) | Schedule Trigger (Configurable Interval) | Drive: List files (Input folder) | ## AI-Driven construction execution monitoring / How it works / Setup checklist |
| Drive: List files (Input folder) | googleDrive | Lists files in input folder | Config | Filter: Keep images only | ## 1. Retrieve and Filter Construction Images |
| Filter: Keep images only | filter | Keep only image MIME types | Drive: List files (Input folder) | IF: Images Found? | ## 1. Retrieve and Filter Construction Images |
| IF: Images Found? | if | Branch if any image items exist | Filter: Keep images only | Drive: Download image; End: No images found | ## 1. Retrieve and Filter Construction Images |
| End: No images found | noOp | Ends run when no images | IF: Images Found? (false) | — | ## 1. Retrieve and Filter Construction Images |
| Drive: Download image | googleDrive | Downloads image binary | IF: Images Found? (true) | AI: Analyze image | ## 2. AI-Powered Construction Image Analysis |
| OpenAI Chat Model | lmChatOpenAi | Provides GPT‑4.1‑mini model to agent | — (model binding) | AI: Analyze image (ai_languageModel) | ## 2. AI-Powered Construction Image Analysis |
| AI: Analyze image | langchain.agent | Generates technical report from image | Drive: Download image + OpenAI Chat Model | Prepare: Image data | ## 2. AI-Powered Construction Image Analysis |
| Prepare: Image data | set | Consolidates id/name/binary/report for downstream | AI: Analyze image | Gmail: Send report | ## 3. Distribute Report and Log Work Performance Data |
| Gmail: Send report | gmail | Emails report with image attachment | Prepare: Image data | Drive: Share image (anyone reader) | ## 3. Distribute Report and Log Work Performance Data |
| Drive: Share image (anyone reader) | googleDrive | Makes image publicly viewable for embedding | Gmail: Send report | Sheets: Append log row | ## 3. Distribute Report and Log Work Performance Data |
| Sheets: Append log row | googleSheets | Appends timestamp/name/report + IMAGE() preview | Drive: Share image (anyone reader) | Drive: Move image (Processed folder) | ## 3. Distribute Report and Log Work Performance Data |
| Drive: Move image (Processed folder) | googleDrive | Moves processed image to prevent reprocessing | Sheets: Append log row | End: Processing Complete | ## 3. Distribute Report and Log Work Performance Data |
| End: Processing Complete | noOp | Successful completion endpoint | Drive: Move image (Processed folder) | — | ## 3. Distribute Report and Log Work Performance Data |
| Sticky Note | stickyNote | Workflow description + setup checklist | — | — | ## AI-Driven construction execution monitoring (content itself) |
| Sticky Note3 | stickyNote | Section label | — | — | ## 1. Retrieve and Filter Construction Images |
| Sticky Note4 | stickyNote | Section label | — | — | ## 2. AI-Powered Construction Image Analysis |
| Sticky Note5 | stickyNote | Section label | — | — | ## 3. Distribute Report and Log Work Performance Data |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: *AI-driven construction execution monitoring* (or your preferred name).

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Configure interval (recommended: every X minutes/hours; avoid “every few seconds” unless needed).
   - Connect: **Schedule Trigger → Config**

3. **Add node: Config (Set)**
   - Node type: **Set**
   - Add fields:
     - `runTimestamp` (String): `{{ DateTime.now().setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm:ss') }}`
     - `inputFolderId` (String): your Google Drive folder ID containing incoming photos
     - `processedFolderId` (String): folder ID to move processed photos into
     - `emailTo` (String): recipient email
     - `emailSubjectPrefix` (String): e.g., `Construction Photo Report`
     - `logSpreadsheetId` (String): Google Sheets spreadsheet ID
     - `logSheetName` (String): sheet/tab name (e.g., `Página1`)
     - (Optional) `reportLanguage` (String): `English` (only useful if you also wire it into the prompt later)
   - Connect: **Config → Drive: List files (Input folder)**

4. **Add node: Google Drive — List files**
   - Node type: **Google Drive**
   - Operation/resource: list files in folder (file/folder resource)
   - Folder ID: `{{ $json.inputFolderId }}`
   - Enable **Return All**
   - Fields: include at least `id`, `name`, `mimeType`
   - Credentials: connect **Google Drive OAuth2**
   - Connect: **List files → Filter: Keep images only**

5. **Add node: Filter — Keep images only**
   - Node type: **Filter**
   - Condition: `{{ $json.mimeType }}` **startsWith** `image/`
   - Set **Always Output Data** = true
   - Connect: **Filter → IF: Images Found?**

6. **Add node: IF — Images Found?**
   - Node type: **IF**
   - Condition (Boolean “is true”):
     - Left value: `{{ $items().some(item => Object.keys(item.json).length > 0) }}`
   - Connect:
     - **True → Drive: Download image**
     - **False → End: No images found** (NoOp)

7. **Add node: NoOp — End: No images found**
   - Node type: **No Operation (NoOp)**
   - No config needed.

8. **Add node: Google Drive — Download image**
   - Node type: **Google Drive**
   - Operation: **Download**
   - File ID: `{{ $json.id }}`
   - Credentials: same Drive OAuth2
   - Connect: **Download image → AI: Analyze image**

9. **Add node: OpenAI Chat Model**
   - Node type: **OpenAI Chat Model** (LangChain / n8n)
   - Model: **gpt-4.1-mini**
   - Credentials: **OpenAI API** (API key)
   - Connect the model to the agent using the **AI languageModel** connection:
     - **OpenAI Chat Model (ai_languageModel) → AI: Analyze image**

10. **Add node: AI Agent — Analyze image**
   - Node type: **AI Agent** (LangChain agent)
   - Prompt mode: **Define**
   - Paste the provided civil-engineering prompt (stage, quality, non-conformities, safety, recommendations; no speculation).
   - Ensure the node is configured (where applicable) to accept image/binary input for vision analysis (depends on your n8n/agent node UI).
   - Connect: **AI: Analyze image → Prepare: Image data**

11. **Add node: Set — Prepare: Image data**
   - Node type: **Set**
   - Add fields:
     - `imageId` (String): `{{ $('Drive: Download image').item.json.id }}`
     - `imageName` (String): `{{ $('Drive: Download image').item.json.name }}`
     - `binaryImage` (Binary): `{{ $('Drive: Download image').item.binary.data }}`
     - `imageAnalysis` (String): `{{ $json.output }}`
   - Connect: **Prepare: Image data → Gmail: Send report**

12. **Add node: Gmail — Send report**
   - Node type: **Gmail**
   - Operation: Send
   - To: `{{ $('Config').item.json.emailTo }}`
   - Subject: `{{ $('Config').item.json.emailSubjectPrefix }} - {{ $('Config').item.json.runTimestamp }}`
   - Message body: `{{ $json.imageAnalysis }}`
   - Attachment: attach binary property `binaryImage`
   - Credentials: **Gmail OAuth2**
   - Connect: **Gmail: Send report → Drive: Share image (anyone reader)**

13. **Add node: Google Drive — Share image (anyone reader)**
   - Node type: **Google Drive**
   - Operation: **Share**
   - File ID: `{{ $('Prepare: Image data').item.json.imageId }}`
   - Permission: `Anyone` + `Reader`
   - Connect: **Share image → Sheets: Append log row**

14. **Add node: Google Sheets — Append log row**
   - Node type: **Google Sheets**
   - Operation: **Append**
   - Spreadsheet ID: `{{ $('Config').item.json.logSpreadsheetId }}`
   - Sheet name: `{{ $('Config').item.json.logSheetName }}`
   - Map columns:
     - `Data/hora`: `{{ $('Config').item.json.runTimestamp }}`
     - `Foto`: `{{ $('Prepare: Image data').item.json.imageName }}`
     - `Relato`: `{{ $('Prepare: Image data').item.json.imageAnalysis }}`
     - `Imagem`: `=IMAGE("https://drive.google.com/uc?export=view&id={{ $('Prepare: Image data').item.json.imageId }}")`
       - Use a **single** leading `=` so Google Sheets recognizes it as a formula.
   - Credentials: **Google Sheets OAuth2**
   - Connect: **Sheets: Append log row → Drive: Move image (Processed folder)**

15. **Add node: Google Drive — Move image (Processed folder)**
   - Node type: **Google Drive**
   - Operation: **Move**
   - File ID: `{{ $('Prepare: Image data').item.json.imageId }}`
   - Destination folder ID: `{{ $('Config').item.json.processedFolderId }}`
   - Connect: **Move image → End: Processing Complete**

16. **Add node: NoOp — End: Processing Complete**
   - Node type: **No Operation (NoOp)**

17. **Credentials checklist**
   - Google Drive OAuth2: must have access to input folder and processed folder, and permission to change sharing settings.
   - Gmail OAuth2: must be authorized to send email as the authenticated account.
   - Google Sheets OAuth2: must have edit access to the spreadsheet.
   - OpenAI API: valid key with access to `gpt-4.1-mini` (and vision capability if required by your analysis approach).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Driven construction execution monitoring” sticky note explains the end-to-end flow and provides a setup checklist (Drive folder IDs, Gmail auth, OpenAI key, Sheets log IDs, processed folder ID, schedule interval). | Sticky Note content in the canvas |
| The Sheets image preview relies on a public Drive link and the `IMAGE()` formula using `https://drive.google.com/uc?export=view&id=...`. Public sharing may be blocked by admin policy. | Applies to “Drive: Share image” + “Sheets: Append log row” |
| The configured variable `reportLanguage` is set to `English` but is not referenced in downstream nodes in the provided workflow; incorporate it into the prompt if multilingual output is desired. | Config node design consideration |
| The provided Sheets formula mapping shows `==IMAGE(...)` (double equals) in the workflow JSON; it should typically be `=IMAGE(...)`. | Potential logging/formatting issue |

