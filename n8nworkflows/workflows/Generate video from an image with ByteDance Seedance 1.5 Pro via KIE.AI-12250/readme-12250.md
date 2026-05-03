Generate video from an image with ByteDance Seedance 1.5 Pro via KIE.AI

https://n8nworkflows.xyz/workflows/generate-video-from-an-image-with-bytedance-seedance-1-5-pro-via-kie-ai-12250


# Generate video from an image with ByteDance Seedance 1.5 Pro via KIE.AI

## 1. Workflow Overview

**Purpose:** This n8n workflow generates a short video from a single image using **ByteDance Seedance 1.5 Pro** via the **KIE.AI** API. It submits a generation task, then **polls** the task status every few seconds until it succeeds or fails, extracts the resulting video URL(s), and downloads the video file.

**Typical use cases**
- Turning a product/photo image into a short vertical/horizontal video clip for social media
- Automating creative generation pipelines where the input is an image URL + prompt
- Batch processing by replacing the manual trigger and/or “Set Properties” with a source node (Sheets/DB/Webhook)

### Logical blocks
**1.1 Manual input & parameter preparation**
- Manual trigger and setting `prompt` + `image_url`.

**1.2 Task submission (create video generation job)**
- HTTP request to KIE.AI `createTask` for the Seedance model.

**1.3 Polling loop (wait → check status → route)**
- Wait 5 seconds → query status (`recordInfo`) → Switch routes based on state → continue waiting or proceed.

**1.4 Result parsing & download**
- Parse `resultJson` to extract `resultUrls` → download the first video URL.

---

## 2. Block-by-Block Analysis

### 2.1 Manual input & parameter preparation

**Overview:** Starts the workflow manually and defines the two required inputs: a text prompt and a publicly accessible image URL.

**Nodes involved:**
- `When clicking ‘Execute workflow’`
- `Set Properties`

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – entry point for manual runs.
- **Key configuration:** None (default).
- **Inputs/outputs:**  
  - Input: none  
  - Output → `Set Properties`
- **Failure/edge cases:** None (only depends on user executing).

#### Node: Set Properties
- **Type / role:** Set (`n8n-nodes-base.set`) – prepares payload fields used downstream.
- **Configuration choices (interpreted):**
  - Creates two string fields:
    - `prompt` = `YOUR_VIDEO_PROMPT_DESCRIPTION`
    - `image_url` = `YOUR_IMAGE_URL`
- **Key expressions/variables:** Static placeholder strings; downstream nodes reference:
  - `{{$json.prompt}}`
  - `{{$json.image_url}}`
- **Inputs/outputs:**  
  - Input ← Manual Trigger  
  - Output: (not directly connected in this JSON; see **Edge case** below)
- **Failure/edge cases:**
  - If left as placeholders, the API call will generate poor results or fail.
  - `image_url` must be **publicly reachable** (often HTTPS) or KIE.AI will fail to fetch it.
- **Important wiring note:** In the provided workflow connections, `Set Properties` is **not connected** to `Submit Video Generation Request`. As-is, the workflow won’t start the API call unless you add that connection (see Section 4).

---

### 2.2 Task submission (create video generation job)

**Overview:** Sends a POST request to KIE.AI to create a video generation task using Seedance 1.5 Pro with prompt + image URL and fixed video settings.

**Nodes involved:**
- `Submit Video Generation Request`

#### Node: Submit Video Generation Request
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – creates a KIE.AI job.
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Authentication:**
  - Uses **HTTP Bearer Auth** credential (generic credential type) named `KIA.AI`.
- **Body (JSON):** Sends:
  - `model`: `bytedance/seedance-1.5-pro`
  - `input.prompt`: `{{$json.prompt}}`
  - `input.input_urls[0]`: `{{$json.image_url}}`
  - `input.aspect_ratio`: `"9:16"`
  - `input.resolution`: `"720p"`
  - `input.duration`: `"8"`
- **Options/reliability:**
  - `retryOnFail: true`
- **Outputs / connections:**
  - Output → `Wait for Video Generation`
- **Failure/edge cases:**
  - 401/403 if Bearer token is missing/invalid.
  - 4xx if payload shape is rejected (e.g., wrong model name, invalid duration/resolution).
  - If `image_url` is not accessible, the task may be created but later end in `fail`.
  - API rate limiting or temporary 5xx responses (retries help, but not guaranteed).

---

### 2.3 Polling loop (wait → check status → route)

**Overview:** Implements a polling mechanism: waits 5 seconds, checks job state, and routes based on status. On “queuing/waiting/generating” it loops; on “success” it proceeds; on “fail” it falls through a configured path (see routing notes).

**Nodes involved:**
- `Wait for Video Generation`
- `Check Video Generation Status`
- `Switch Video Generation Status`

#### Node: Wait for Video Generation
- **Type / role:** Wait (`n8n-nodes-base.wait`) – delays execution.
- **Configuration:** Waits **5 seconds**.
- **Outputs / connections:**
  - Output → `Check Video Generation Status`
- **Failure/edge cases:**
  - Large numbers of concurrent runs can create queue pressure in n8n.
  - Very long generation times can keep a run active for a long time (operational consideration).

#### Node: Check Video Generation Status
- **Type / role:** HTTP Request – queries KIE.AI for task progress/result.
- **Endpoint:** `GET https://api.kie.ai/api/v1/jobs/recordInfo` (method not explicitly set; defaults to GET in n8n HTTP Request)
- **Query string:**
  - `taskId = {{$json.data.taskId}}`
- **Authentication:** Same HTTP Bearer Auth credential `KIA.AI`.
- **Outputs / connections:**
  - Output → `Switch Video Generation Status`
- **Failure/edge cases:**
  - If the incoming item does not contain `data.taskId` (e.g., missing connection from submission node), expression resolves to empty → API returns error.
  - 401/403 for auth, 404 if taskId unknown/expired.

#### Node: Switch Video Generation Status
- **Type / role:** Switch (`n8n-nodes-base.switch`) – routes by `data.state`.
- **Routing rules (string equals):**
  - `fail` if `{{$json.data.state}} === "fail"`
  - `success` if `... === "success"`
  - `generating` if `... === "generating"`
  - `queuing` if `... === "queuing"`
  - `waiting` if `... === "waiting"`
- **Error handling:** `onError: continueRegularOutput` (won’t stop workflow on internal switch evaluation errors)
- **Outputs / connections (as wired):**
  - Output 0 (fail) → `Submit Video Generation Request`
  - Output 1 (success) → `Extract Video URL`
  - Output 2 (generating) → `Wait for Video Generation`
  - Output 3 (queuing) → `Wait for Video Generation`
  - Output 4 (waiting) → `Wait for Video Generation`
- **Important behavior note:** Routing `fail → Submit Video Generation Request` causes **automatic resubmission** on failure (potential infinite loop and repeated charges/usage). In many cases you’d instead stop, notify, or limit retries.
- **Failure/edge cases:**
  - If `data.state` is missing/null, none of the rules match; depending on n8n switch behavior, items may be dropped (no output), ending the run silently.
  - If KIE.AI introduces new states (e.g., `cancelled`), they won’t match and may stall/stop.
  - Infinite polling if the task never reaches success/fail (no max-attempt guard is present).

---

### 2.4 Result parsing & download

**Overview:** Once the task is successful, extracts the resulting video URL(s) from a stringified JSON field and downloads the first URL.

**Nodes involved:**
- `Extract Video URL`
- `Download Video File`

#### Node: Extract Video URL
- **Type / role:** Code (`n8n-nodes-base.code`) – parses result JSON safely.
- **Logic summary:**
  - Reads all input items.
  - Handles the case where the HTTP response might be an array at top-level.
  - Gets `first.data.resultJson` (string).
  - `JSON.parse()` it; extracts `parsed.resultUrls` (array).
  - Outputs `{ resultUrls }`.
- **Key variables/fields:**
  - Input expected: `data.resultJson` containing JSON string like `{"resultUrls":[...]}`
  - Output: `resultUrls: [] | string[]`
- **Outputs / connections:**
  - Output → `Download Video File`
- **Failure/edge cases:**
  - If `resultJson` is empty or not valid JSON, it returns `resultUrls: []`.
  - If KIE.AI changes schema (`resultUrls` renamed), download will fail downstream.
  - Multiple items: it emits one output item per input item.

#### Node: Download Video File
- **Type / role:** HTTP Request – downloads the video binary from URL.
- **URL:** `{{$json.resultUrls[0]}}`
- **Outputs / connections:** None (final node).
- **Failure/edge cases:**
  - If `resultUrls` is empty, URL becomes `undefined` → request fails.
  - If the URL is time-limited or requires headers/auth, download may 403/404.
  - By default, HTTP Request may return JSON/text; to store as a binary file you typically set **“Response Format: File”** (not shown here). As configured, it may not produce a usable binary depending on n8n defaults/version.

---

### 2.5 Sticky notes / embedded documentation

**Overview:** Two sticky notes provide author/contact info and a setup/integration checklist; one empty sticky note is used likely as a visual grouping container.

**Nodes involved:**
- `Sticky Note7`
- `Sticky Note2`
- `Sticky Note4` (empty)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note7 | Sticky Note | Author/contact info |  |  | ## Muhammad Farooq Iqbal - Automation Expert & n8n Creator… **LinkedIn**: https://linkedin.com/in/muhammadfarooqiqbal **Portfolio**: https://mfarooqone.github.io/n8n/ **UpWork**: https://www.upwork.com/freelancers/~011aeba159896e2eba |
| Sticky Note2 | Sticky Note | Setup & integration guidance |  |  | ## 🔧 Setup & Integration Guide … Get API key from https://kie.ai/ … Polling checks every 5 seconds |
| Sticky Note4 | Sticky Note | Visual grouping (empty) |  |  |  |
| When clicking ‘Execute workflow’ | Manual Trigger | Entry point (manual run) |  | Set Properties |  |
| Set Properties | Set | Defines `prompt` and `image_url` | When clicking ‘Execute workflow’ | (none in JSON) |  |
| Submit Video Generation Request | HTTP Request | Create KIE.AI generation task | Switch Video Generation Status (fail path) *(and should also receive from Set Properties)* | Wait for Video Generation |  |
| Wait for Video Generation | Wait | Delay between polling attempts | Submit Video Generation Request; Switch Video Generation Status (generating/queuing/waiting) | Check Video Generation Status |  |
| Check Video Generation Status | HTTP Request | Query task state/result by taskId | Wait for Video Generation | Switch Video Generation Status |  |
| Switch Video Generation Status | Switch | Route by `data.state` | Check Video Generation Status | Submit Video Generation Request (fail); Extract Video URL (success); Wait for Video Generation (others) |  |
| Extract Video URL | Code | Parse `resultJson` and output `resultUrls` | Switch Video Generation Status (success) | Download Video File |  |
| Download Video File | HTTP Request | Download the generated video | Extract Video URL |  |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n and name it:  
   `Generate video from an image with ByteDance Seedance 1.5 Pro via KIE.AI`

2) **Add Manual Trigger**
- Node: **Manual Trigger**
- Name: `When clicking ‘Execute workflow’`

3) **Add Set node for inputs**
- Node: **Set**
- Name: `Set Properties`
- Add fields:
  - `prompt` (String) = `YOUR_VIDEO_PROMPT_DESCRIPTION`
  - `image_url` (String) = `YOUR_IMAGE_URL`
- Connect: `When clicking ‘Execute workflow’` → `Set Properties`

4) **Create KIE.AI Bearer credential**
- Go to **Credentials** → create **HTTP Bearer Auth**
- Token: your KIE.AI API key from https://kie.ai/
- Name it e.g. `KIA.AI`

5) **Add HTTP Request to create the task**
- Node: **HTTP Request**
- Name: `Submit Video Generation Request`
- Method: **POST**
- URL: `https://api.kie.ai/api/v1/jobs/createTask`
- Authentication: **Generic Credential Type** → **HTTP Bearer Auth** → select `KIA.AI`
- Body Content Type: **JSON**
- Body (use expressions for prompt and image):
  - `model`: `bytedance/seedance-1.5-pro`
  - `input.prompt`: `{{$json.prompt}}`
  - `input.input_urls`: `[{{$json.image_url}}]`
  - `input.aspect_ratio`: `9:16` (adjust as needed)
  - `input.resolution`: `720p` (adjust as needed)
  - `input.duration`: `8` (seconds; adjust as needed)
- Connect: `Set Properties` → `Submit Video Generation Request`

6) **Add Wait node (poll interval)**
- Node: **Wait**
- Name: `Wait for Video Generation`
- Mode: time interval
- Unit: **seconds**, Amount: **5**
- Connect: `Submit Video Generation Request` → `Wait for Video Generation`

7) **Add HTTP Request to check status**
- Node: **HTTP Request**
- Name: `Check Video Generation Status`
- Method: **GET**
- URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
- Authentication: **HTTP Bearer Auth** → `KIA.AI`
- Query parameter:
  - `taskId` = `{{$json.data.taskId}}`
- Connect: `Wait for Video Generation` → `Check Video Generation Status`

8) **Add Switch node to route by state**
- Node: **Switch**
- Name: `Switch Video Generation Status`
- Value to evaluate: `{{$json.data.state}}`
- Add 5 rules (string equals):
  - `fail`
  - `success`
  - `generating`
  - `queuing`
  - `waiting`
- Connect: `Check Video Generation Status` → `Switch Video Generation Status`

9) **Loop for in-progress states**
- Connect Switch outputs:
  - `generating` → `Wait for Video Generation`
  - `queuing` → `Wait for Video Generation`
  - `waiting` → `Wait for Video Generation`

10) **Handle success → parse result URLs**
- Add node: **Code**
- Name: `Extract Video URL`
- Paste logic (same behavior as provided): parse `data.resultJson` and output `resultUrls`.
- Connect: Switch `success` → `Extract Video URL`

11) **Download the generated video**
- Add node: **HTTP Request**
- Name: `Download Video File`
- URL: `{{$json.resultUrls[0]}}`
- (Recommended) Set **Response Format** to **File** so the output is binary video data.
- Connect: `Extract Video URL` → `Download Video File`

12) **Decide how to handle failures (important)**
- The provided workflow connects `fail → Submit Video Generation Request` (resubmits).
- Recommended safer options:
  - Add an **Error/Stop** path (e.g., “Respond with error”, “Send Slack/Email”, or “Terminate”)
  - Add a **max retry counter** (e.g., increment a field each loop and stop after N attempts)
- If you want to replicate exactly: connect Switch `fail` → `Submit Video Generation Request` (be aware of potential infinite retries).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| KIE.AI API key required; configure as HTTP Bearer Auth credential | https://kie.ai/ |
| Author / support contact: Muhammad Farooq Iqbal – Email: mfarooqiqbal143@gmail.com, Phone: +923036991118 | LinkedIn: https://linkedin.com/in/muhammadfarooqiqbal |
| Portfolio | https://mfarooqone.github.io/n8n/ |
| Upwork profile | https://www.upwork.com/freelancers/~011aeba159896e2eba |
| Image must be publicly accessible (HTTPS recommended); formats PNG/JPG/JPEG | Setup note content |
| Polling interval is 5 seconds; typical generation time 1–5 minutes | Setup note content |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.