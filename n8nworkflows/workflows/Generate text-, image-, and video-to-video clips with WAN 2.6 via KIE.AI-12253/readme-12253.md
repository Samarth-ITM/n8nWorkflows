Generate text-, image-, and video-to-video clips with WAN 2.6 via KIE.AI

https://n8nworkflows.xyz/workflows/generate-text---image---and-video-to-video-clips-with-wan-2-6-via-kie-ai-12253


# Generate text-, image-, and video-to-video clips with WAN 2.6 via KIE.AI

## 1. Workflow Overview

**Title:** Generate text-, image-, and video-to-video clips with WAN 2.6 via KIE.AI  
**Purpose:** This workflow uses the **KIE.AI API** to generate videos with **WAN 2.6** in three modes—**text-to-video**, **image-to-video**, and **video-to-video**—by creating a generation task, polling its status every 5 seconds, then downloading the final video once complete.

**Typical use cases**
- Create short AI video clips (5/10/15s) from text prompts.
- Animate a still image into a short video.
- Transform an existing video using a prompt (style/scene changes).

**Logical blocks (by dependency)**
1.1 **Entry & parameter setup (Text-to-Video path active by default)**  
1.2 **Text-to-Video job submission → polling → download**  
1.3 **Image-to-Video job submission → polling → download** (not connected to the trigger in this JSON)  
1.4 **Video-to-Video job submission → polling → download** (not connected to the trigger in this JSON)  
1.5 **Documentation / Credits (Sticky Notes)**

> Important: Only the **Text-to-Video** branch is connected to the **Manual Trigger**. The Image-to-Video and Video-to-Video branches exist but are **not reachable** unless you connect them to a trigger (or manually execute nodes).

---

## 2. Block-by-Block Analysis

### 1.1 Entry & Parameter Setup (Text-to-Video)

**Overview:** Starts the workflow manually and defines core generation parameters (prompt, duration, resolution) used by the text-to-video request.

**Nodes involved**
- **When clicking ‘Execute workflow’**
- **Set Video Parameters**

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; workflow entry point.
- **Configuration:** No parameters; runs when user clicks *Execute workflow*.
- **Connections:**  
  - Output → **Set Video Parameters**
- **Edge cases / failures:** None (manual start).

#### Node: Set Video Parameters
- **Type / role:** Set node; builds the input payload fields used downstream.
- **Key fields set:**
  - `prompt` (string): `YOUR_VIDEO_PROMPT_DESCRIPTION` (placeholder)
  - `duration` (string): `5` (expected by API as 5/10/15 seconds per sticky note)
  - `resolution` (string): `720p` (expected values: 720p/1080p)
- **Connections:**  
  - Input ← Manual Trigger  
  - Output → **Submit Video Generation Request**
- **Edge cases / failures:**
  - Leaving placeholders unchanged will produce generic/invalid results.
  - If KIE.AI expects numeric duration but receives a string, it may still work; otherwise could fail validation.

---

### 1.2 Text-to-Video: Submit → Poll → Download

**Overview:** Creates a WAN 2.6 text-to-video task, waits/polls until status becomes `success`, extracts the resulting URL(s), then downloads the first video.

**Nodes involved**
- Submit Video Generation Request
- Wait for Video Generation
- Check Video Generation Status
- Switch Video Generation Status
- Extract Video URL
- Download Video File

#### Node: Submit Video Generation Request
- **Type / role:** HTTP Request; submits a generation task to KIE.AI.
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Auth:** HTTP Bearer Auth credential (generic) named **“KIA.AI”** (note: sticky note says “KIE.AI”; credential name in JSON is “KIA.AI”).
- **Body (JSON) structure (expressions):**
  - `model`: `wan/2-6-text-to-video`
  - `input.prompt`: `{{ $json.prompt }}`
  - `input.duration`: `{{ $json.duration }}`
  - `input.resolution`: `{{ $json.resolution }}`
  - `input.multi_shots`: `false`
- **Connections:**  
  - Input ← Set Video Parameters  
  - Output → Wait for Video Generation
- **Version:** node typeVersion **4.2** (HTTP Request)
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid.
  - 429 rate limiting.
  - 4xx if unsupported duration/resolution/model.
  - Unexpected response shape may break downstream expressions referencing `$json.data.taskId`.

#### Node: Wait for Video Generation
- **Type / role:** Wait node; delays before polling status again.
- **Configuration:** Wait **5 seconds**.
- **Connections:**  
  - Input ← Submit Video Generation Request OR Switch (loop states)  
  - Output → Check Video Generation Status
- **Edge cases / failures:** Excessive looping can run long; consider a max-tries guard (not present).

#### Node: Check Video Generation Status
- **Type / role:** HTTP Request; queries task status from KIE.AI.
- **Endpoint:** `GET https://api.kie.ai/api/v1/jobs/recordInfo` (implemented as HTTP Request with query parameters)
- **Query parameter:**  
  - `taskId = {{ $json.data.taskId }}`
- **Auth:** Same Bearer credential **“KIA.AI”**.
- **Connections:**  
  - Input ← Wait for Video Generation  
  - Output → Switch Video Generation Status
- **Version:** typeVersion **4.2**
- **Edge cases / failures:**
  - If the incoming JSON doesn’t contain `data.taskId` (for example, if an upstream node returned a different schema), query will be empty and API may fail.
  - Transient network errors during polling.

#### Node: Switch Video Generation Status
- **Type / role:** Switch; routes by `{{ $json.data.state }}`.
- **onError:** `continueRegularOutput` (workflow will not hard-fail on switch evaluation issues).
- **Rules / outputs (renamed):**
  - `fail` if state == `"fail"`
  - `success` if state == `"success"`
  - `generating` if state == `"generating"`
  - `queuing` if state == `"queuing"`
  - `waiting` if state == `"waiting"`
- **Connections (as configured):**
  - `fail` → **Submit Video Generation Request** (re-submits a new task)
  - `success` → **Extract Video URL**
  - `generating` → **Wait for Video Generation**
  - `queuing` → **Wait for Video Generation**
  - `waiting` → **Wait for Video Generation**
- **Version:** typeVersion **3.2**
- **Edge cases / failures:**
  - If `data.state` is missing/unknown, it won’t match; depending on n8n Switch behavior, items may be dropped or routed to default (no explicit default path is configured here).
  - The `fail` route **recreates the task** rather than surfacing the error; this can cause repeated charges/usage and hide failure reasons.

#### Node: Extract Video URL
- **Type / role:** Code node; parses KIE.AI’s `data.resultJson` and extracts `resultUrls`.
- **Key logic:**
  - Handles the case where the HTTP node returns an array at top level: `const first = Array.isArray(payload) ? payload[0] : payload;`
  - Reads: `first?.data?.resultJson` (string)
  - `JSON.parse(resultJsonStr)` then returns `parsed.resultUrls ?? []`
  - Outputs: `{ resultUrls }`
- **Connections:**  
  - Input ← Switch Video Generation Status (success)  
  - Output → Download Video File
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - If `resultJson` is empty or invalid JSON, returns `resultUrls: []`.
  - Downstream download will fail if `resultUrls[0]` is undefined.

#### Node: Download Video File
- **Type / role:** HTTP Request; downloads the generated video from the first URL.
- **URL:** `{{ $json.resultUrls[0] }}`
- **Connections:**  
  - Input ← Extract Video URL  
  - Output: none (end)
- **Version:** typeVersion **4.3**
- **Edge cases / failures:**
  - If URL is missing/expired, download fails.
  - Node is not configured to “Download as binary data” explicitly in the provided JSON; depending on n8n defaults, it may just fetch metadata/text instead of storing binary. If you intend to save the file, enable binary download and add a storage node (e.g., Write Binary File / S3 / Google Drive).

---

### 1.3 Image-to-Video: Submit → Poll → Download (present but not triggered)

**Overview:** Builds an image-to-video request with an `image_url`, submits it, polls status, then downloads the resulting video.

**Nodes involved**
- Set Prompt & Image Url
- Submit Video Generation a
- Wait for Image-to-Video Generation
- Check Video Status
- Switch Image-to-Video Status
- Video URL1
- Download Video1

#### Node: Set Prompt & Image Url
- **Type / role:** Set node; defines prompt + image input.
- **Fields set:**
  - `prompt`, `duration`, `resolution`
  - `image_url`: `YOUR_IMAGE_URL` (placeholder)
- **Connections:**  
  - Output → Submit Video Generation a
- **Edge cases / failures:** Placeholder image URL will fail; API may require publicly accessible URL.

#### Node: Submit Video Generation a
- **Type / role:** HTTP Request; creates image-to-video task.
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Auth:** Bearer credential **“KIA.AI”**
- **Body:**
  - `model`: `wan/2-6-image-to-video`
  - `input.image_urls`: `[ "{{ $json.image_url }}" ]`
  - plus prompt/duration/resolution/multi_shots
- **Connections:**  
  - Output → Wait for Image-to-Video Generation
- **Version:** 4.2
- **Edge cases / failures:** Same as text-to-video plus inaccessible image URL (403/404), unsupported format.

#### Node: Wait for Image-to-Video Generation
- **Type / role:** Wait 5 seconds then poll.
- **Connections:** Output → Check Video Status

#### Node: Check Video Status
- **Type / role:** HTTP Request; checks recordInfo with `taskId={{ $json.data.taskId }}`.
- **Connections:** Output → Switch Image-to-Video Status

#### Node: Switch Image-to-Video Status
- **Type / role:** Switch on `data.state` identical to the text-to-video switch.
- **Connections (as configured):**
  - `fail` → Submit Video Generation a (re-submit)
  - `success` → Video URL1 (extract URLs)
  - `generating` / `queuing` / `waiting` → Wait for Image-to-Video Generation (poll loop)
- **Edge cases:** Same “unknown state” and “re-submit on fail” concerns.

#### Node: Video URL1
- **Type / role:** Code node; parses `data.resultJson` and extracts `resultUrls` (same code as Extract Video URL).
- **Connections:** Output → Download Video1

#### Node: Download Video1
- **Type / role:** HTTP Request download from `{{ $json.resultUrls[0] }}`.
- **Edge cases:** Same binary/download considerations.

---

### 1.4 Video-to-Video: Submit → Poll → Download (present but not triggered)

**Overview:** Uses a source video URL and prompt to generate a transformed video, polls status, then downloads.

**Nodes involved**
- Set Video URL and Prompt
- Submit Video Generation
- Wait for Video-to-Video Generation
- Check Video Generation
- Switch Video Generation
- Video URL
- Download Video

#### Node: Set Video URL and Prompt
- **Type / role:** Set node; defines prompt, duration, resolution, and source `video_urls`.
- **Fields set:**
  - `video_urls`: sample MP4 URL (public)
- **Connections:** Output → Submit Video Generation
- **Edge cases:** Source URL must be reachable and in supported format; large files may increase processing time.

#### Node: Submit Video Generation
- **Type / role:** HTTP Request; creates video-to-video task.
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Body:**
  - `model`: `wan/2-6-video-to-video`
  - `input.video_urls`: `[ "{{ $json.video_urls }}" ]`
  - plus prompt/duration/resolution/multi_shots
- **Connections:** Output → Wait for Video-to-Video Generation

#### Node: Wait for Video-to-Video Generation
- **Type / role:** Wait 5 seconds then poll.
- **Connections:** Output → Check Video Generation

#### Node: Check Video Generation
- **Type / role:** HTTP Request; checks recordInfo with `taskId={{ $json.data.taskId }}`.
- **Connections:** Output → Switch Video Generation

#### Node: Switch Video Generation
- **Type / role:** Switch on `data.state` identical to others.
- **Connections (as configured):**
  - `fail` → Submit Video Generation (re-submit)
  - `success` → Video URL (extract URLs)
  - `generating` / `queuing` / `waiting` → Wait for Video-to-Video Generation
- **Edge cases:** Same loop/re-submit risks.

#### Node: Video URL
- **Type / role:** Code node; parse `data.resultJson` to `resultUrls`.
- **Connections:** Output → Download Video

#### Node: Download Video
- **Type / role:** HTTP Request download from `{{ $json.resultUrls[0] }}`.

---

### 1.5 Documentation / Credits (Sticky Notes)

**Overview:** Provides usage instructions and author contact details; does not affect execution.

**Nodes involved**
- Sticky Note10
- Text-to-Video Section
- Image-to-Video Section
- Video-to-Video Section
- Sticky Note7

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Entry point (manual run) | — | Set Video Parameters | ## How it works  \nThis workflow generates videos using WAN 2.6 AI through the KIE.AI API. You can create videos three ways: text-to-video (from descriptions), image-to-video (animate images), or video-to-video (transform existing videos). Each workflow submits your request, waits for processing (typically 1-5 minutes), checks status every 5 seconds, and downloads the completed video when ready.  \n## Setup steps  \n1. Get your KIE.AI API key from https://kie.ai/  \n2. In n8n, create an HTTP Bearer Auth credential named "KIE.AI" and paste your API key  \n3. Choose your workflow type and update the corresponding 'Set' node:  \n- Text-to-Video: Update 'Set Video Parameters' (prompt, duration: 5/10/15s, resolution: 720p/1080p)  \n- Image-to-Video: Update 'Set Prompt & Image Url' (prompt, image_url, duration, resolution)  \n- Video-to-Video: Update 'Set Video URL and Prompt' (prompt, video_urls, duration, resolution)  \n4. Click "Execute Workflow" to test. The workflow automatically polls for completion and downloads your video. |
| Set Video Parameters | Set | Define prompt/duration/resolution for text-to-video | When clicking ‘Execute workflow’ | Submit Video Generation Request | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Submit Video Generation Request | HTTP Request | Create text-to-video task | Set Video Parameters; Switch Video Generation Status (fail) | Wait for Video Generation | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Wait for Video Generation | Wait | Delay between status polls | Submit Video Generation Request; Switch Video Generation Status (generating/queuing/waiting) | Check Video Generation Status | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Check Video Generation Status | HTTP Request | Poll task status (text-to-video) | Wait for Video Generation | Switch Video Generation Status | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Switch Video Generation Status | Switch | Route by `data.state` for text-to-video | Check Video Generation Status | Submit Video Generation Request; Extract Video URL; Wait for Video Generation | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Extract Video URL | Code | Parse `resultJson` → `resultUrls[]` (text-to-video) | Switch Video Generation Status (success) | Download Video File | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Download Video File | HTTP Request | Download generated video (text-to-video) | Extract Video URL | — | **Text-to-Video Workflow**  \nGenerates videos from text prompts using WAN 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Set Prompt & Image Url | Set | Define prompt + `image_url` for image-to-video | — | Submit Video Generation a | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Submit Video Generation a | HTTP Request | Create image-to-video task | Set Prompt & Image Url; Switch Image-to-Video Status (fail) | Wait for Image-to-Video Generation | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Wait for Image-to-Video Generation | Wait | Delay between status polls (image-to-video) | Submit Video Generation a; Switch Image-to-Video Status (generating/queuing/waiting) | Check Video Status | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Check Video Status | HTTP Request | Poll task status (image-to-video) | Wait for Image-to-Video Generation | Switch Image-to-Video Status | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Switch Image-to-Video Status | Switch | Route by `data.state` (image-to-video) | Check Video Status | Submit Video Generation a; Video URL1; Wait for Image-to-Video Generation | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Video URL1 | Code | Parse `resultJson` → `resultUrls[]` (image-to-video) | Switch Image-to-Video Status (success) | Download Video1 | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Download Video1 | HTTP Request | Download generated video (image-to-video) | Video URL1 | — | **Image-to-Video Workflow**  \nAnimates existing images into videos using WAN 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Set Video URL and Prompt | Set | Define prompt + source video URL for video-to-video | — | Submit Video Generation | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Submit Video Generation | HTTP Request | Create video-to-video task | Set Video URL and Prompt; Switch Video Generation (fail) | Wait for Video-to-Video Generation | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Wait for Video-to-Video Generation | Wait | Delay between status polls (video-to-video) | Submit Video Generation; Switch Video Generation (generating/queuing/waiting) | Check Video Generation | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Check Video Generation | HTTP Request | Poll task status (video-to-video) | Wait for Video-to-Video Generation | Switch Video Generation | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Switch Video Generation | Switch | Route by `data.state` (video-to-video) | Check Video Generation | Submit Video Generation; Video URL; Wait for Video-to-Video Generation | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Video URL | Code | Parse `resultJson` → `resultUrls[]` (video-to-video) | Switch Video Generation (success) | Download Video | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Download Video | HTTP Request | Download generated video (video-to-video) | Video URL | — | **Video-to-Video Workflow**  \nTransforms existing videos using WAN 2.6. Provide a video URL and prompt, then the workflow handles generation and download. |
| Sticky Note10 | Sticky Note | Embedded usage/setup notes | — | — |  |
| Text-to-Video Section | Sticky Note | Visual section header | — | — |  |
| Image-to-Video Section | Sticky Note | Visual section header | — | — |  |
| Video-to-Video Section | Sticky Note | Visual section header | — | — |  |
| Sticky Note7 | Sticky Note | Author info/credits/contact | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials (required)**
1. In n8n, go to **Credentials → New**.
2. Choose **HTTP Bearer Auth**.
3. Name it (recommended) **KIE.AI** (the provided workflow uses a credential named “KIA.AI”; either rename in nodes or keep consistent).
4. Paste your **KIE.AI API key** (from https://kie.ai/).
5. Save.

2) **Create the Manual Trigger**
1. Add node: **Manual Trigger**.
2. Name: **When clicking ‘Execute workflow’**.

3) **Build the Text-to-Video path (the only path connected by default)**
1. Add node: **Set**  
   - Name: **Set Video Parameters**  
   - Add fields:
     - `prompt` (String) = your text prompt
     - `duration` (String) = `5` (or `10` / `15` if supported)
     - `resolution` (String) = `720p` (or `1080p`)
2. Connect: **Manual Trigger → Set Video Parameters**
3. Add node: **HTTP Request**  
   - Name: **Submit Video Generation Request**  
   - Method: **POST**  
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`  
   - Authentication: **Bearer Auth** (select your credential)  
   - Body: **JSON** and set to:
     - model: `wan/2-6-text-to-video`
     - input: prompt/duration/resolution from the Set node
     - multi_shots: false
4. Connect: **Set Video Parameters → Submit Video Generation Request**
5. Add node: **Wait**  
   - Name: **Wait for Video Generation**
   - Wait: **5 seconds**
6. Connect: **Submit Video Generation Request → Wait for Video Generation**
7. Add node: **HTTP Request**
   - Name: **Check Video Generation Status**
   - Method: GET (or keep default with **Send Query Parameters**)
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
   - Authentication: Bearer (same credential)
   - Query param: `taskId` = expression `{{$json.data.taskId}}`
8. Connect: **Wait for Video Generation → Check Video Generation Status**
9. Add node: **Switch**
   - Name: **Switch Video Generation Status**
   - Value to evaluate: expression `{{$json.data.state}}`
   - Add 5 outputs (rename outputs):
     - fail = `fail`
     - success = `success`
     - generating = `generating`
     - queuing = `queuing`
     - waiting = `waiting`
10. Connect: **Check Video Generation Status → Switch Video Generation Status**
11. Create the polling loop:
   - Connect outputs `generating`, `queuing`, `waiting` → **Wait for Video Generation**
12. Handle success:
   1. Add node: **Code**
      - Name: **Extract Video URL**
      - Code: parse `data.resultJson` (string) and return `resultUrls` array.
   2. Connect Switch `success` → **Extract Video URL**
   3. Add node: **HTTP Request**
      - Name: **Download Video File**
      - URL: expression `{{$json.resultUrls[0]}}`
      - (Optional but recommended) enable “Download as binary” and set a binary property name (e.g., `data`), then add a storage node.
   4. Connect: **Extract Video URL → Download Video File**
13. (Optional) Handle fail:
   - Current JSON re-submits on `fail` by connecting `fail` → **Submit Video Generation Request**.
   - Safer alternative: connect `fail` to a **Stop and Error** or **Slack/Email alert** node.

5) **Build the Image-to-Video path (and connect it if you want it runnable)**
1. Add **Set** node: **Set Prompt & Image Url** with `prompt`, `duration`, `resolution`, `image_url`.
2. Add **HTTP Request** node: **Submit Video Generation a**
   - POST createTask, model `wan/2-6-image-to-video`
   - input.image_urls = array with `{{$json.image_url}}`
3. Add **Wait**: **Wait for Image-to-Video Generation** (5 seconds)
4. Add **HTTP Request**: **Check Video Status** (recordInfo with `taskId={{$json.data.taskId}}`)
5. Add **Switch**: **Switch Image-to-Video Status** (route by state; loop on generating/queuing/waiting)
6. Add **Code**: **Video URL1** (extract `resultUrls`)
7. Add **HTTP Request**: **Download Video1** (`{{$json.resultUrls[0]}}`)
8. To make it executable: connect **Manual Trigger → Set Prompt & Image Url** (or add a separate trigger / switch between modes).

6) **Build the Video-to-Video path (and connect it if you want it runnable)**
1. Add **Set** node: **Set Video URL and Prompt** with `prompt`, `duration`, `resolution`, `video_urls`.
2. Add **HTTP Request**: **Submit Video Generation** with model `wan/2-6-video-to-video` and `video_urls` array.
3. Add **Wait**: **Wait for Video-to-Video Generation** (5 seconds)
4. Add **HTTP Request**: **Check Video Generation** (recordInfo with taskId)
5. Add **Switch**: **Switch Video Generation** (route by state; loop on generating/queuing/waiting)
6. Add **Code**: **Video URL** (extract `resultUrls`)
7. Add **HTTP Request**: **Download Video**
8. To make it executable: connect **Manual Trigger → Set Video URL and Prompt** (or add a mode selector).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your KIE.AI API key and use HTTP Bearer Auth in n8n | https://kie.ai/ |
| Author / support contact: Muhammad Farooq Iqbal – Automation Expert & n8n Creator. Email: mfarooqiqbal143@gmail.com. Phone: +923036991118. LinkedIn: https://linkedin.com/in/muhammadfarooqiqbal. Portfolio: https://mfarooqone.github.io/n8n/. UpWork: https://www.upwork.com/freelancers/~011aeba159896e2eba | From workflow sticky note |
| Processing pattern: submit → wait/poll every 5s → download when state becomes success (often 1–5 minutes) | From “How it works” sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.