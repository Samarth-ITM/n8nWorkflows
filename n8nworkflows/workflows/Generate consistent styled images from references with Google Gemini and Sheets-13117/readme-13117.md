Generate consistent styled images from references with Google Gemini and Sheets

https://n8nworkflows.xyz/workflows/generate-consistent-styled-images-from-references-with-google-gemini-and-sheets-13117


# Generate consistent styled images from references with Google Gemini and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate consistent styled images from references with Google Gemini and Sheets  
**Internal name:** AI Style Analysis and Consistent Image Generation with Gemini

This workflow is an automated image generation pipeline that preserves *visual consistency* by first analyzing a **reference image** (style, palette, composition) with **Google Gemini**, then generating a **new image** aligned to that style, storing it in **Google Drive**, and writing results back to **Google Sheets**. It is triggered via a **Webhook** and sends progress notifications via **ntfy.sh**.

### 1.1 Trigger & Immediate Response (Webhook + CORS)
Receives an external POST request, instantly returns a JSON “started” response (with CORS headers), then continues processing asynchronously.

### 1.2 Task Retrieval & Batch Loop
Reads rows (prompts/tasks) from Google Sheets and processes them one at a time using Split in Batches.

### 1.3 Per-Item Pipeline: Download → Style Analysis → Image Generation
For each row, downloads the reference image (binary), analyzes the style with Gemini, then generates a new image using an image-generation Gemini model.

### 1.4 Post-processing & Storage
Ensures the generated image is valid binary, adjusts filename/metadata, uploads to Google Drive, then updates Google Sheets with success info.

### 1.5 Error Handling, Rate Limiting, and Notifications
Uses IF checks and “continueRegularOutput” on key nodes plus Wait nodes to avoid rate limits; updates Sheets with error status and sends ntfy alerts.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Immediate Webhook Response
**Overview:** Accepts a POST webhook call and immediately responds with a JSON confirmation and permissive CORS headers, while the workflow continues execution.  
**Nodes involved:** `Webhook`, `Respond to Webhook`

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point trigger.
- **Configuration (interpreted):**
  - HTTP Method: **POST**
  - Path: uses the workflow’s generated UUID path (`d14dbf90-e292-4c61-ac70-5b7414894abe`)
  - Response mode: **Respond with “Respond to Webhook” node** (async pattern).
- **Inputs/Outputs:**
  - **Output →** `Respond to Webhook`
- **Edge cases / failures:**
  - Missing/invalid JSON body: generally not fatal unless later nodes rely on it (this workflow does not strongly depend on request body).
  - If webhook is called with OPTIONS (preflight): not handled here; CORS headers are in response node, but preflight handling may still require n8n/web server config.
- **Version notes:** TypeVersion **2.1**.

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — returns immediate HTTP response.
- **Configuration (interpreted):**
  - Respond with: **JSON**
  - Response body includes ISO timestamp: `{{ $now.toISO() }}`
  - Adds CORS headers:
    - `Access-Control-Allow-Origin: *`
    - `Access-Control-Allow-Methods: POST, GET, OPTIONS`
    - `Access-Control-Allow-Headers: Content-Type, Authorization`
- **Connections:**
  - **Input ←** `Webhook`
  - **Output →** `Read – Google Sheets – Pending Prompts` and `ntfy - Start Alert` (parallel start)
- **Edge cases / failures:**
  - If expression `$now.toISO()` fails (very unlikely), response body may error; otherwise safe.
- **Version notes:** TypeVersion **1.5**.

---

### Block 2.2 — Notifications: Start / Finish / Per-Task
**Overview:** Sends progress messages to an ntfy.sh topic at workflow start, per-task success, per-task error, and workflow completion.  
**Nodes involved:** `ntfy - Start Alert`, `ntfy - Workflow Completion Alert`, `ntfy - Task Success Alert`, `ntfy - Task Error Alert`

#### Node: ntfy - Start Alert
- **Type / role:** `n8n-nodes-base.httpRequest` — POST to ntfy topic to indicate start.
- **Configuration:**
  - URL: `https://ntfy.sh/YOUR-PREFER-NAME` (placeholder)
  - Method: POST
  - Sends JSON body containing `topic`, `message`, `title: "start"`
- **Connections:**
  - **Input ←** `Respond to Webhook`
- **Edge cases / failures:**
  - Topic placeholder not replaced → notifications go to wrong/invalid topic.
  - ntfy.sh downtime → node fails (no explicit onError set here; could stop workflow depending on n8n error settings).
- **Version notes:** TypeVersion **4.3**.

#### Node: ntfy - Workflow Completion Alert
- **Type / role:** HTTP Request to ntfy to signal workflow completion.
- **Configuration:**
  - URL: `https://ntfy.sh/YOUR-PREFER-NAME`
  - Body includes `title: "finish"`
- **Connections:**
  - **Input ←** `Split – Process Items Individually` (first output branch)
- **Edge cases:** same as above.
- **Version notes:** TypeVersion **4.3**.

#### Node: ntfy - Task Success Alert
- **Type / role:** HTTP Request to ntfy to signal one task completed.
- **Configuration:** `title: "task_completed"`
- **Connections:**
  - **Input ←** `Wait #2` (after upload, before looping back)
- **Edge cases:** same as above.
- **Version notes:** TypeVersion **4.3**.

#### Node: ntfy - Task Error Alert
- **Type / role:** HTTP Request to ntfy to signal task error.
- **Configuration:** `title: "error"`
- **Connections:**
  - **Input ←** `Update – Google Sheets – Error Status`
- **Edge cases:** same as above.
- **Version notes:** TypeVersion **4.3**.

---

### Block 2.3 — Read Tasks from Google Sheets & Iterate
**Overview:** Reads prompt/task rows from Google Sheets, then iterates them one-by-one via Split in Batches.  
**Nodes involved:** `Read – Google Sheets – Pending Prompts`, `Split – Process Items Individually`

#### Node: Read – Google Sheets – Pending Prompts
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows that represent tasks to process.
- **Configuration (interpreted):**
  - Uses **Google Sheets OAuth2** credential: “Google Sheets account”.
  - **Sheet name** and **Document ID** are currently placeholders/empty expressions in this node (`value: ""` and `documentId: "="`), meaning it likely requires user configuration to work.
  - Operation is not explicitly visible in parameters snippet; by node name it’s intended to **read pending prompts** (commonly “Read”/“Get Many”).
- **Connections:**
  - **Input ←** `Respond to Webhook`
  - **Output →** `Split – Process Items Individually`
- **Key data expectations:**
  - Columns referenced later: `referans_url`, `gorsel_id`, `ana_prompt`, `stil_prompt`, plus some status/matching fields.
- **Edge cases / failures:**
  - Missing document/sheet IDs → runtime error.
  - OAuth token expired / insufficient permission → auth error.
  - If sheet returns zero rows → Split in Batches will output “no items” and the workflow completion branch may fire depending on n8n behavior/settings.
- **Version notes:** TypeVersion **4.7**.

#### Node: Split – Process Items Individually
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterates items.
- **Configuration:**
  - Batch size not explicitly set in shown parameters (defaults typically apply). Intended usage: **one-by-one** (per sticky note).
  - `executeOnce: false` indicates it loops until items are exhausted.
- **Connections:**
  - **Input ←** `Read – Google Sheets – Pending Prompts`
  - **Output 0 →** `ntfy - Workflow Completion Alert` (this is unusual; see edge case)
  - **Output 1 →** `Download –  Reference Image`
  - Additionally loops back into itself from `Wait #3` and `Wait #4` (after success/error).
- **Important behavioral note:**
  - Split in Batches has two outputs: **“Done”** and **“Next batch”** (naming varies by UI). Here, the workflow wires:
    - One branch to *completion alert*,
    - One branch to *download reference image*.
  - If these are swapped, you might send “finish” at the wrong time.
- **Edge cases / failures:**
  - Miswired outputs → “finish” alert might fire per item or at start.
  - Empty input → completion branch fires immediately.
- **Version notes:** TypeVersion **3**.

---

### Block 2.4 — Download Reference Image (Binary Input)
**Overview:** Downloads each row’s reference image URL as binary for Gemini analysis.  
**Nodes involved:** `Download –  Reference Image`, `Wait #1`

#### Node: Download –  Reference Image
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads file as binary.
- **Configuration:**
  - URL from sheet row: `{{ $json.referans_url }}`
  - Response format: **file** (binary)
  - `onError: continueRegularOutput` → errors won’t stop workflow; downstream must check results.
- **Connections:**
  - **Input ←** `Split – Process Items Individually`
  - **Output →** `Wait #1`
- **Edge cases / failures:**
  - Invalid URL / 404 / timeout → node continues with missing binary; later IF checks should catch failures, but current analysis check is on Gemini output (not download output).
  - Large images may exceed limits; consider setting timeout/max size.
- **Version notes:** TypeVersion **4.2**.

#### Node: Wait #1
- **Type / role:** `n8n-nodes-base.wait` — pacing before Gemini analysis.
- **Configuration:** waits **2** (unit depends on node default; typically seconds).
- **Connections:**
  - **Input ←** `Download –  Reference Image`
  - **Output →** `Analyze – Gemini – Visual Style`
- **Edge cases:** minimal; if wait is too short and you still hit rate limits, increase.
- **Version notes:** TypeVersion **1.1**.

---

### Block 2.5 — Gemini Style Analysis & Validation
**Overview:** Sends the downloaded reference image to Gemini for style analysis, then checks whether the analysis result contains text before proceeding.  
**Nodes involved:** `Analyze – Gemini – Visual Style`, `If  - Check Analysis Result`

#### Node: Analyze – Gemini – Visual Style
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini multimodal analysis on image.
- **Configuration:**
  - Resource: **image**
  - Operation: **analyze**
  - Input type: **binary**
  - Binary property name: `data`
  - Model: `models/gemini-2.5-flash`
  - `simplify: false` → outputs full Gemini response structure (candidates, parts, etc.).
  - Text prompt: placeholder `YOUR PROMPT` (must be replaced with your style-analysis instruction).
  - `onError: continueRegularOutput` and `retryOnFail: false`
- **Connections:**
  - **Input ←** `Wait #1`
  - **Output →** `If  - Check Analysis Result`
- **Key output used later:**
  - The IF node checks: `$json.candidates[0].content.parts[0].text`
- **Edge cases / failures:**
  - If Gemini returns no candidates/parts (policy refusal, error, empty response), the expression may evaluate to `undefined`.
  - Missing binary `data` → Gemini node errors; because it continues regular output, downstream must detect missing analysis.
- **Version notes:** TypeVersion **1.1** (requires the n8n LangChain Gemini node package availability).

#### Node: If  - Check Analysis Result
- **Type / role:** `n8n-nodes-base.if` — validation gate.
- **Condition:**
  - Checks whether analysis text **exists**:
    - `{{ $json.candidates[0].content.parts[0].text }} exists`
- **Connections:**
  - **True →** `Create – Gemini – New Image From Style`
  - **False →** `Update – Google Sheets – Error Status`
- **Edge cases / failures:**
  - If the leftValue expression errors due to missing path, IF may treat as not existing or throw depending on strictness; here it uses strict validation options.
- **Version notes:** TypeVersion **2.3**.

---

### Block 2.6 — Gemini Image Generation & Validation
**Overview:** Generates a new image using Gemini’s image model (intended to incorporate the analyzed style) and checks that binary image output exists before continuing.  
**Nodes involved:** `Create – Gemini – New Image From Style`, `If - Check Image Generation`

#### Node: Create – Gemini – New Image From Style
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — image generation.
- **Configuration:**
  - Resource: **image**
  - Model: `models/gemini-2.5-flash-image` (labeled “Nano Banana” in cached name)
  - Prompt: placeholder `YOUR PROMPT` (must be replaced; typically should include:
    - the main prompt (`ana_prompt`)
    - style constraints derived from analysis text (`stil_prompt` or Gemini analysis output)
  - Options: `sampleCount: 1`
  - Credential: **Google Gemini(PaLM) Api account**
  - `onError: continueRegularOutput`
- **Connections:**
  - **Input ←** `If  - Check Analysis Result` (true branch)
  - **Output →** `If - Check Image Generation`
- **Edge cases / failures:**
  - If prompt not properly constructed, style consistency will be poor.
  - Policy refusal may return no binary.
- **Version notes:** TypeVersion **1.1**.

#### Node: If - Check Image Generation
- **Type / role:** IF gate verifying generation output.
- **Condition:**
  - Checks `{{ $binary.data !== undefined }}` is **true**
- **Connections:**
  - **True →** `ransform – Code – Prepare Image Binary`
  - **False →** `Update – Google Sheets – Error Status`
- **Edge cases:**
  - If binary property name differs (not `data`), this check fails.
- **Version notes:** TypeVersion **2.3**.

---

### Block 2.7 — Prepare Binary, Upload to Drive, Mark Success
**Overview:** Normalizes the generated image binary (especially filename), uploads it to Google Drive, and updates the sheet as successful, then loops to next item.  
**Nodes involved:** `ransform – Code – Prepare Image Binary`, `Upload – Google Drive – Store Generated Image`, `Wait #2`, `Update – Google Sheets – Success Status`, `Wait #3`

#### Node: ransform – Code – Prepare Image Binary
- **Type / role:** `n8n-nodes-base.code` — manipulates binary metadata and attaches JSON fields.
- **Configuration (logic):**
  - Validates that `items[0].binary.data` exists; otherwise throws: **“DALL-E binary data bulunamadı!”** (message is legacy/mislabeled; it’s Gemini, not DALL·E).
  - Pulls reference info from: `$('Download –  Reference Image').item.json`
    - `gorsel_id` defaults to `"generated"` if missing
    - builds `fileName = ${gorselId}_output.png`
  - Returns:
    - `json`: `gorsel_id`, `fileName`, `ana_prompt`, `stil_prompt`
    - `binary.data`: copies existing binary metadata and overwrites `fileName`
- **Connections:**
  - **Input ←** `If - Check Image Generation` (true)
  - **Output →** `Upload – Google Drive – Store Generated Image`
- **Edge cases / failures:**
  - If “Download – Reference Image” did not run or has no `.item.json` in current item pairing, `$('Download –  Reference Image').item.json` can be undefined or mismatched.
  - Throws hard error if binary missing (will stop unless handled by workflow-level settings).
- **Version notes:** TypeVersion **2**.

#### Node: Upload – Google Drive – Store Generated Image
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads generated binary to Drive.
- **Configuration:**
  - Drive ID and Folder ID are placeholders (must be set).
  - Output not simplified (`simplifyOutput: false`)
  - Requests additional fields via app properties UI: `webViewLink,webContentLink,id,name`
  - The “name” field is currently an expression placeholder (`name: "="`), implying it should likely be set to `{{$json.fileName}}`.
- **Connections:**
  - **Input ←** `ransform – Code – Prepare Image Binary`
  - **Output →** `Wait #2`
- **Edge cases / failures:**
  - Missing folder/drive IDs → error.
  - Service account vs OAuth permissions mismatch (Drive sharing settings).
- **Version notes:** TypeVersion **3**.

#### Node: Wait #2
- **Type / role:** Wait for pacing after upload.
- **Configuration:** amount **2**
- **Connections:**
  - **Input ←** `Upload – Google Drive – Store Generated Image`
  - **Output →** `Update – Google Sheets – Success Status` and `ntfy - Task Success Alert` (parallel)
- **Version notes:** TypeVersion **1.1**.

#### Node: Update – Google Sheets – Success Status
- **Type / role:** `n8n-nodes-base.googleSheets` — records success back to the sheet.
- **Configuration (interpreted):**
  - Authentication: **serviceAccount** (credential “Service Account account 2”)
  - Spreadsheet:
    - Document: `1WqalnqFoFdJngObmgW7HF4JIU7XcK9QnDoiExazAXLU`
    - Sheet/tab: **Prompts 2** (gid `54612697`)
  - Operation: **update**
  - Matching column: `tarih` (date) — **this is likely a mistake** if you intend to update the row you processed; normally you match by `gorsel_id` or `row_number`.
  - Mapping includes only:
    - `tarih` (expression placeholder `"="`)
    - `row_number: 0` (likely incorrect; row_number is typically read-only and should come from the input item)
- **Connections:**
  - **Input ←** `Wait #2`
  - **Output →** `Wait #3`
- **Edge cases / failures:**
  - If matching on `tarih` and it’s blank/non-unique, updates may affect the wrong row or none.
  - If `row_number` is not set correctly, update may fail.
- **Version notes:** TypeVersion **4.7**.

#### Node: Wait #3
- **Type / role:** pacing before next batch.
- **Configuration:** amount **2**
- **Connections:**
  - **Input ←** `Update – Google Sheets – Success Status`
  - **Output →** `Split – Process Items Individually` (loop)
- **Version notes:** TypeVersion **1.1**.

---

### Block 2.8 — Error Path: Update Sheet, Alert, Loop
**Overview:** If analysis or generation fails, writes an error status to Sheets, sends error notification, and loops to the next item after a short wait.  
**Nodes involved:** `Update – Google Sheets – Error Status`, `Wait #4`

#### Node: Update – Google Sheets – Error Status
- **Type / role:** Google Sheets update for failures.
- **Configuration:**
  - Operation: **update**
  - Matching column: `gorsel_id`
  - Document ID and Sheet ID are placeholders (`"="`) and must be configured.
  - Currently only maps:
    - `gorsel_id` (expression placeholder `"="`)
    - `row_number: 0` (again likely incorrect unless you truly intend row 0)
  - `onError: continueRegularOutput` (so even if it can’t update, workflow continues)
- **Connections:**
  - **Input ←** failure branches from `If  - Check Analysis Result` and `If - Check Image Generation`
  - **Output →** `Wait #4` and `ntfy - Task Error Alert` (parallel)
- **Edge cases / failures:**
  - If placeholders not set, this node errors but continues output; you may silently lose error logging.
  - No actual “hata_mesaji” (error message) mapping is set, despite schema containing it.
- **Version notes:** TypeVersion **4.7**.

#### Node: Wait #4
- **Type / role:** pacing before continuing loop after an error.
- **Configuration:** amount **2**
- **Connections:**
  - **Input ←** `Update – Google Sheets – Error Status`
  - **Output →** `Split – Process Items Individually` (loop)
- **Version notes:** TypeVersion **1.1**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry trigger (POST) | — | Respond to Webhook |  |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Immediate JSON response + CORS | Webhook | Read – Google Sheets – Pending Prompts; ntfy - Start Alert |  |
| ntfy - Start Alert | n8n-nodes-base.httpRequest | Start notification | Respond to Webhook | — | ## Monitoring & Notifications; This workflow uses ntfy.sh for real-time progress updates.; Topic: Replace the topic name ai-gorsel-uretimi100 in all HTTP Request nodes with your own unique topic.; Live Feed: Connect your dashboard or mobile app to the same ntfy topic to see live logs and status changes. |
| Read – Google Sheets – Pending Prompts | n8n-nodes-base.googleSheets | Read tasks from Sheets | Respond to Webhook | Split – Process Items Individually | ## Setup Instructions; - Google Sheets: Create a sheet with columns: gorsel_id, ana_prompt, stil_prompt, referans_url, durum.; - Credentials: Set up and connect Google Sheets, Google Drive, and Google Gemini API credentials.; - Resource IDs: Update the Google Sheets "Document ID" and Google Drive "Folder ID" in the respective nodes to match your own files.; - Webhooks: Use the Webhook URL in your dashboard or external app to trigger the process. |
| Split – Process Items Individually | n8n-nodes-base.splitInBatches | Process rows one-by-one (loop) | Read – Google Sheets – Pending Prompts; Wait #3; Wait #4 | ntfy - Workflow Completion Alert; Download –  Reference Image | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| ntfy - Workflow Completion Alert | n8n-nodes-base.httpRequest | Completion notification | Split – Process Items Individually | — | ## Monitoring & Notifications; This workflow uses ntfy.sh for real-time progress updates.; Topic: Replace the topic name ai-gorsel-uretimi100 in all HTTP Request nodes with your own unique topic.; Live Feed: Connect your dashboard or mobile app to the same ntfy topic to see live logs and status changes. |
| Download –  Reference Image | n8n-nodes-base.httpRequest | Download reference image as binary | Split – Process Items Individually | Wait #1 | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Wait #1 | n8n-nodes-base.wait | Rate limiting before analysis | Download –  Reference Image | Analyze – Gemini – Visual Style | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Analyze – Gemini – Visual Style | @n8n/n8n-nodes-langchain.googleGemini | Analyze visual style from reference image | Wait #1 | If  - Check Analysis Result | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| If  - Check Analysis Result | n8n-nodes-base.if | Validate analysis output exists | Analyze – Gemini – Visual Style | Create – Gemini – New Image From Style (true); Update – Google Sheets – Error Status (false) | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Create – Gemini – New Image From Style | @n8n/n8n-nodes-langchain.googleGemini | Generate new image in the analyzed style | If  - Check Analysis Result (true) | If - Check Image Generation | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| If - Check Image Generation | n8n-nodes-base.if | Validate generated binary exists | Create – Gemini – New Image From Style | ransform – Code – Prepare Image Binary (true); Update – Google Sheets – Error Status (false) | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| ransform – Code – Prepare Image Binary | n8n-nodes-base.code | Set filename/metadata for upload | If - Check Image Generation (true) | Upload – Google Drive – Store Generated Image | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Upload – Google Drive – Store Generated Image | n8n-nodes-base.googleDrive | Store generated image in Drive | ransform – Code – Prepare Image Binary | Wait #2 | ## Setup Instructions; - Google Sheets: Create a sheet with columns: gorsel_id, ana_prompt, stil_prompt, referans_url, durum.; - Credentials: Set up and connect Google Sheets, Google Drive, and Google Gemini API credentials.; - Resource IDs: Update the Google Sheets "Document ID" and Google Drive "Folder ID" in the respective nodes to match your own files.; - Webhooks: Use the Webhook URL in your dashboard or external app to trigger the process. |
| Wait #2 | n8n-nodes-base.wait | Rate limiting after upload | Upload – Google Drive – Store Generated Image | Update – Google Sheets – Success Status; ntfy - Task Success Alert | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| ntfy - Task Success Alert | n8n-nodes-base.httpRequest | Per-task success notification | Wait #2 | — | ## Monitoring & Notifications; This workflow uses ntfy.sh for real-time progress updates.; Topic: Replace the topic name ai-gorsel-uretimi100 in all HTTP Request nodes with your own unique topic.; Live Feed: Connect your dashboard or mobile app to the same ntfy topic to see live logs and status changes. |
| Update – Google Sheets – Success Status | n8n-nodes-base.googleSheets | Mark task success in Sheets | Wait #2 | Wait #3 | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Wait #3 | n8n-nodes-base.wait | Pacing before next item | Update – Google Sheets – Success Status | Split – Process Items Individually | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Update – Google Sheets – Error Status | n8n-nodes-base.googleSheets | Record error status in Sheets | If  - Check Analysis Result (false); If - Check Image Generation (false) | Wait #4; ntfy - Task Error Alert | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| Wait #4 | n8n-nodes-base.wait | Pacing after error | Update – Google Sheets – Error Status | Split – Process Items Individually | ## Logic & Error Handling; - Rate Limiting: 'Wait' nodes are strategically placed to prevent hitting API rate limits during bulk processing.; - Looping: The workflow processes images one by one to ensure stability.; - Error Management: If an analysis or generation fails, the "Update – Google Sheets – Error Status" node automatically records the failure reason in your spreadsheet. |
| ntfy - Task Error Alert | n8n-nodes-base.httpRequest | Per-task error notification | Update – Google Sheets – Error Status | — | ## Monitoring & Notifications; This workflow uses ntfy.sh for real-time progress updates.; Topic: Replace the topic name ai-gorsel-uretimi100 in all HTTP Request nodes with your own unique topic.; Live Feed: Connect your dashboard or mobile app to the same ntfy topic to see live logs and status changes. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Generate consistent styled images from references with Google Gemini and Sheets** (or your preferred name).
   - Set timezone if needed (this workflow uses **Europe/Istanbul**).

2. **Add a Webhook trigger**
   - Node: **Webhook**
   - Method: **POST**
   - Response: **Using “Respond to Webhook” node**
   - Copy the production webhook URL for your external caller.

3. **Add “Respond to Webhook”**
   - Respond with: **JSON**
   - Add CORS headers:
     - `Access-Control-Allow-Origin = *`
     - `Access-Control-Allow-Methods = POST, GET, OPTIONS`
     - `Access-Control-Allow-Headers = Content-Type, Authorization`
   - Response body example:
     - `status: "success"`
     - `message: "Workflow başlatıldı"`
     - `timestamp: {{$now.toISO()}}`
   - Connect: **Webhook → Respond to Webhook**

4. **Add ntfy start notification**
   - Node: **HTTP Request**
   - Method: POST
   - URL: `https://ntfy.sh/<YOUR_TOPIC>`
   - Body: JSON with `topic`, `message`, `title: "start"`
   - Connect: **Respond to Webhook → ntfy - Start Alert**

5. **Add Google Sheets “Read pending prompts”**
   - Node: **Google Sheets**
   - Credentials: configure **Google Sheets OAuth2**
   - Select your **Spreadsheet (Document ID)** and **Sheet/Tab**
   - Configure the read operation to return the rows you want to process (commonly all rows where `durum = "Pending"`; implement via filters if you use a “Read with filter” approach, or read all then filter with an IF node—this JSON implies “pending” by naming but doesn’t show filtering).
   - Ensure your sheet has (at minimum) columns:
     - `gorsel_id`, `ana_prompt`, `stil_prompt`, `referans_url`, `durum`
   - Connect: **Respond to Webhook → Read – Google Sheets – Pending Prompts**

6. **Add Split in Batches**
   - Node: **Split In Batches**
   - Set **Batch Size = 1** (recommended for stability)
   - Connect: **Read – Google Sheets – Pending Prompts → Split – Process Items Individually**
   - Wire outputs carefully:
     - **Done output →** `ntfy - Workflow Completion Alert`
     - **Next batch output →** `Download – Reference Image`

7. **Add ntfy completion notification**
   - Node: **HTTP Request**
   - POST to `https://ntfy.sh/<YOUR_TOPIC>`
   - Body JSON with `title: "finish"`
   - Connect: **Split Done → ntfy - Workflow Completion Alert**

8. **Add “Download reference image”**
   - Node: **HTTP Request**
   - URL: `{{$json.referans_url}}`
   - Response: **File** (binary)
   - Set **On Error: Continue (continueRegularOutput)**
   - Connect: **Split Next → Download – Reference Image**

9. **Add Wait before analysis**
   - Node: **Wait**
   - Amount: `2` (seconds recommended; adjust for rate limits)
   - Connect: **Download → Wait #1**

10. **Add Gemini Analyze (Visual Style)**
   - Node: **Google Gemini (LangChain)**
   - Resource: **Image**
   - Operation: **Analyze**
   - Model: `models/gemini-2.5-flash`
   - Input type: **Binary**
   - Binary property: `data`
   - Prompt: write your style analysis instruction (replace `YOUR PROMPT`). Example intent:
     - “Analyze the reference image style (palette, lighting, composition, medium) and return a concise style guide…”
   - Set **On Error: Continue**
   - Connect: **Wait #1 → Analyze – Gemini – Visual Style**
   - **Credential:** configure Gemini/PaLM API credential as required by your node version.

11. **Add IF to validate analysis**
   - Node: **IF**
   - Condition: string **exists** on:
     - `{{$json.candidates[0].content.parts[0].text}}`
   - Connect: **Analyze → IF (analysis check)**

12. **Add Gemini Image Generation**
   - Node: **Google Gemini (LangChain)**
   - Resource: **Image**
   - Model: `models/gemini-2.5-flash-image`
   - Prompt: replace `YOUR PROMPT`; typically combine:
     - the user’s main prompt: `{{$json.ana_prompt}}`
     - style constraints: either `{{$json.stil_prompt}}` from the sheet and/or the analysis text from the previous node.
   - Options: `sampleCount = 1`
   - On Error: Continue
   - Connect: **IF true → Create – Gemini – New Image From Style**
   - Connect: **IF false → Update – Google Sheets – Error Status** (create this node in step 18)

13. **Add IF to validate generated image binary**
   - Node: **IF**
   - Condition: boolean true on:
     - `{{$binary.data !== undefined}}`
   - Connect: **Create – Gemini → IF (image check)**
   - False branch → error status node (step 18)

14. **Add Code node to set filename + JSON metadata**
   - Node: **Code**
   - Paste logic equivalent to:
     - Ensure `$input.all()[0].binary.data` exists
     - Read `gorsel_id`, `ana_prompt`, `stil_prompt` from the current item (or from the download node if you maintain pairing)
     - Set `binary.data.fileName = `${gorsel_id}_output.png``
   - Connect: **Image check true → Code**

15. **Add Google Drive upload**
   - Node: **Google Drive**
   - Configure credentials (OAuth2 or Service Account depending on your environment)
   - Set **Folder ID** (target folder)
   - Set file name to `{{$json.fileName}}`
   - Ensure the node uploads the **binary `data`**
   - Request output fields: `webViewLink, webContentLink, id, name` (optional but useful)
   - Connect: **Code → Upload – Google Drive**

16. **Add Wait after upload**
   - Node: **Wait** amount `2`
   - Connect: **Drive Upload → Wait #2**

17. **Add Success update to Google Sheets**
   - Node: **Google Sheets** (Update)
   - Prefer matching by a stable key:
     - `gorsel_id` or `row_number` (recommended) rather than `tarih`
   - Map fields typically:
     - `durum = "Success"`
     - `gorsel_url = <Drive link (webViewLink or webContentLink)>`
     - `tarih = {{$now.toISO()}}`
     - `hata_mesaji = ""`
   - Connect: **Wait #2 → Update – Google Sheets – Success Status**

18. **Add Error update to Google Sheets**
   - Node: **Google Sheets** (Update)
   - Configure Document ID + Sheet
   - Match by `gorsel_id` (as in the JSON)
   - Map fields typically:
     - `durum = "Error"`
     - `hata_mesaji = <some message>` (from Gemini response, HTTP error, or a static label)
     - `tarih = {{$now.toISO()}}`
   - Set **On Error: Continue**
   - Connect both failure branches to this node:
     - **Analysis IF false → Error update**
     - **Image generation IF false → Error update**

19. **Add ntfy per-task success and error notifications**
   - Success notification:
     - HTTP Request POST to `https://ntfy.sh/<YOUR_TOPIC>`, `title: "task_completed"`
     - Connect: **Wait #2 → ntfy - Task Success Alert**
   - Error notification:
     - HTTP Request POST to `https://ntfy.sh/<YOUR_TOPIC>`, `title: "error"`
     - Connect: **Error update → ntfy - Task Error Alert**

20. **Add Wait nodes for looping**
   - After success update: **Wait #3** (2s) → connect back to **Split in Batches**
   - After error update: **Wait #4** (2s) → connect back to **Split in Batches**

21. **Credentials checklist**
   - **Google Sheets OAuth2**: used by the “Read pending prompts” node.
   - **Google API Service Account** (or OAuth2): used by the “Update success/error” nodes in the JSON; keep consistent with your Drive/Sheets sharing model.
   - **Google Gemini (PaLM) API**: required for both Gemini nodes (analyze + generate).
   - Ensure your Sheet and Drive folder are shared with the service account email if using service accounts.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Setup Instructions: Create a sheet with columns: gorsel_id, ana_prompt, stil_prompt, referans_url, durum. Set up credentials. Update Document ID and Folder ID. Use the Webhook URL to trigger.” | Applies to Google Sheets/Drive/Gemini nodes and general setup. |
| “Monitoring & Notifications: This workflow uses ntfy.sh… Replace topic ai-gorsel-uretimi100 with your own unique topic… Connect dashboard/mobile app to same topic.” | ntfy nodes (`https://ntfy.sh/<topic>`). |
| “Logic & Error Handling: Wait nodes prevent API rate limits; process images one by one; error status node records failure reason.” | Wait + IF + error update loop. |
| Spreadsheet link visible in node cache: `https://docs.google.com/spreadsheets/d/1WqalnqFoFdJngObmgW7HF4JIU7XcK9QnDoiExazAXLU/edit#gid=54612697` | Reference shown in cached sheet selection for “Prompts 2”. |

