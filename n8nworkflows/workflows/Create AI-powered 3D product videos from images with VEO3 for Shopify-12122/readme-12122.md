Create AI-powered 3D product videos from images with VEO3 for Shopify

https://n8nworkflows.xyz/workflows/create-ai-powered-3d-product-videos-from-images-with-veo3-for-shopify-12122


# Create AI-powered 3D product videos from images with VEO3 for Shopify

## 1. Workflow Overview

**Purpose:** This n8n workflow collects a product image via an n8n Form, stores assets in Google Drive, removes the image background, logs product data into Google Sheets, uses OpenAI/LangChain to analyze the image and generate a structured prompt, calls a **VEO3** video generation API to create an AI “3D product” style video, polls for completion, downloads the result, updates the sheet with the final video link, and then triggers downstream notifications (Rapiwa, Gmail, Discord).

**Typical use case:** Shopify merchants (or product teams) who want to automatically turn product images into short AI-generated product videos and keep a tracking sheet of inputs/outputs.

### Logical blocks
1. **1.1 Intake & Workspace Setup (Form → Drive folder + permissions)**
2. **1.2 Image Retrieval + Background Removal**
3. **1.3 Spreadsheet Logging (new product row + storing remove-bg URL)**
4. **1.4 Image Analysis + Prompt/Spec Generation (OpenAI + Agent + Structured Parser)**
5. **1.5 Video Generation with VEO3 + Status Polling**
6. **1.6 Video Download + Sheet Update**
7. **1.7 Notifications / Post-processing**

> Note: Most nodes have empty `parameters` in the provided JSON export, so configuration below is inferred from node names, connections, and typical implementation patterns. When reproducing, you will need to fill in missing specifics (Drive folder IDs, Sheets document/tab, API endpoints, auth headers, etc.).

---

## 2. Block-by-Block Analysis

### 2.1 Intake & Workspace Setup (Form → Drive folder + permissions)

**Overview:** Receives user input via an n8n Form and prepares a Google Drive folder for the run (create folder and share/grant access).

**Nodes involved:**  
- On Form Submission  
- Create Folder  
- Give Access to folder  

#### Node: **On Form Submission**
- **Type / role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — workflow entry point.
- **Configuration (interpreted):** Should define the form fields (e.g., product name, product URL/handle, image upload or image URL, email).
- **Connections:**  
  - Output → **Create Folder**
- **Edge cases / failures:**  
  - Missing required form fields; file upload size limits; webhook disabled; form not published.
- **Version notes:** typeVersion `2.2`.

#### Node: **Create Folder**
- **Type / role:** `Google Drive` — creates a folder to store run assets.
- **Configuration (interpreted):**
  - Operation likely “Create Folder”.
  - Parent folder ID likely a fixed “workspace” directory.
  - Folder name typically derived from form fields (e.g., product name + timestamp).
- **Expressions likely used:** `{{$now}}`, `{{$json.<field>}}` for naming.
- **Connections:**  
  - Input: **On Form Submission**  
  - Output → **Give Access to folder**
- **Failure modes:** Drive auth expired; missing permission to create in parent; invalid parent folder ID.
- **Version notes:** typeVersion `3`.

#### Node: **Give Access to folder**
- **Type / role:** `Google Drive` — shares folder or sets permissions.
- **Configuration (interpreted):**
  - Operation: “Share / Add permission”.
  - Role: viewer/commenter/editor depending on needs.
  - Target: email(s) from the form or a fixed team email.
- **Connections:**  
  - Input: **Create Folder**  
  - Output → **Get Image File**
- **Failure modes:** Invalid email; domain sharing restrictions; permission propagation delays.
- **Version notes:** typeVersion `3`.

---

### 2.2 Image Retrieval + Background Removal

**Overview:** Fetches/normalizes the submitted image file, then calls an external service to remove background, then uploads the processed result to Drive.

**Nodes involved:**  
- Get Image File  
- Remove Image Background  
- Upload Remove BG image  

#### Node: **Get Image File**
- **Type / role:** `Code` (`n8n-nodes-base.code`) — converts form input into a usable binary/image payload (download from URL, decode, rename, etc.).
- **Configuration (interpreted):**
  - Likely reads either:
    - binary data from the form upload, or
    - an image URL field and downloads it.
  - Normalizes output to `binary` (e.g., `binary.data`) for subsequent HTTP/Drive operations.
- **Key settings:**  
  - `executeOnce: true` (runs once per workflow execution)  
  - `retryOnFail: true`  
  - `alwaysOutputData: true` (ensures downstream nodes receive an item even if code returns empty JSON)
- **Connections:**  
  - Input: **Give Access to folder**  
  - Output (fan-out) → **Remove Image Background** AND **Insert New Product**
- **Failure modes:** malformed URL; download blocked; missing binary property; code exceptions.
- **Version notes:** typeVersion `2`.

#### Node: **Remove Image Background**
- **Type / role:** `HTTP Request` — calls a background removal API (e.g., remove.bg or similar).
- **Configuration (interpreted):**
  - POST request with image binary as multipart/form-data or base64.
  - Requires API key header (e.g., `X-Api-Key`) or bearer token.
  - Response is likely an image binary (PNG with transparency).
- **Connections:**  
  - Input: **Get Image File**  
  - Output → **Upload Remove BG image**
- **Failure modes:** API quota exceeded; 4xx due to missing key; payload too large; timeout; returns JSON error instead of image.
- **Version notes:** typeVersion `4.2`.

#### Node: **Upload Remove BG image**
- **Type / role:** `Google Drive` — uploads the processed image.
- **Configuration (interpreted):**
  - Operation: “Upload”.
  - Destination folder: the folder created earlier (Drive folder ID from **Create Folder** output, typically passed through items or via expressions).
  - File name: derived (e.g., `<product>-nobg.png`).
  - Likely sets “Share link” or retrieves webContentLink/webViewLink.
- **Connections:**  
  - Input: **Remove Image Background**  
  - Output → **Get row(s) in sheet**
- **Failure modes:** missing binary data; insufficient Drive permissions; name collisions; upload size limits.
- **Version notes:** typeVersion `3`.

---

### 2.3 Spreadsheet Logging (new product row + storing remove-bg URL)

**Overview:** Writes a new “product” entry to Google Sheets and then updates that row with the remove-background image URL for traceability.

**Nodes involved:**  
- Insert New Product  
- Get row(s) in sheet  
- Update Remove BG URL  

#### Node: **Insert New Product**
- **Type / role:** `Google Sheets` — inserts a new row with product metadata.
- **Configuration (interpreted):**
  - Operation: “Append / Add row”.
  - Columns may include: product title, original image link, created folder link, timestamp, status.
  - Uses values from **Get Image File** and form submission fields.
- **Connections:**  
  - Input: **Get Image File**  
  - Output: not connected downstream (but likely should be used to capture the inserted row ID/row number).
- **Failure modes:** wrong sheet/tab; permission denied; data type mismatch; header mismatch.
- **Version notes:** typeVersion `4.5`.

#### Node: **Get row(s) in sheet**
- **Type / role:** `Google Sheets` — finds the correct row to update (the one inserted).
- **Configuration (interpreted):**
  - Operation: “Lookup / Read rows” with a filter (e.g., by product ID, timestamp, filename).
  - Because **Insert New Product** is not connected forward, this node likely re-queries using a unique key from earlier nodes.
- **Connections:**  
  - Input: **Upload Remove BG image**  
  - Output → **Update Remove BG URL**
- **Failure modes:** multiple matching rows; no rows returned; filter expression wrong.
- **Version notes:** typeVersion `4.7`.

#### Node: **Update Remove BG URL**
- **Type / role:** `Google Sheets` — updates the located row with the Drive link to the processed image.
- **Configuration (interpreted):**
  - Operation: “Update row”.
  - Sets column like `remove_bg_url` to the uploaded file’s share link.
- **Connections:**  
  - Input: **Get row(s) in sheet**  
  - Output → **Analyze image**
- **Failure modes:** missing row identifier; protected range; rate limits.
- **Version notes:** typeVersion `4.5`.

---

### 2.4 Image Analysis + Prompt/Spec Generation (OpenAI + Agent + Structured Parser)

**Overview:** Analyzes the (background-removed) product image and generates a structured set of instructions/prompts that will be used to generate a video.

**Nodes involved:**  
- Analyze image  
- AI Agent  
- OpenAI  
- Think  
- Parser Output  

#### Node: **Analyze image**
- **Type / role:** `OpenAI` (LangChain) (`@n8n/n8n-nodes-langchain.openAi`) — performs vision analysis or text generation based on the image.
- **Configuration (interpreted):**
  - Likely uses an OpenAI vision-capable model to describe the product, materials, color, category, camera suggestions, etc.
  - Takes input image URL or binary; in n8n this often requires mapping to the node’s expected “image” field.
- **Connections:**  
  - Input: **Update Remove BG URL**  
  - Output → **AI Agent**
- **Failure modes:** unsupported image format; model not vision-enabled; token limits; invalid OpenAI credentials.
- **Version notes:** typeVersion `1.8`.

#### Node: **AI Agent**
- **Type / role:** `LangChain Agent` — orchestrates tools/model/output parsing to produce final structured payload for VEO3 generation.
- **Configuration (interpreted):**
  - Receives analysis text (and possibly metadata) and decides what to generate next (prompt, negative prompt, duration, aspect ratio, camera motion, background).
  - Uses:
    - **OpenAI** as the chat model,
    - **Think** as an internal reasoning tool (non-output),
    - **Parser Output** to enforce structured JSON output for downstream API call.
- **Connections:**  
  - Inputs:
    - Main: **Analyze image**
    - `ai_languageModel`: **OpenAI**
    - `ai_tool`: **Think**
    - `ai_outputParser`: **Parser Output**
  - Output (main) → **Generation Video using VEO3**
- **Failure modes:** parser schema mismatch; agent returns non-JSON; prompt injection via form fields; hallucinated parameter values not supported by VEO3.
- **Version notes:** typeVersion `2.2`.

#### Node: **OpenAI**
- **Type / role:** `LM Chat OpenAI` — provides the chat model for the agent.
- **Configuration (interpreted):**
  - Model selection (e.g., GPT-4.1 / GPT-4o / similar), temperature, max tokens.
  - Requires OpenAI credentials in n8n.
- **Connections:**  
  - Output (`ai_languageModel`) → **AI Agent**
- **Failure modes:** auth; model unavailable; rate limits; organization restrictions.
- **Version notes:** typeVersion `1.2`.

#### Node: **Think**
- **Type / role:** `toolThink` — LangChain “thinking” tool for intermediate reasoning.
- **Configuration (interpreted):** Typically no external config; used by agent to structure its steps.
- **Connections:**  
  - Output (`ai_tool`) → **AI Agent**
- **Failure modes:** none typical; but can increase latency/cost if misused.
- **Version notes:** typeVersion `1.1`.

#### Node: **Parser Output**
- **Type / role:** `Structured Output Parser` — enforces schema (JSON fields) returned by the agent.
- **Configuration (interpreted):**
  - Should define a schema like:
    - `prompt`, `negative_prompt`, `duration_seconds`, `style`, `camera_moves`, `seed`, etc.
- **Connections:**  
  - Output (`ai_outputParser`) → **AI Agent**
- **Failure modes:** strict parsing errors if agent output deviates; missing required fields.
- **Version notes:** typeVersion `1.2`.

---

### 2.5 Video Generation with VEO3 + Status Polling

**Overview:** Sends the structured generation request to the VEO3 endpoint, then checks status; if not complete, waits 20 seconds and checks again until complete.

**Nodes involved:**  
- Generation Video using VEO3  
- Check Video Status  
- if  
- Wait 20s  

#### Node: **Generation Video using VEO3**
- **Type / role:** `HTTP Request` — triggers video generation job.
- **Configuration (interpreted):**
  - POST to a VEO3 generation endpoint with payload created by the agent.
  - Response likely includes a `job_id` / `task_id`.
- **Connections:**  
  - Input: **AI Agent**  
  - Output → **Check Video Status**
- **Failure modes:** invalid payload; auth failure; job creation returns 202 with missing ID; rate limits.
- **Version notes:** typeVersion `4.2`.

#### Node: **Check Video Status**
- **Type / role:** `HTTP Request` — polls job status using job/task ID.
- **Configuration (interpreted):**
  - GET/POST to status endpoint: `.../status?job_id=...`
  - Expects fields like `status: queued|running|completed|failed` and possibly a `video_url`.
- **Connections:**  
  - Inputs: **Generation Video using VEO3** and **Wait 20s** (loop back)
  - Output → **if**
- **Failure modes:** job ID not found; transient 5xx; long-running job; inconsistent status responses.
- **Version notes:** typeVersion `4.2`.

#### Node: **if**
- **Type / role:** `IF` — routes based on status check result.
- **Configuration (interpreted):**
  - Condition likely: `status == "completed"` (true branch) else wait (false branch).
- **Connections:**  
  - Input: **Check Video Status**
  - Output 1 (true) → **Download Video**
  - Output 2 (false) → **Wait 20s**
- **Failure modes:** expression errors if `status` missing; never-ending loop if completion never happens; missing handling for `failed`.
- **Version notes:** typeVersion `2.2`.

#### Node: **Wait 20s**
- **Type / role:** `Wait` — delays polling.
- **Configuration (interpreted):** fixed 20 seconds.
- **Connections:**  
  - Input: **if** (false branch)
  - Output → **Check Video Status**
- **Failure modes:** large backlog if many runs; executions held open; may hit execution time limits on some hosting plans.
- **Version notes:** typeVersion `1.1`.

---

### 2.6 Video Download + Sheet Update

**Overview:** Downloads the completed video, then updates the original spreadsheet row with the final video link (and possibly stores the video in Drive, though no Drive upload node is present here).

**Nodes involved:**  
- Download Video  
- Update Video Link  

#### Node: **Download Video**
- **Type / role:** `HTTP Request` — fetches the generated video file.
- **Configuration (interpreted):**
  - GET to `video_url` returned by status endpoint.
  - Response handled as binary (mp4).
- **Connections:**  
  - Input: **if** (true branch)
  - Output → **Update Video Link**
- **Failure modes:** signed URL expired; large file download timeouts; binary handling misconfigured.
- **Version notes:** typeVersion `4.2`.

#### Node: **Update Video Link**
- **Type / role:** `Google Sheets` — stores the output reference.
- **Configuration (interpreted):**
  - Operation: “Update row”.
  - Column like `video_url` or `video_drive_link`.
  - If you intend to store in Drive, you’d normally add an “Upload video to Drive” node; currently this workflow only “downloads” then updates a link (likely the remote URL).
- **Connections:**  
  - Input: **Download Video**
  - Output → **Do nothing**
- **Failure modes:** cannot find row; concurrent edits; sheet permissions.
- **Version notes:** typeVersion `4.5`.

---

### 2.7 Notifications / Post-processing

**Overview:** After updating the sheet, the workflow fans out to 3 notification/integration nodes. A NoOp is used as a visual junction.

**Nodes involved:**  
- Do nothing  
- Rapiwa  
- Send a message (Gmail)  
- Post on message (Discord)  

#### Node: **Do nothing**
- **Type / role:** `NoOp` — aggregator/junction node.
- **Configuration:** none.
- **Connections:**  
  - Input: **Update Video Link**
  - Output (fan-out) → **Rapiwa**, **Send a message**, **Post on message**
- **Failure modes:** none.
- **Version notes:** typeVersion `1`.

#### Node: **Rapiwa**
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — custom/community node (unknown service from JSON alone).
- **Configuration (interpreted):**
  - Likely sends WhatsApp/SMS/marketing automation message, or triggers an external system.
  - Requires Rapiwa credentials/config.
- **Connections:**  
  - Input: **Do nothing**
  - Output: none
- **Failure modes:** missing custom node installation; credential/auth errors; API downtime.
- **Version notes:** typeVersion `1`.
- **Special requirement:** This node type must be installed on your n8n instance (community node).

#### Node: **Send a message**
- **Type / role:** `Gmail` — sends an email notification.
- **Configuration (interpreted):**
  - To: likely email from form submission.
  - Subject/body includes product + video link.
  - Requires Google OAuth2 credentials for Gmail scope.
- **Connections:**  
  - Input: **Do nothing**
  - Output: none
- **Failure modes:** OAuth expired; Gmail API not enabled; rate limits; invalid recipient.
- **Version notes:** typeVersion `2.1`.

#### Node: **Post on message**
- **Type / role:** `Discord` — posts a message to a channel.
- **Configuration (interpreted):**
  - Uses Discord webhook or bot token.
  - Message includes links (remove-bg image, video URL, sheet row link).
- **Connections:**  
  - Input: **Do nothing**
  - Output: none
- **Failure modes:** invalid webhook URL; 429 rate limit; formatting issues.
- **Version notes:** typeVersion `2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On Form Submission | formTrigger | Entry point (collect product info/image) | — | Create Folder |  |
| Create Folder | googleDrive | Create per-run Drive folder | On Form Submission | Give Access to folder |  |
| Give Access to folder | googleDrive | Share/grant permissions on folder | Create Folder | Get Image File |  |
| Get Image File | code | Normalize/download image into binary + metadata | Give Access to folder | Remove Image Background; Insert New Product |  |
| Remove Image Background | httpRequest | Call background-removal API | Get Image File | Upload Remove BG image |  |
| Upload Remove BG image | googleDrive | Upload processed (no-bg) image to Drive | Remove Image Background | Get row(s) in sheet |  |
| Insert New Product | googleSheets | Append new product row | Get Image File | — |  |
| Get row(s) in sheet | googleSheets | Find the row to update | Upload Remove BG image | Update Remove BG URL |  |
| Update Remove BG URL | googleSheets | Update row with no-bg image URL | Get row(s) in sheet | Analyze image |  |
| Analyze image | langchain.openAi | Vision/text analysis of product image | Update Remove BG URL | AI Agent |  |
| AI Agent | langchain.agent | Produce structured VEO3 generation request | Analyze image (+ OpenAI/Think/Parser) | Generation Video using VEO3 |  |
| OpenAI | lmChatOpenAi | Chat model for agent | — | AI Agent |  |
| Think | toolThink | Agent tool (internal reasoning) | — | AI Agent |  |
| Parser Output | outputParserStructured | Enforce structured JSON output | — | AI Agent |  |
| Generation Video using VEO3 | httpRequest | Create video generation job | AI Agent | Check Video Status |  |
| Check Video Status | httpRequest | Poll job status | Generation Video using VEO3; Wait 20s | if |  |
| if | if | Route: completed vs wait/poll again | Check Video Status | Download Video; Wait 20s |  |
| Wait 20s | wait | Delay between polls | if (false) | Check Video Status |  |
| Download Video | httpRequest | Download final video binary | if (true) | Update Video Link |  |
| Update Video Link | googleSheets | Update row with final video link | Download Video | Do nothing |  |
| Do nothing | noOp | Fan-out junction for notifications | Update Video Link | Rapiwa; Send a message; Post on message |  |
| Rapiwa | rapiwa | Custom notification/integration | Do nothing | — |  |
| Send a message | gmail | Email notification | Do nothing | — |  |
| Post on message | discord | Discord notification | Do nothing | — |  |
| Sticky Note | stickyNote | Comment (empty) | — | — |  |
| Sticky Note1 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note2 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note3 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note4 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note5 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note7 | stickyNote | Comment (empty) | — | — |  |

> All sticky notes have empty `content` in the JSON, so there is nothing to replicate in the “Sticky Note” column.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Shopify 3D Product Video Maker from Images Using VEO3* (or the provided title).
2. **Add “On Form Submission” (Form Trigger)**  
   - Define form fields such as:
     - `product_name` (text, required)
     - `image` (file upload) *or* `image_url` (url)
     - `email` (text/email)
     - optional: `shopify_handle`, `notes`
   - Save the form and note the public form URL for testing.

3. **Add “Create Folder” (Google Drive)**  
   - Credentials: Google Drive OAuth2 with permission to create folders.  
   - Operation: **Create Folder**  
   - Parent Folder: choose a fixed “Runs” folder.  
   - Folder Name: use an expression like: `{{$json.product_name}} - {{$now}}`  
   - Connect: **On Form Submission → Create Folder**

4. **Add “Give Access to folder” (Google Drive)**  
   - Operation: **Share / Add Permission** (exact label depends on n8n Drive node options)  
   - File/Folder: use the folder ID returned by **Create Folder**.  
   - Grantee: `{{$json.email}}` or a fixed team email.  
   - Role: Viewer/Editor as desired.  
   - Connect: **Create Folder → Give Access to folder**

5. **Add “Get Image File” (Code)**  
   - Implement logic to output one item containing:
     - `json` fields (product_name, email, etc.)
     - `binary.data` containing the image
   - If using `image_url`, download it (via `this.helpers.httpRequest`) and attach to `binary`.
   - If using form upload, map the uploaded binary through.
   - Enable:
     - **Retry on fail**
     - **Always output data**
   - Connect: **Give Access to folder → Get Image File**

6. **Add “Remove Image Background” (HTTP Request)**  
   - Configure to call your remove-bg provider:
     - Method: POST
     - Authentication: header API key / bearer token
     - Send image as multipart/form-data (binary) or base64 per provider requirements
     - Response: **File/Binary**
   - Connect: **Get Image File → Remove Image Background**

7. **Add “Upload Remove BG image” (Google Drive)**  
   - Operation: Upload
   - Binary property: `data` (or whatever you set)
   - File name: `{{$json.product_name}}-nobg.png`
   - Folder ID: from **Create Folder** output (ensure it is still available in the item; you may need to merge data or pass folder id along in JSON).
   - Connect: **Remove Image Background → Upload Remove BG image**

8. **Add “Insert New Product” (Google Sheets)**  
   - Credentials: Google Sheets OAuth2.
   - Operation: Append/Add row
   - Spreadsheet + Sheet tab: pick your tracking sheet.
   - Map columns from the form and/or image metadata.
   - Connect: **Get Image File → Insert New Product**  
   - (Optional but recommended) Capture the inserted row ID/row number for later updates.

9. **Add “Get row(s) in sheet” (Google Sheets)**  
   - Operation: Lookup/Read rows
   - Filter: use a unique key (e.g., timestamp, product_name + execution ID) that matches the inserted row.
   - Connect: **Upload Remove BG image → Get row(s) in sheet**

10. **Add “Update Remove BG URL” (Google Sheets)**  
   - Operation: Update row
   - Row identifier: from **Get row(s) in sheet**
   - Set `remove_bg_url` to the Drive share link (from **Upload Remove BG image** output).
   - Connect: **Get row(s) in sheet → Update Remove BG URL**

11. **Add “Analyze image” (OpenAI / LangChain OpenAI node)**  
   - Credentials: OpenAI API key.
   - Configure to analyze the remove-bg image:
     - Provide image URL (Drive public link) or binary depending on node capability.
     - Prompt: “Describe the product, materials, colors, category, and propose a short cinematic 3D product video concept…”
   - Connect: **Update Remove BG URL → Analyze image**

12. **Add AI orchestration nodes**
   - **OpenAI (LM Chat OpenAI)**: choose a model and set temperature.
   - **Parser Output (Structured Output Parser)**: define required JSON schema for VEO3 request fields (prompt, duration, aspect ratio, etc.).
   - **Think (toolThink)**: add as tool.
   - **AI Agent**:
     - Attach:
       - `OpenAI` to the agent’s **Language Model** input
       - `Parser Output` to the agent’s **Output Parser**
       - `Think` to the agent’s **Tools**
     - Main input comes from **Analyze image**
   - Connect:  
     - **Analyze image → AI Agent**  
     - **OpenAI → AI Agent (ai_languageModel)**  
     - **Parser Output → AI Agent (ai_outputParser)**  
     - **Think → AI Agent (ai_tool)**

13. **Add “Generation Video using VEO3” (HTTP Request)**  
   - Method: POST
   - URL: your VEO3 generation endpoint
   - Auth: API key/bearer
   - Body: map fields from **AI Agent** structured output
   - Expect response contains `job_id`
   - Connect: **AI Agent → Generation Video using VEO3**

14. **Add polling loop**
   - **Check Video Status (HTTP Request)**: call status endpoint using `job_id`
   - **if**: condition `status == "completed"` (and ideally handle `"failed"`)
   - **Wait 20s**: fixed wait
   - Connect:  
     - **Generation Video using VEO3 → Check Video Status**  
     - **Check Video Status → if**  
     - **if (false) → Wait 20s → Check Video Status** (loop)  
     - **if (true) → Download Video**

15. **Add “Download Video” (HTTP Request)**
   - GET the `video_url` from status response
   - Response: binary (mp4)
   - Connect: **if (true) → Download Video**

16. **Add “Update Video Link” (Google Sheets)**
   - Operation: Update row (same row found earlier; you may need to re-lookup or carry row id forward)
   - Set `video_url` to the final hosted URL (or upload to Drive first if preferred)
   - Connect: **Download Video → Update Video Link**

17. **Add “Do nothing” (NoOp)** and notifications
   - Connect: **Update Video Link → Do nothing**
   - Add and connect in parallel:
     - **Rapiwa** (requires installing the community node + credentials)
     - **Gmail** “Send a message” (OAuth2; compose message with video link)
     - **Discord** “Post on message” (webhook URL; post summary)

18. **Credentials checklist**
   - Google Drive OAuth2 (Drive API enabled)
   - Google Sheets OAuth2 (Sheets API enabled)
   - Gmail OAuth2 (Gmail API enabled) if emailing
   - OpenAI API key
   - Remove-bg provider API key
   - VEO3 endpoint credentials (API key/bearer)
   - Discord webhook (or bot token)
   - Rapiwa credentials + installed node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow provided is inactive (`active: false`). | Enable after credentials and endpoints are configured. |
| Sticky notes exist but their contents are empty in the exported JSON. | No additional embedded documentation was provided in-notes. |
| Polling loop currently has no explicit “failed” branch. | Consider adding handling when status == `failed` to stop looping and notify. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.