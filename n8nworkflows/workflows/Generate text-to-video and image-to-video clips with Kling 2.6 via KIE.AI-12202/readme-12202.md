Generate text-to-video and image-to-video clips with Kling 2.6 via KIE.AI

https://n8nworkflows.xyz/workflows/generate-text-to-video-and-image-to-video-clips-with-kling-2-6-via-kie-ai-12202


# Generate text-to-video and image-to-video clips with Kling 2.6 via KIE.AI

## 1. Workflow Overview

**Title:** Generate text-to-video and image-to-video clips with Kling 2.6 via KIE.AI  
**Purpose:** This workflow uses the **KIE.AI** API to generate videos with the **Kling 2.6** model in two modes:
- **Text-to-Video:** generate a clip from a text prompt, with optional sound, duration, and aspect ratio.
- **Image-to-Video:** animate an existing image URL into a video clip, with optional sound and duration.

It follows a common asynchronous-job pattern:
1) submit a generation task → 2) poll the task status every 5 seconds → 3) when successful, parse the returned `resultJson` and download the resulting video.

### 1.1 Entry / Parameter Setup
User starts the workflow manually and sets generation parameters (text-to-video by default).

### 1.2 Text-to-Video Job Submission + Polling
Submits a Kling 2.6 text-to-video task and repeatedly checks status until it becomes `success` or `fail`.

### 1.3 Text-to-Video Result Extraction + Download
Parses `data.resultJson` (stringified JSON) to extract `resultUrls`, then downloads the first URL.

### 1.4 Image-to-Video Job Submission + Polling
A parallel branch (not connected to the manual trigger in this JSON) that submits an image-to-video task and polls similarly.

### 1.5 Image-to-Video Result Extraction + Download
Parses `data.resultJson` to extract `resultUrls`, then downloads the first URL.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / On-canvas notes
**Overview:** Sticky notes describing how the workflow works, setup steps, and creator contact information.  
**Nodes involved:** `Sticky Note1`, `Text-to-Video Section`, `Image-to-Video Section`, `Sticky Note7`

#### Sticky Note1
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Explains Kling 2.6 generation via KIE.AI, polling cadence (5 seconds), and setup steps (API key, credential name, which Set node to edit).
- **Connections:** None (notes don’t connect).
- **Edge cases:** None.

#### Text-to-Video Section
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Describes the text-to-video part and polling/download behavior.
- **Connections:** None.

#### Image-to-Video Section
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Describes image-to-video part and the need for an image URL + prompt.
- **Connections:** None.

#### Sticky Note7
- **Type / role:** Sticky Note (credits/contact)
- **Configuration:** Creator bio + contact links.
- **Connections:** None.

---

### Block B — Manual entry + Text-to-Video parameterization
**Overview:** Starts execution manually and defines the input parameters for text-to-video generation.  
**Nodes involved:** `When clicking ‘Execute workflow’`, `Set Text to Video Parameters`

#### When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (entry point)
- **Configuration:** No parameters; runs when user clicks **Execute workflow** in n8n.
- **Output:** Sends a single empty item into the flow.
- **Connections:** Outputs to `Set Text to Video Parameters`.
- **Edge cases:** None.

#### Set Text to Video Parameters
- **Type / role:** Set node (creates the payload fields used later)
- **Configuration choices (interpreted):**
  - Sets:
    - `prompt` (string): placeholder `YOUR_VIDEO_PROMPT_DESCRIPTION`
    - `sound` (boolean): `false`
    - `duration` (string): `10` (note: used as a number later; see edge cases)
    - `aspect_ratio` (string): `9:16`
- **Key expressions/variables:** Static values; downstream nodes reference `$json.prompt`, `$json.sound`, `$json.duration`, `$json.aspect_ratio`.
- **Connections:** To `Submit Video Generation Request`.
- **Edge cases / failures:**
  - **Type mismatch risk:** `duration` is stored as a **string**, but the HTTP body injects it unquoted as JSON (expects numeric). If you set a non-numeric string, the request body becomes invalid JSON.
  - `aspect_ratio` is also injected **unquoted** in the HTTP body, but is set to `9:16` (not valid JSON unless quoted). This is a likely breakage in the current template unless the API expects a non-standard format or n8n coerces it unexpectedly. See Block C for details.

---

### Block C — Text-to-Video task submission
**Overview:** Submits a Kling 2.6 text-to-video generation task to KIE.AI.  
**Nodes involved:** `Submit Video Generation Request`

#### Submit Video Generation Request
- **Type / role:** HTTP Request (createTask endpoint)
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.kie.ai/api/v1/jobs/createTask`
  - **Auth:** HTTP Bearer Auth credential (named **“KIA.AI”** in this workflow; note sticky says **“KIE.AI”**)
  - **Body:** JSON (constructed via expression) with:
    - `model`: `"kling-2.6/text-to-video"`
    - `input.prompt`: from `{{$json.prompt}}`
    - `input.sound`: from `{{$json.sound}}` (boolean)
    - `input.duration`: from `{{$json.duration}}` (injected raw)
    - `input.aspect_ratio`: from `{{$json.aspect_ratio}}` (injected raw)
  - **Retry on fail:** enabled
- **Input / output:**
  - **Input:** item from `Set Text to Video Parameters`
  - **Output:** expected to include `data.taskId` (used later)
  - Connected to `Wait for Video Generation`
- **Edge cases / failures:**
  - **Invalid JSON body risk** due to unquoted `duration`/`aspect_ratio`. Recommended safe body formatting:
    - `duration`: ensure numeric (Set node “number” type, or wrap with `{{ Number($json.duration) }}`)
    - `aspect_ratio`: quote it in JSON: `"aspect_ratio": "{{ $json.aspect_ratio }}"`
  - **Auth failures:** invalid/expired API key → 401/403.
  - **API validation:** missing/invalid prompt, unsupported aspect ratio/duration, etc.
  - **Timeout/network:** standard HTTP node errors.

---

### Block D — Text-to-Video polling loop (status check + switch)
**Overview:** Waits 5 seconds, checks task status, and routes based on `data.state`. Loops while queued/generating/waiting.  
**Nodes involved:** `Wait for Video Generation`, `Check Video Generation Status`, `Switch Video Generation Status`

#### Wait for Video Generation
- **Type / role:** Wait node (delay between polls)
- **Configuration:** Wait 5 seconds.
- **Connections:** From `Submit Video Generation Request` and also looped back from the switch states; outputs to `Check Video Generation Status`.
- **Edge cases:**
  - Long jobs can cause many loop iterations; consider n8n execution limits/timeouts on your instance.

#### Check Video Generation Status
- **Type / role:** HTTP Request (recordInfo endpoint)
- **Configuration:**
  - **Method:** GET (default)
  - **URL:** `https://api.kie.ai/api/v1/jobs/recordInfo`
  - **Query param:** `taskId={{ $json.data.taskId }}`
  - **Auth:** HTTP Bearer Auth credential “KIA.AI”
  - **Retry on fail:** enabled
- **Input / output:**
  - **Input:** item that must contain `data.taskId` (from createTask response)
  - **Output:** expected to include `data.state` and, on success, `data.resultJson`
  - Connected to `Switch Video Generation Status`
- **Edge cases / failures:**
  - If the previous node output does **not** have `data.taskId` (API schema changes or error responses), the expression resolves to empty → API likely returns error.
  - API may rate-limit frequent polling.

#### Switch Video Generation Status
- **Type / role:** Switch (route by generation state)
- **Configuration:**
  - Routes based on `={{ $json.data.state }}`
  - Outputs (renamed):
    - `fail` → goes to `Submit Video Generation Request` (this is unusual; see edge case)
    - `success` → goes to `Extract Video URL`
    - `generating` → goes to `Wait for Video Generation`
    - `queuing` → goes to `Wait for Video Generation`
    - `waiting` → goes to `Wait for Video Generation`
  - **On error:** continue regular output
  - **Retry on fail:** enabled
- **Input / output:**
  - **Input:** status response from `Check Video Generation Status`
- **Edge cases / failures:**
  - **Fail path likely incorrect:** on `fail`, it re-submits a new generation request (`Submit Video Generation Request`) rather than stopping/logging. This can cause infinite retries and unexpected costs.
  - Unhandled states: if API returns a state not listed, nothing matches; depending on n8n Switch behavior and “continue regular output”, items may go to default output (not wired) and effectively stop silently.

---

### Block E — Text-to-Video result parsing + download
**Overview:** Extracts the final video URL(s) from a stringified JSON field, then downloads the first URL.  
**Nodes involved:** `Extract Video URL`, `Download Video File`

#### Extract Video URL
- **Type / role:** Code node (JSON parsing / normalization)
- **Configuration choices:**
  - Iterates over all input items.
  - Handles cases where HTTP response is an array at the top level.
  - Reads `first.data.resultJson` (string), parses JSON, extracts `parsed.resultUrls` array.
  - Outputs `{ resultUrls }`.
- **Input / output:**
  - **Input:** successful `recordInfo` response (should contain `data.resultJson`)
  - **Output:** `resultUrls` array → consumed by downloader
  - Connected to `Download Video File`
- **Edge cases / failures:**
  - If `data.resultJson` is missing or not valid JSON, output will be `resultUrls: []` (silent failure). Downstream `resultUrls[0]` becomes `undefined`.

#### Download Video File
- **Type / role:** HTTP Request (download binary/video file)
- **Configuration:**
  - **URL:** `={{ $json.resultUrls[0] }}`
  - Method: GET (default)
- **Input / output:**
  - **Input:** item with `resultUrls` array
  - **Output:** downloads the file (in practice you likely want “Download”/binary settings; here it’s a plain HTTP call)
- **Edge cases / failures:**
  - If `resultUrls[0]` is undefined, request fails with invalid URL.
  - Remote hosting may require signed URLs that expire; downloading too late can fail.
  - Consider setting response format to **File** / **Binary** if you intend to store it (Drive/S3) afterward.

---

### Block F — Image-to-Video parameterization + submission
**Overview:** Defines prompt/image parameters and submits an image-to-video generation request. This branch is currently **not connected** to the manual trigger in the provided workflow, so it won’t run unless you connect it.  
**Nodes involved:** `Set Prompt & Image Url`, `Submit Video Generation1`

#### Set Prompt & Image Url
- **Type / role:** Set node (defines image-to-video inputs)
- **Configuration:**
  - `prompt`: placeholder `YOUR_VIDEO_PROMPT_DESCRIPTION`
  - `duration`: string `5` (again numeric-in-string)
  - `image_urls`: placeholder `YOUR_IMAGE_URL` (a single URL string)
  - `sound`: boolean `true`
- **Connections:** To `Submit Video Generation1`
- **Edge cases:**
  - Duration type mismatch risk (string used as number in JSON).

#### Submit Video Generation1
- **Type / role:** HTTP Request (createTask for image-to-video)
- **Configuration:**
  - **POST** `https://api.kie.ai/api/v1/jobs/createTask`
  - **Auth:** HTTP Bearer Auth “KIA.AI”
  - **Body JSON:**
    - `model`: `"kling-2.6/image-to-video"`
    - `input.prompt`: `{{$json.prompt}}`
    - `input.image_urls`: array with one element: `{{$json.image_urls}}`
    - `input.sound`: `{{$json.sound}}`
    - `input.duration`: `{{$json.duration}}` (raw)
  - **Retry on fail:** enabled
- **Output / connections:** To `Wait for Image-to-Video Generation`
- **Edge cases / failures:**
  - Invalid/blocked image URL, unsupported format, inaccessible host (403), large files.
  - If `duration` is not numeric, request JSON can break.

---

### Block G — Image-to-Video polling loop
**Overview:** Waits 5 seconds, checks status, routes based on `data.state`, loops until success or fail.  
**Nodes involved:** `Wait for Image-to-Video Generation`, `Check Video Status`, `Switch`

#### Wait for Image-to-Video Generation
- **Type / role:** Wait node (poll delay)
- **Configuration:** 5 seconds.
- **Connections:** From `Submit Video Generation1` and looped back from `Switch` for queued/generating/waiting; outputs to `Check Video Status`.
- **Edge cases:** Same as text-to-video polling: execution duration and rate limits.

#### Check Video Status
- **Type / role:** HTTP Request (recordInfo)
- **Configuration:**
  - GET `https://api.kie.ai/api/v1/jobs/recordInfo`
  - Query: `taskId={{ $json.data.taskId }}`
  - Auth: Bearer “KIA.AI”
- **Connections:** To `Switch`
- **Edge cases:** Missing `data.taskId` causes invalid query.

#### Switch
- **Type / role:** Switch (route by `data.state`)
- **Configuration:**
  - Conditions on `={{ $json.data.state }}`
  - Outputs:
    - `fail` → `Submit Video Generation1` (same “retry from scratch” issue)
    - `success` → `Video URL`
    - `generating` → `Wait for Image-to-Video Generation`
    - `queuing` → `Wait for Image-to-Video Generation`
    - `waiting` → `Wait for Image-to-Video Generation`
- **Edge cases:**
  - Fail path can loop forever and resubmit new jobs.
  - Unhandled states not connected.

---

### Block H — Image-to-Video result parsing + download
**Overview:** Parses `data.resultJson` to extract `resultUrls` and downloads the first URL.  
**Nodes involved:** `Video URL`, `Download Video1`

#### Video URL
- **Type / role:** Code node (same logic as Extract Video URL)
- **Configuration:** Same parsing strategy and output `{ resultUrls }`.
- **Connections:** To `Download Video1`
- **Edge cases:** Same as text-to-video parsing: empty/invalid `resultJson` → `resultUrls: []`.

#### Download Video1
- **Type / role:** HTTP Request (download video)
- **Configuration:** URL `={{ $json.resultUrls[0] }}`
- **Edge cases:** Undefined URL, expired signed link, binary handling not configured.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set Text to Video Parameters | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Set Text to Video Parameters | Set | Define text-to-video inputs | When clicking ‘Execute workflow’ | Submit Video Generation Request | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Submit Video Generation Request | HTTP Request | Create text-to-video task (KIE.AI createTask) | Set Text to Video Parameters; Switch Video Generation Status (fail) | Wait for Video Generation | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Wait for Video Generation | Wait | Poll delay (5s) | Submit Video Generation Request; Switch Video Generation Status (generating/queuing/waiting) | Check Video Generation Status | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Check Video Generation Status | HTTP Request | Fetch task status (KIE.AI recordInfo) | Wait for Video Generation | Switch Video Generation Status | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Switch Video Generation Status | Switch | Route by `data.state` | Check Video Generation Status | Submit Video Generation Request; Extract Video URL; Wait for Video Generation | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Extract Video URL | Code | Parse `data.resultJson` to `resultUrls` | Switch Video Generation Status (success) | Download Video File | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Download Video File | HTTP Request | Download final video from URL | Extract Video URL | — | **Text-to-Video Workflow**: Generates videos from text prompts using Kling 2.6. Set your parameters, submit the request, and the workflow polls for completion before downloading. |
| Set Prompt & Image Url | Set | Define image-to-video inputs | — | Submit Video Generation1 | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Submit Video Generation1 | HTTP Request | Create image-to-video task (KIE.AI createTask) | Set Prompt & Image Url; Switch (fail) | Wait for Image-to-Video Generation | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Wait for Image-to-Video Generation | Wait | Poll delay (5s) | Submit Video Generation1; Switch (generating/queuing/waiting) | Check Video Status | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Check Video Status | HTTP Request | Fetch task status (KIE.AI recordInfo) | Wait for Image-to-Video Generation | Switch | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Switch | Switch | Route by `data.state` | Check Video Status | Submit Video Generation1; Video URL; Wait for Image-to-Video Generation | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Video URL | Code | Parse `data.resultJson` to `resultUrls` | Switch (success) | Download Video1 | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Download Video1 | HTTP Request | Download final video from URL | Video URL | — | **Image-to-Video Workflow**: Animates existing images into videos using Kling 2.6. Provide an image URL and prompt, then the workflow handles generation and download. |
| Sticky Note1 | Sticky Note | Setup/how-it-works notes | — | — | ## How it works… (setup steps + key + which Set nodes to edit) |
| Text-to-Video Section | Sticky Note | Section label/notes | — | — | **Text-to-Video Workflow** (section description) |
| Image-to-Video Section | Sticky Note | Section label/notes | — | — | **Image-to-Video Workflow** (section description) |
| Sticky Note7 | Sticky Note | Credits/contact | — | — | Muhammad Farooq Iqbal contact + links |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials (required)**
   1. In n8n: **Credentials → New**
   2. Select **HTTP Bearer Auth**
   3. Name it (the sticky note suggests **“KIE.AI”**; the nodes currently reference **“KIA.AI”**—pick one and use it consistently)
   4. Paste your KIE.AI API key (from https://kie.ai/)

2) **Create the Text-to-Video path**
   1. Add **Manual Trigger** node named: `When clicking ‘Execute workflow’`.
   2. Add a **Set** node named `Set Text to Video Parameters` and configure fields:
      - `prompt` (string): your prompt text
      - `sound` (boolean): true/false
      - `duration` (number recommended): e.g. 10
      - `aspect_ratio` (string): e.g. `9:16`
   3. Add **HTTP Request** node named `Submit Video Generation Request`:
      - Method: **POST**
      - URL: `https://api.kie.ai/api/v1/jobs/createTask`
      - Authentication: **HTTP Bearer Auth** (select your credential)
      - Body: JSON containing:
        - model: `kling-2.6/text-to-video`
        - input: prompt, sound, duration, aspect_ratio
      - Enable **Retry on fail**
      - (Important) Ensure JSON is valid: quote `aspect_ratio` and send `duration` as a number.
   4. Add **Wait** node named `Wait for Video Generation`: 5 seconds.
   5. Add **HTTP Request** node named `Check Video Generation Status`:
      - GET `https://api.kie.ai/api/v1/jobs/recordInfo`
      - Query parameter: `taskId` = expression from createTask output (typically `{{$json.data.taskId}}`)
      - Auth: same Bearer credential
   6. Add a **Switch** node named `Switch Video Generation Status`:
      - Switch on `{{$json.data.state}}`
      - Create outputs for: `fail`, `success`, `generating`, `queuing`, `waiting`
   7. Add a **Code** node named `Extract Video URL` that:
      - Parses `{{$json.data.resultJson}}` (string) as JSON
      - Extracts `resultUrls` array
   8. Add **HTTP Request** node named `Download Video File`:
      - URL: `{{$json.resultUrls[0]}}`

   9. **Connect nodes (Text-to-Video)**
      - Manual Trigger → Set Text to Video Parameters → Submit Video Generation Request → Wait for Video Generation → Check Video Generation Status → Switch Video Generation Status
      - Switch `success` → Extract Video URL → Download Video File
      - Switch `generating`/`queuing`/`waiting` → Wait for Video Generation (loop)
      - Switch `fail` → (recommended) route to an error-handling node (Stop & Error / notification). If you replicate exactly, connect to `Submit Video Generation Request` (but this can cause repeated job creation).

3) **Create the Image-to-Video path (optional second branch)**
   1. Add a **Set** node named `Set Prompt & Image Url`:
      - `prompt` (string)
      - `image_urls` (string) = a publicly reachable image URL
      - `duration` (number recommended)
      - `sound` (boolean)
   2. Add **HTTP Request** node `Submit Video Generation1`:
      - POST `https://api.kie.ai/api/v1/jobs/createTask`
      - model: `kling-2.6/image-to-video`
      - input.image_urls: an array containing the URL (e.g. `["{{$json.image_urls}}"]`)
      - Auth: Bearer credential
   3. Add **Wait** node `Wait for Image-to-Video Generation` (5 seconds)
   4. Add **HTTP Request** node `Check Video Status` (GET recordInfo with `taskId={{$json.data.taskId}}`)
   5. Add **Switch** node `Switch` (route by `data.state` same outputs)
   6. Add **Code** node `Video URL` (same parsing as `Extract Video URL`)
   7. Add **HTTP Request** node `Download Video1` using `{{$json.resultUrls[0]}}`
   8. Connect:
      - Set Prompt & Image Url → Submit Video Generation1 → Wait → Check Video Status → Switch
      - Switch `success` → Video URL → Download Video1
      - Switch `generating/queuing/waiting` → Wait (loop)
      - Switch `fail` → (recommended) error handling (template loops back to Submit Video Generation1)

4) **Add sticky notes (optional)**
   - Add sticky notes with the provided content for setup instructions and credits.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your KIE.AI API key | https://kie.ai/ |
| Create HTTP Bearer Auth credential (named “KIE.AI” per note; nodes use “KIA.AI”) | n8n Credentials |
| LinkedIn: Connect with me | https://linkedin.com/in/muhammadfarooqiqbal |
| Portfolio: View my work | https://mfarooqone.github.io/n8n/ |
| Upwork Profile | https://www.upwork.com/freelancers/~011aeba159896e2eba |
| Contact email/phone (from note) | mfarooqiqbal143@gmail.com / +923036991118 |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.