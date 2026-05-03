Generate product videos from images with BrowserAct and Google Drive

https://n8nworkflows.xyz/workflows/generate-product-videos-from-images-with-browseract-and-google-drive-12704


# Generate product videos from images with BrowserAct and Google Drive

## 1. Workflow Overview

**Workflow name:** BrowserAct Product Image to AI Video Automation  
**Purpose:** Collect product records (including image URLs) via a BrowserAct browser automation, then for each product: download the image, convert it to Base64, submit it to an AI video generation API (Google Generative Language “Veo” long-running operation), poll until completion, download the generated video, and upload both source image and output video to Google Drive.

### 1.1 Data Collection (Browser automation → structured product items)
- Runs a BrowserAct workflow that returns a JSON string of product objects.
- Parses that string into individual n8n items.

### 1.2 Batching & Iteration Control
- Limits number of products processed (currently **2**).
- Iterates product-by-product using Split in Batches.

### 1.3 Image Preparation (download → Base64 → binary file)
- Downloads each product image from its URL.
- Converts the response into a Base64 string for the video API.
- Converts the Base64 back into a binary file for Drive upload (source image archiving).

### 1.4 Video Generation (submit long-running job → wait/poll → download)
- Builds a fixed prompt.
- Submits a long-running video generation request (8s, 9:16).
- Waits 120 seconds, checks status, loops until `done=true`.
- Downloads the final video file from the returned `uri`.

### 1.5 Google Drive Sync (upload source + upload video)
- Uploads the source image to Google Drive root.
- Uploads the generated video to a specific folder.
- Continues to next product until batch finishes.

---

## 2. Block-by-Block Analysis

### Block A — Trigger & BrowserAct Data Collection
**Overview:** Manually starts the workflow, then executes a BrowserAct workflow to scrape/collect product data in a real browser and returns structured output for downstream steps.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Run a workflow (BrowserAct)
- Parse BrowserAct output

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; entry point for testing and ad-hoc runs.
- **Configuration:** No parameters.
- **Outputs:** One empty trigger item to **Run a workflow**.
- **Edge cases:** None (only manual execution).

#### Node: Run a workflow
- **Type / role:** `browseract.browserAct`; runs a BrowserAct hosted workflow.
- **Configuration choices:**
  - Operation: **WORKFLOW**
  - Workflow ID: `73948758645931553`
  - Workflow config mapping schema exists but inputs are marked “removed”; defaults defined in BrowserAct are used unless re-enabled.
- **Credentials:** BrowserAct account credential required.
- **Input → Output:**
  - Input: trigger item.
  - Output: BrowserAct result object; downstream code node expects **`$json.output.string`** to contain JSON text.
- **Failure types / edge cases:**
  - Auth/credential invalid.
  - BrowserAct workflow fails or returns non-JSON text.
  - Output schema changes (e.g., `output.string` missing).

#### Node: Parse BrowserAct output
- **Type / role:** Code node; parses BrowserAct output JSON string into n8n items.
- **Key logic:**
  - Reads `const raw = $json.output.string;`
  - `JSON.parse(raw)` with explicit error handling.
  - Ensures parsed result is an **array**; otherwise throws.
  - Returns `products.map(item => ({ json: item }))`
- **Input → Output connections:**
  - Input: from **Run a workflow**
  - Output: array of product items → **limit_products**
- **Failure types / edge cases:**
  - `output.string` missing/undefined.
  - Invalid JSON (throws “JSON parse failed…”).
  - Parsed object not an array (throws “Parsed data is not an array”).
  - Very large arrays may increase runtime/memory.

**Sticky note context (covers this area):**
- **“Title：Data collection”** — Explains BrowserAct collection step and that it outputs product records with image URLs/metadata.

---

### Block B — Product Limiting & Iteration
**Overview:** Controls how many products are processed and iterates through them one-by-one, enabling scalable batch processing.

**Nodes involved:**
- limit_products
- iterate_products (Split in Batches)

#### Node: limit_products
- **Type / role:** Limit; caps number of items passed forward.
- **Configuration choices:** `maxItems: 2`
- **Input → Output:**
  - Input: parsed product items.
  - Output: first 2 items → **iterate_products**
- **Edge cases:**
  - If fewer than 2 items exist, passes all available items (normal).
  - If ordering matters, ensure BrowserAct output is already sorted as desired.

#### Node: iterate_products
- **Type / role:** Split In Batches; iterates products in batches (default batch size if not set).
- **Configuration choices:** No explicit batch size set in parameters shown; n8n defaults apply.
- **Connections / flow pattern:**
  - From **limit_products** → **iterate_products**
  - **Second output (continue)** routes to both:
    - **iterate_products** (to fetch next batch/item)
    - **fetch_image** (to process current item)
  - After finishing a product and uploading the output video, flow returns to **iterate_products** to continue.
- **Edge cases / failure types:**
  - Misconfigured batch size can cause unexpected iteration behavior.
  - If downstream node errors mid-loop, remaining products won’t process unless error handling is added.

---

### Block C — Image Preparation (Download → Base64 → Binary → Upload source)
**Overview:** For each product, downloads the product image, converts it to Base64 for the video API payload, then converts it into binary for uploading as a source image file to Google Drive.

**Nodes involved:**
- fetch_image
- Convert image to Base64
- Prepare image for video generation
- upload_source_image

#### Node: fetch_image
- **Type / role:** HTTP Request; downloads the image from the product record.
- **Configuration choices:**
  - URL: `={{ $json.image_url }}`
  - Method defaults to GET.
- **Input → Output:**
  - Input: current product item from **iterate_products**
  - Output: response body stored in `$json.data` (expected by next nodes)
- **Edge cases / failure types:**
  - `image_url` missing or invalid.
  - 403/404/timeout; remote server blocks scraping.
  - Non-image content returned (HTML, JSON, etc.).
  - Large images may exceed memory/time limits.

#### Node: Convert image to Base64
- **Type / role:** Extract From File; here used as “binaryToPropery” (binary → JSON property).
- **Configuration choices:**
  - Operation: **binaryToPropery**
  - Expects incoming item to have binary data; in practice, this depends on how `fetch_image` returns data in your n8n version/config.
- **Output:** Produces Base64 string at **`$node['Convert image to Base64'].json.data`** (used later in the video API call).
- **Edge cases:**
  - If `fetch_image` does not produce binary, conversion fails.
  - If response isn’t treated as file/binary, you may need `responseFormat: file` in `fetch_image`.

#### Node: Prepare image for video generation
- **Type / role:** Convert to File; converts a JSON property into binary.
- **Configuration choices:**
  - Operation: **toBinary**
  - Source property: `data`
- **Role in this workflow:** Creates a binary file version of the image so it can be uploaded to Google Drive as a `.jpg`.
- **Edge cases:**
  - If `data` is not present or not valid base64/content, conversion fails.

#### Node: upload_source_image
- **Type / role:** Google Drive; uploads the source image.
- **Configuration choices:**
  - File name: `={{ $("iterate_products").item.json.title }}.jpg`
  - Drive: **My Drive**
  - Folder: **root** (Drive root)
- **Credentials:** Google Drive OAuth2 required.
- **Input → Output:** Receives binary from **Prepare image for video generation**, uploads to Drive, then continues to **set_prompt**.
- **Edge cases / failure types:**
  - Missing `title` results in filenames like `.jpg` or `undefined.jpg`.
  - Google OAuth token expired/insufficient permissions.
  - Upload fails if no binary property exists.

**Sticky note context (covers this area):**
- **“Title：Image preparation”** — Explains downloading and converting images into the required format for the API.

---

### Block D — Video Request Construction & Submission
**Overview:** Creates a fixed prompt, then sends a long-running request to Google’s Generative Language API using the image Base64 content.

**Nodes involved:**
- set_prompt
- Submit video generation request

#### Node: set_prompt
- **Type / role:** Set; injects a constant prompt for video generation.
- **Configuration choices:**
  - Sets field `prompt` to:  
    “Generate a professional 8-second e-commerce showcase video… slow, elegant 360-degree pan…”
- **Input → Output:** Input from **upload_source_image**, output to **Submit video generation request**.
- **Edge cases:**
  - Overwrites existing `prompt` field if already present in product item.

#### Node: Submit video generation request
- **Type / role:** HTTP Request; submits the Veo long-running generation job.
- **Configuration choices:**
  - POST `https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-generate-preview:predictLongRunning`
  - Auth: **genericCredentialType → httpHeaderAuth** (expects an Authorization header, typically `Bearer <token>`).
  - JSON body includes:
    - `prompt`: `{{ $json.prompt }}`
    - `image.bytesBase64Encoded`: `{{ $node['Convert image to Base64'].json.data }}`
    - `parameters.durationSeconds`: 8
    - `parameters.aspectRatio`: `"9:16"`
- **Output expectations:**
  - Must return an operation resource with a `name` field (used by the status check).
- **Failure types / edge cases:**
  - Invalid/expired API key or OAuth token in header auth.
  - Payload too large (image size limits).
  - Model endpoint availability/version changes.
  - If response doesn’t contain `name`, polling step breaks.

---

### Block E — Async Polling Loop (Wait → Check → If done)
**Overview:** Implements an asynchronous polling pattern: wait 120 seconds, check operation status, and loop until the API returns `done: true`.

**Nodes involved:**
- Wait for video generation
- Check video generation status
- check_response (IF)

#### Node: Wait for video generation
- **Type / role:** Wait; pauses execution for a fixed delay.
- **Configuration choices:**
  - Amount: **120** seconds
- **Connections:**
  - From **Submit video generation request** → Wait
  - From **check_response (false path)** → Wait (loop)
  - Wait → **Check video generation status**
- **Edge cases:**
  - Too short wait increases API calls; too long increases overall runtime.
  - Workflow execution time limits (on some hosting plans) can be hit.

#### Node: Check video generation status
- **Type / role:** HTTP Request; polls operation endpoint.
- **Configuration choices:**
  - GET `=https://generativelanguage.googleapis.com/v1beta/{{ $json.name }}`
  - Auth: same `httpHeaderAuth`
- **Input → Output:**
  - Input: output from Wait, which carries forward the operation `name`.
  - Output is expected to contain `done` boolean and, when done, nested response content.
- **Edge cases:**
  - If `$json.name` is missing (bad submit response), URL becomes invalid.
  - API may return transient errors; no retry/backoff configured.

#### Node: check_response
- **Type / role:** IF; decides whether to proceed to download.
- **Condition:** `{{ $json.done }}` **equals** `true`
- **Outputs:**
  - **True path →** download_video
  - **False path →** Wait for video generation (loop)
- **Edge cases:**
  - `done` missing or not boolean; strict validation may fail or evaluate false.
  - Operation may complete with error; some APIs return `done:true` plus an error object—this workflow does not explicitly handle that.

**Sticky note context (covers this area):**
- **“Title：Video generation”** — Notes that the asynchronous Wait step is required and should not be removed.

---

### Block F — Download Generated Video & Upload to Google Drive
**Overview:** Once generation completes, downloads the video file from the returned URI and uploads it to a Google Drive folder, then continues iteration for the next product.

**Nodes involved:**
- download_video
- upload_output_video

#### Node: download_video
- **Type / role:** HTTP Request; downloads the produced video file as binary.
- **Configuration choices:**
  - URL: `={{ $node["Check video generation status"].json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
  - Response format: **file** (binary)
  - Auth: `httpHeaderAuth`
- **Input → Output:** True path of IF → downloads binary → **upload_output_video**
- **Edge cases / failure types:**
  - The referenced path may differ depending on API response shape; if missing, expression fails.
  - `generatedSamples[0]` may not exist.
  - Video URI may require different auth than the API (signed URL vs bearer token).

#### Node: upload_output_video
- **Type / role:** Google Drive; uploads the `.mp4` output video.
- **Configuration choices:**
  - File name: `={{ $("iterate_products").item.json.title }}.mp4`
  - Folder: ID `1lRC0KN0DBl9lljHzPN1NyEVF60_xorMB` (cached name “n8n video”)
- **Credentials:** Google Drive OAuth2.
- **Output / loopback:** After upload, connects back to **iterate_products** to process the next product.
- **Edge cases:**
  - Same filename collisions if titles repeat.
  - Missing binary data (if download failed).
  - Drive permissions to target folder.

**Sticky note context (covers this area):**
- **“Title：Google Drive sync”** — Upload source images and generated videos to Drive.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual start | — | Run a workflow | Title：Data collection |
| Run a workflow | browseract.browserAct | Run BrowserAct extraction workflow | When clicking ‘Execute workflow’ | Parse BrowserAct output | Title：Data collection |
| Parse BrowserAct output | code | Parse JSON string into product items | Run a workflow | limit_products | Title：Data collection |
| limit_products | limit | Cap number of product items | Parse BrowserAct output | iterate_products |  |
| iterate_products | splitInBatches | Iterate products sequentially | limit_products, upload_output_video | fetch_image (+ loop to itself) |  |
| fetch_image | httpRequest | Download product image from URL | iterate_products | Convert image to Base64 | Title：Image preparation |
| Convert image to Base64 | extractFromFile | Convert image file/binary to Base64 property | fetch_image | Prepare image for video generation | Title：Image preparation |
| Prepare image for video generation | convertToFile | Convert Base64/data property to binary file | Convert image to Base64 | upload_source_image | Title：Image preparation |
| upload_source_image | googleDrive | Upload source image to Drive | Prepare image for video generation | set_prompt | Title：Google Drive sync |
| set_prompt | set | Add fixed text prompt | upload_source_image | Submit video generation request |  |
| Submit video generation request | httpRequest | Start long-running video generation operation | set_prompt | Wait for video generation | Title：Video generation |
| Wait for video generation | wait | Delay between polling attempts | Submit video generation request, check_response (false) | Check video generation status | Title：Video generation |
| Check video generation status | httpRequest | Poll operation status by name | Wait for video generation | check_response | Title：Video generation |
| check_response | if | Branch when `done=true` | Check video generation status | download_video (true), Wait for video generation (false) | Title：Video generation |
| download_video | httpRequest | Download generated video binary | check_response (true) | upload_output_video | Title：Video generation |
| upload_output_video | googleDrive | Upload .mp4 to Drive folder | download_video | iterate_products | Title：Google Drive sync |
| Sticky Note | stickyNote | Global explanation & setup steps | — | — | How it works… (full note content) |
| Sticky Note1 | stickyNote | Block label: Data collection | — | — | Title：Data collection… |
| Sticky Note4 | stickyNote | Block label: Image preparation | — | — | Title：Image preparation… |
| Sticky Note3 | stickyNote | Block label: Video generation | — | — | Title：Video generation… |
| Sticky Note2 | stickyNote | Block label: Google Drive sync | — | — | Title：Google Drive sync… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (inactive by default).
2. **Add Manual Trigger** node:
   - Node type: *Manual Trigger*
   - Name: `When clicking ‘Execute workflow’`

3. **Add BrowserAct node**:
   - Node type: *BrowserAct*
   - Name: `Run a workflow`
   - Operation/Type: **WORKFLOW**
   - Workflow ID: `73948758645931553`
   - Credentials: create/select **BrowserAct account** credential.
   - Connect: Manual Trigger → BrowserAct.

4. **Add Code node to parse output**:
   - Node type: *Code*
   - Name: `Parse BrowserAct output`
   - Paste logic that:
     - reads `$json.output.string`
     - `JSON.parse`
     - verifies array
     - returns `products.map(item => ({ json: item }))`
   - Connect: BrowserAct → Code.

5. **Add Limit node**:
   - Node type: *Limit*
   - Name: `limit_products`
   - Max items: **2** (adjust as needed)
   - Connect: Code → Limit.

6. **Add Split In Batches node**:
   - Node type: *Split in Batches*
   - Name: `iterate_products`
   - Leave default options (or set batch size explicitly if desired).
   - Connect: Limit → Split in Batches.
   - Ensure the Split in Batches “next batch” loop is used later (step 15).

7. **Add HTTP Request to download image**:
   - Node type: *HTTP Request*
   - Name: `fetch_image`
   - Method: GET
   - URL expression: `{{ $json.image_url }}`
   - Connect: Split in Batches (output that processes current item) → fetch_image.

8. **Add Extract From File for Base64**:
   - Node type: *Extract From File*
   - Name: `Convert image to Base64`
   - Operation: **Binary to Property** (shown as `binaryToPropery`)
   - Connect: fetch_image → Convert image to Base64.
   - If this fails in your environment, change `fetch_image` to return a file/binary response so this node has binary input.

9. **Add Convert to File**:
   - Node type: *Convert to File*
   - Name: `Prepare image for video generation`
   - Operation: **toBinary**
   - Source Property: `data`
   - Connect: Convert image to Base64 → Prepare image for video generation.

10. **Add Google Drive node to upload source image**:
    - Node type: *Google Drive*
    - Name: `upload_source_image`
    - Operation: upload file (default in this node setup)
    - File name: `{{ $("iterate_products").item.json.title }}.jpg`
    - Drive: My Drive
    - Folder: root (or choose a folder)
    - Credentials: create/select **Google Drive OAuth2** credential.
    - Connect: Prepare image for video generation → upload_source_image.

11. **Add Set node for prompt**:
    - Node type: *Set*
    - Name: `set_prompt`
    - Add field `prompt` with the desired prompt text (the workflow uses a fixed 8-second studio 360-pan prompt).
    - Connect: upload_source_image → set_prompt.

12. **Add HTTP Request to submit video generation**:
    - Node type: *HTTP Request*
    - Name: `Submit video generation request`
    - Method: POST
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-generate-preview:predictLongRunning`
    - Authentication: **Header Auth** (generic credential type)
      - Create `httpHeaderAuth` credential containing required header(s), typically `Authorization: Bearer <TOKEN>` (or the scheme required by your API setup).
    - Body: JSON, containing:
      - prompt from `{{$json.prompt}}`
      - image bytes from `{{$node['Convert image to Base64'].json.data}}`
      - parameters: `durationSeconds: 8`, `aspectRatio: "9:16"`
    - Connect: set_prompt → Submit video generation request.

13. **Add Wait node**:
    - Node type: *Wait*
    - Name: `Wait for video generation`
    - Wait time: **120 seconds**
    - Connect: Submit video generation request → Wait.

14. **Add HTTP Request to poll status**:
    - Node type: *HTTP Request*
    - Name: `Check video generation status`
    - Method: GET
    - URL expression: `https://generativelanguage.googleapis.com/v1beta/{{ $json.name }}`
    - Authentication: same **Header Auth** credential as submit.
    - Connect: Wait → Check video generation status.

15. **Add IF node to check completion**:
    - Node type: *IF*
    - Name: `check_response`
    - Condition: Boolean → `{{ $json.done }}` equals `true`
    - Connect: Check video generation status → IF.
    - Connect IF **false** output → Wait (to keep polling).

16. **Add HTTP Request to download the video**:
    - Node type: *HTTP Request*
    - Name: `download_video`
    - URL expression (as in workflow):  
      `{{ $node["Check video generation status"].json.response.generateVideoResponse.generatedSamples[0].video.uri }}`
    - Response: set to **File** (binary).
    - Authentication: Header Auth (if required by the video URI).
    - Connect IF **true** output → download_video.

17. **Add Google Drive node to upload video**:
    - Node type: *Google Drive*
    - Name: `upload_output_video`
    - File name: `{{ $("iterate_products").item.json.title }}.mp4`
    - Folder: select target folder (the workflow uses folder ID `1lRC0KN0DBl9lljHzPN1NyEVF60_xorMB`)
    - Connect: download_video → upload_output_video.

18. **Close the loop for next product**:
    - Connect: upload_output_video → iterate_products (to request next item/batch).

19. **(Optional but recommended) Add error handling**:
    - Add retries on HTTP nodes.
    - Handle “done=true with error” responses explicitly.
    - Validate `image_url` and `title` before use.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How it works: Uses BrowserAct to collect product data, iterates products, downloads image, converts to Base64, calls AI video generation API, polls status (Wait should not be removed), downloads video, uploads image+video to Google Drive; includes setup steps (connect BrowserAct + Google Drive, review BrowserAct inputs, adjust limits, test run). | From the large “How it works” sticky note inside the workflow |
| Title：Data collection — Run a BrowserAct workflow to collect structured product items using real browser execution; outputs product records containing image URLs and metadata. | Sticky note “Title：Data collection” |
| Title：Image preparation — Download each product image and prepare it for AI video generation; fetched individually and converted into required format. | Sticky note “Title：Image preparation” |
| Title：Video generation — Submit request with image+prompt; wait and repeatedly check status until ready; asynchronous pattern required and Wait should not be removed. | Sticky note “Title：Video generation” |
| Title：Google Drive sync — Upload source images and generated videos to Google Drive. | Sticky note “Title：Google Drive sync” |