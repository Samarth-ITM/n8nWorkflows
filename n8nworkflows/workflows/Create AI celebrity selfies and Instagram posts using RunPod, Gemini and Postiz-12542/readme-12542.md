Create AI celebrity selfies and Instagram posts using RunPod, Gemini and Postiz

https://n8nworkflows.xyz/workflows/create-ai-celebrity-selfies-and-instagram-posts-using-runpod--gemini-and-postiz-12542


# Create AI celebrity selfies and Instagram posts using RunPod, Gemini and Postiz

## 1. Workflow Overview

**Purpose:**  
This workflow creates an AI-edited “selfie with a celebrity” image using **RunPod (Nano Banana Pro Edit)**, generates a **viral Instagram caption + hashtags** with **Google Gemini**, stores the final image in **Google Drive**, and publishes it to **Instagram** through **Postiz**.

**Primary use cases:**
- Fast generation of celebrity-style selfie edits from a user-provided image URL and prompt
- Automated social copywriting based on the final image
- One-click publishing to an Instagram-connected social scheduler (Postiz)

### 1.1 Logical Blocks
1. **Input Reception & Normalization**
   - Form submission collects image URL, prompt, and aspect ratio; data is normalized into consistent field names.
2. **AI Image Generation (RunPod) + Polling**
   - Submits a RunPod job, waits, polls job status until completed.
3. **Post-Processing: Caption Generation (Gemini) + Image Retrieval**
   - Once completed, Gemini analyzes the resulting image and returns caption/hashtags; the image file is downloaded.
4. **Aggregation & Publishing**
   - Caption text and uploaded image reference are merged, image is uploaded to Postiz, and a social post is created.
5. **Archival**
   - Image is uploaded to a specified Google Drive folder for storage.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization

**Overview:**  
Collects user inputs (selfie URL, creative prompt, and aspect ratio) via an n8n Form Trigger, then maps them into standardized JSON keys used downstream.

**Nodes Involved:**  
- **On form submission**
- **Set data**

#### Node: On form submission
- **Type / Role:** `n8n-nodes-base.formTrigger` — entry point; starts workflow on form submission.
- **Configuration (interpreted):**
  - Form title: **“Selfie with Celebrities”**
  - Fields:
    - `IMAGE_URL` (required): URL to the user’s selfie
    - `PROMPT` (required): text prompt for the edit (default provided)
    - `FORMAT` dropdown (required): aspect ratio (`1:1`, `4:3`, `9:16`, `16:9`; default `9:16`)
- **Key variables produced:** `$json.IMAGE_URL`, `$json.PROMPT`, `$json.FORMAT`
- **Connections:**
  - Output → **Set data**
- **Edge cases / failures:**
  - Invalid or inaccessible `IMAGE_URL` (403/404/hotlink protection) will later break RunPod or image download.
  - Very large images or slow hosts may cause RunPod job issues (timeouts on their side) or later download issues.

#### Node: Set data
- **Type / Role:** `n8n-nodes-base.set` — maps form fields to internal keys used across the workflow.
- **Configuration (interpreted):**
  - Creates:
    - `image_url` = `{{$json.IMAGE_URL}}`
    - `prompt` = `{{$json.PROMPT}}`
    - `aspect_ratio` = `{{$json.FORMAT}}`
- **Connections:**
  - Input: **On form submission**
  - Output → **Generate selfie**
- **Edge cases / failures:**
  - If a form field name changes, expressions will resolve to empty strings and RunPod call becomes invalid.

**Sticky note coverage (context):**
- “## STEP 1 - Set prompt and your selfie / Sends the image and prompt”

---

### Block 2 — AI Image Generation (RunPod) + Polling

**Overview:**  
Submits a RunPod inference job to create the edited selfie. Because sync mode is disabled, the workflow waits and polls the job status until it returns `COMPLETED`.

**Nodes Involved:**  
- **Generate selfie**
- **Wait 20 sec.**
- **Get status clip**
- **Completed?**

#### Node: Generate selfie
- **Type / Role:** `n8n-nodes-base.httpRequest` — creates a RunPod job (async).
- **Configuration (interpreted):**
  - POST `https://api.runpod.ai/v2/nano-banana-pro-edit/run`
  - Body (JSON) includes:
    - `prompt`: `{{$json.prompt}}`
    - `images`: `[ "{{$json.image_url}}" ]`
    - `resolution`: `"1k"`
    - `output_format`: `"jpeg"`
    - `enable_base64_output`: `false`
    - `enable_sync_mode`: `false` (forces asynchronous execution)
    - `aspect_ratio`: `{{$json.aspect_ratio}}`
  - Header: `Content-Type: application/json`
  - Auth: **HTTP Bearer Auth** credential (RunPod)
- **Expected output:** includes a RunPod `id` used to poll status.
- **Connections:**
  - Output → **Wait 20 sec.**
- **Failure types / edge cases:**
  - 401/403 if Bearer token missing/invalid.
  - 400 if prompt/images/aspect ratio invalid or empty.
  - RunPod model endpoint name `nano-banana-pro-edit` must exist for the account.
  - If user image URL is blocked, RunPod may fail the job (later status may show FAILED).

#### Node: Wait 20 sec.
- **Type / Role:** `n8n-nodes-base.wait` — delay between polling attempts.
- **Configuration:** waits **20 seconds**.
- **Connections:**
  - Input: **Generate selfie** or **Completed? (false path)**
  - Output → **Get status clip**
- **Edge cases:**
  - Workflow may loop for a long time if job never completes (no max retries implemented).

#### Node: Get status clip
- **Type / Role:** `n8n-nodes-base.httpRequest` — polls RunPod job status.
- **Configuration (interpreted):**
  - GET `https://api.runpod.ai/v2/nano-banana-pro-edit/status/{{ $json.id }}`
  - Auth: **HTTP Bearer Auth** (RunPod)
- **Connections:**
  - Output → **Completed?**
- **Failure types / edge cases:**
  - If `id` is missing (bad prior response), URL becomes invalid.
  - 404 if the job ID is not found (wrong endpoint/account).

#### Node: Completed?
- **Type / Role:** `n8n-nodes-base.if` — branching on job completion state.
- **Condition:** `{{$json.status}}` equals `"COMPLETED"`
- **Connections:**
  - **True path** → **Get image** AND **SMC Agent** (both triggered)
  - **False path** → **Wait 20 sec.** (poll again)
- **Failure types / edge cases:**
  - If RunPod returns `FAILED` or `CANCELLED`, this node keeps looping forever (since only checks for `COMPLETED`).
  - If `status` field missing/renamed, strict validation may cause condition to fail and loop indefinitely.

**Sticky note coverage (context):**
- “## STEP 2 - Generate image with celebrities / Add RunPod API credentials under httpBearerAuth”

---

### Block 3 — Post-Processing: Caption Generation (Gemini) + Image Retrieval

**Overview:**  
Once RunPod completes, the workflow (a) downloads the generated image and (b) sends the image URL to Gemini for visual analysis and caption/hashtag generation.

**Nodes Involved:**  
- **Get image**
- **SMC Agent**

#### Node: SMC Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini multimodal image analysis to generate Instagram copy.
- **Configuration (interpreted):**
  - Resource: **image**
  - Operation: **analyze**
  - Model: `models/gemini-2.5-flash`
  - Image URL: `{{$json.output.result}}` (from RunPod completed status payload)
  - Prompt: detailed instruction set:
    - analyze image details
    - define viral angle
    - write caption with hook + CTA
    - add 5–10 hashtags
    - do not mention AI-generated
    - max 2–3 emojis
  - Credential: Google Gemini / PaLM API
- **Expected output:** text content (caption + hashtags) in a Gemini response structure (later referenced as `$('Merge').item.json.content.parts[0].text`).
- **Connections:**
  - Output → **Merge** (input index 0)
- **Failure types / edge cases:**
  - Invalid/expired Gemini credentials.
  - Model name availability can vary by region/account.
  - If the RunPod `output.result` is not publicly reachable, Gemini may fail to fetch the image.
  - Output schema can differ by node version; the downstream expression assumes `content.parts[0].text` exists.

#### Node: Get image
- **Type / Role:** `n8n-nodes-base.httpRequest` — downloads the final image file.
- **Configuration (interpreted):**
  - GET `{{$json.output.result}}`
  - (No explicit “download as binary” setting shown in JSON; in practice this must output binary for later “Upload file” and “Upload to Postiz”.)
- **Connections:**
  - Output → **Merge** (input index 1)
  - Output → **Upload file**
- **Failure types / edge cases:**
  - If `output.result` is a presigned URL that expires, downloads can fail depending on timing.
  - If node is not configured to return **binary**, downstream upload nodes expecting binary field `data` will fail.

**Sticky note coverage (context):**
- “## STEP 3 - Social Media Content Agent / Agenerate a viral-ready Instagram caption and hashtags”

---

### Block 4 — Aggregation & Publishing

**Overview:**  
Combines the Gemini-generated caption with the downloaded image, uploads the image to Postiz, then creates a Postiz post targeted at a specific channel/integration ID (Instagram).

**Nodes Involved:**  
- **Merge**
- **Upload to Postiz**
- **Upload to Social**

#### Node: Merge
- **Type / Role:** `n8n-nodes-base.merge` — combines outputs from Gemini and image download.
- **Configuration (interpreted):**
  - Mode: **combine**
  - Combine by: **combineAll**
  - This produces a single item containing fields from both inputs.
- **Connections:**
  - Input 0: **SMC Agent**
  - Input 1: **Get image**
  - Output → **Upload to Postiz**
- **Failure types / edge cases:**
  - If one branch never returns (Gemini fails or image download fails), merge will not emit output.
  - If binary data is not preserved through merge (depending on configuration/version), Postiz upload may not receive the expected binary field.

#### Node: Upload to Postiz
- **Type / Role:** `n8n-nodes-base.httpRequest` — uploads media to Postiz to obtain a media `id` and `path`.
- **Configuration (interpreted):**
  - POST `https://api.postiz.com/public/v1/upload`
  - Content-Type: `multipart-form-data`
  - Body: form-binary field:
    - `file` from binary property `data`
  - Auth: **HTTP Header Auth** credential (Postiz)
- **Expected output:** JSON containing uploaded media `id` and `path` (used next).
- **Connections:**
  - Output → **Upload to Social**
- **Failure types / edge cases:**
  - Missing header token / invalid Postiz API key → 401/403.
  - If binary field name is not `data`, upload fails.
  - If Merge didn’t carry the binary forward, file will be empty.

#### Node: Upload to Social
- **Type / Role:** `n8n-nodes-postiz.postiz` — creates/schedules a social post in Postiz.
- **Configuration (interpreted):**
  - Scheduled datetime: `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}` (immediate “now” timestamp in ISO-like form)
  - Post payload:
    - Content text: `{{ $('Merge').item.json.content.parts[0].text }}`
    - Media:
      - `id`: `{{$json.id}}`
      - `path`: `{{$json.path}}`
    - `integrationId`: `"XXX"` (**must be replaced** with your real Postiz channel/integration ID)
    - `shortLink`: true
  - Credential: Postiz API account (separate from header-auth used in upload)
- **Connections:**
  - Input: **Upload to Postiz**
  - Output: end
- **Failure types / edge cases:**
  - Placeholder `integrationId: "XXX"` will cause posting to fail until replaced.
  - Expression references `$('Merge')...` assumes Merge node exists in the same execution and has the Gemini output.
  - Time format uses `HH:ii:ss`; if your n8n date formatter expects `mm` for minutes (varies), scheduling could be malformed.

**Sticky note coverage (context):**
- “## STEP 5 - Upload to Instagram / Create a FREE account on Postiz and Set Channel_ID from your Dashboard”

---

### Block 5 — Archival (Google Drive)

**Overview:**  
Uploads the downloaded image to a designated Google Drive folder (“Runpod”) with a timestamped filename.

**Nodes Involved:**  
- **Upload file**

#### Node: Upload file
- **Type / Role:** `n8n-nodes-base.googleDrive` — uploads binary file to Google Drive.
- **Configuration (interpreted):**
  - Operation: upload (binary)
  - File name:
    - `image_{{$now.format('yyyyLLddHHiiss')}}_{{$binary.data.fileName}}`
  - Drive: “My Drive”
  - Folder: ID `1xT3rJm2PMjOTsSQFk6QP_Tt6J8z2R0x4` (displayed as “Runpod”)
  - Credential: Google Drive OAuth2
- **Connections:**
  - Input: **Get image**
  - Output: end
- **Failure types / edge cases:**
  - If `$binary.data.fileName` is missing (common when downloading), filename expression may resolve incorrectly.
  - Google OAuth token expiration or insufficient permissions to folder.
  - If Get image did not output binary `data`, upload fails.

**Sticky note coverage (context):**
- “## STEP 5 - Upload to Drive / Upload the image to Google Drive”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | formTrigger | Entry point; collects IMAGE_URL, PROMPT, FORMAT | — | Set data | ## STEP 1 - Set prompt and your selfie / Sends the image and prompt |
| Set data | set | Normalizes input field names | On form submission | Generate selfie | ## STEP 1 - Set prompt and your selfie / Sends the image and prompt |
| Generate selfie | httpRequest | Submits async RunPod job for image edit | Set data | Wait 20 sec. | ## STEP 2 - Generate image with celebrities / Add [RunPod API](https://get.runpod.io/n3witalia) credentials under `httpBearerAuth` |
| Wait 20 sec. | wait | Delay between status polls | Generate selfie; Completed? (false) | Get status clip | ## STEP 2 - Generate image with celebrities / Add [RunPod API](https://get.runpod.io/n3witalia) credentials under `httpBearerAuth` |
| Get status clip | httpRequest | Polls RunPod job status | Wait 20 sec. | Completed? | ## STEP 2 - Generate image with celebrities / Add [RunPod API](https://get.runpod.io/n3witalia) credentials under `httpBearerAuth` |
| Completed? | if | Branch: completed vs keep polling | Get status clip | (true) Get image, SMC Agent; (false) Wait 20 sec. | ## STEP 2 - Generate image with celebrities / Add [RunPod API](https://get.runpod.io/n3witalia) credentials under `httpBearerAuth` |
| SMC Agent | googleGemini | Analyzes final image and generates caption + hashtags | Completed? (true) | Merge | ## STEP 3 - Social Media Content Agent / Agenerate a viral-ready Instagram caption and hashtags |
| Get image | httpRequest | Downloads final image | Completed? (true) | Merge; Upload file | ## STEP 3 - Social Media Content Agent / Agenerate a viral-ready Instagram caption and hashtags |
| Merge | merge | Combines caption text + image/binary into one item | SMC Agent; Get image | Upload to Postiz |  |
| Upload to Postiz | httpRequest | Uploads binary image to Postiz media endpoint | Merge | Upload to Social | ## STEP 5 - Upload to Instagram / Create a FREE account on [Postiz](https://postiz.com/?ref=n3witalia) and Set Channel_ID from your Dashboard |
| Upload to Social | postiz | Creates/schedules Instagram post in Postiz | Upload to Postiz | — | ## STEP 5 - Upload to Instagram / Create a FREE account on [Postiz](https://postiz.com/?ref=n3witalia) and Set Channel_ID from your Dashboard |
| Upload file | googleDrive | Archives image to Google Drive folder | Get image | — | ## STEP 5 - Upload to Drive / Upload the image to Google Drive |
| Sticky Note | stickyNote | Comment block | — | — | ## Viral Selfie Image With Celebrities using Nano Banana Pro & upload to Instagram … includes links: [RunPod](https://get.runpod.io/n3witalia), [Postiz](https://affiliate.postiz.com/n3witalia) |
| Sticky Note1 | stickyNote | Comment block | — | — | ## START ![image](https://n3wstorage.b-cdn.net/n3witalia/result3.png) |
| Sticky Note2 | stickyNote | Comment block | — | — | ## RESULT ![image](https://n3wstorage.b-cdn.net/n3witalia/result_lbj.jpeg) |
| Sticky Note3 | stickyNote | Comment block | — | — | ## STEP 1 - Set prompt and your selfie / Sends the image and prompt |
| Sticky Note4 | stickyNote | Comment block | — | — | ## STEP 2 - Generate image with celebrities / Add [RunPod API](https://get.runpod.io/n3witalia) credentials under `httpBearerAuth` |
| Sticky Note5 | stickyNote | Comment block | — | — | ## STEP 3 - Social Media Content Agent / Agenerate a viral-ready Instagram caption and hashtags |
| Sticky Note6 | stickyNote | Comment block | — | — | ## STEP 5 - Upload to Drive / Upload the image to Google Drive |
| Sticky Note7 | stickyNote | Comment block | — | — | ## STEP 5 - Upload to Instagram / Create a FREE account on [Postiz](https://postiz.com/?ref=n3witalia) and Set Channel_ID from your Dashboard |
| Sticky Note8 | stickyNote | Comment block | — | — | ## MY NEW YOUTUBE CHANNEL 👉 https://youtube.com/@n3witalia (with cover image link) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Viral Selfie Image With Celebrities*.

2. **Add “On form submission” (Form Trigger)**
   - Title: `Selfie with Celebrities`
   - Add fields:
     - `IMAGE_URL` (Text, required)
     - `PROMPT` (Text, required) and paste your default prompt
     - `FORMAT` (Dropdown, required) with options: `1:1`, `4:3`, `9:16`, `16:9` (default `9:16`)
   - Keep the generated webhook/form URL for testing.

3. **Add “Set data” (Set node)**
   - Add 3 fields:
     - `image_url` = expression `{{$json.IMAGE_URL}}`
     - `prompt` = expression `{{$json.PROMPT}}`
     - `aspect_ratio` = expression `{{$json.FORMAT}}`
   - Connect: **On form submission → Set data**

4. **Add “Generate selfie” (HTTP Request)**
   - Method: **POST**
   - URL: `https://api.runpod.ai/v2/nano-banana-pro-edit/run`
   - Authentication: **Bearer Auth** (create credential “Runpods” with your RunPod API key)
   - Send body: **JSON**
   - Body (use expressions):
     - `input.prompt` = `{{$json.prompt}}`
     - `input.images` = array containing `{{$json.image_url}}`
     - `input.resolution` = `1k`
     - `input.output_format` = `jpeg`
     - `input.enable_base64_output` = `false`
     - `input.enable_sync_mode` = `false`
     - `input.aspect_ratio` = `{{$json.aspect_ratio}}`
   - Add header `Content-Type: application/json`
   - Connect: **Set data → Generate selfie**

5. **Add “Wait 20 sec.” (Wait node)**
   - Amount: `20 seconds`
   - Connect: **Generate selfie → Wait 20 sec.**

6. **Add “Get status clip” (HTTP Request)**
   - Method: **GET**
   - URL: `https://api.runpod.ai/v2/nano-banana-pro-edit/status/{{$json.id}}`
   - Authentication: **Bearer Auth** (same RunPod credential)
   - Connect: **Wait 20 sec. → Get status clip**

7. **Add “Completed?” (IF node)**
   - Condition: String equals
     - Left: `{{$json.status}}`
     - Right: `COMPLETED`
   - Connect: **Get status clip → Completed?**
   - On **false** branch connect back to **Wait 20 sec.** (to poll again)

8. **Add “SMC Agent” (Google Gemini node)**
   - Node type: **Google Gemini (LangChain)**
   - Resource: **image**, Operation: **analyze**
   - Model: `models/gemini-2.5-flash`
   - Image URL: `{{$json.output.result}}`
   - Text prompt: paste the provided instruction block (viral caption + hashtags rules)
   - Credential: set up **Google Gemini/PaLM API** credential
   - Connect: **Completed? (true) → SMC Agent**

9. **Add “Get image” (HTTP Request)**
   - Method: **GET**
   - URL: `{{$json.output.result}}`
   - Configure it to **download the response as binary** and store in binary field named **`data`** (important for the next upload steps).
   - Connect: **Completed? (true) → Get image**

10. **Add “Merge” (Merge node)**
   - Mode: **Combine**
   - Combine by: **Combine All**
   - Connect:
     - **SMC Agent → Merge (Input 1 / index 0)**
     - **Get image → Merge (Input 2 / index 1)**

11. **Add “Upload to Postiz” (HTTP Request)**
   - Method: **POST**
   - URL: `https://api.postiz.com/public/v1/upload`
   - Authentication: **Header Auth** (create Postiz header credential, typically an API key header as required by Postiz)
   - Content type: **multipart/form-data**
   - Add form field:
     - Name: `file`
     - Type: **Binary**
     - Binary property: `data`
   - Connect: **Merge → Upload to Postiz**

12. **Add “Upload to Social” (Postiz node)**
   - Credential: connect your **Postiz API** credential (account)
   - Date/time: `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
   - Post content:
     - Text: expression referencing Gemini output (match your Gemini node output structure; in this workflow it is)  
       `{{ $('Merge').item.json.content.parts[0].text }}`
   - Media:
     - `id` = `{{$json.id}}`
     - `path` = `{{$json.path}}`
   - Integration/Channel ID:
     - Replace `"XXX"` with your real **integrationId** from Postiz dashboard (Instagram channel)
   - Connect: **Upload to Postiz → Upload to Social**

13. **Add “Upload file” (Google Drive node)**
   - Credential: Google Drive OAuth2
   - Folder: select the target folder (or paste folder ID)
   - File name expression (optional):
     - `image_{{$now.format('yyyyLLddHHiiss')}}_{{$binary.data.fileName}}`
   - Binary property: `data`
   - Connect: **Get image → Upload file**

14. **Test end-to-end**
   - Submit the form with a publicly accessible image URL.
   - Confirm the polling loop reaches `COMPLETED`.
   - Verify:
     - Gemini returns caption text
     - Image downloads as binary `data`
     - Postiz upload returns `id` and `path`
     - Postiz post is created successfully
     - File appears in Google Drive folder

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow description includes service links for setup | RunPod: https://get.runpod.io/n3witalia ; Postiz: https://affiliate.postiz.com/n3witalia |
| Postiz account note (Instagram publishing) | https://postiz.com/?ref=n3witalia |
| Example “START” image reference | https://n3wstorage.b-cdn.net/n3witalia/result3.png |
| Example “RESULT” image reference | https://n3wstorage.b-cdn.net/n3witalia/result_lbj.jpeg |
| Author channel link | https://youtube.com/@n3witalia |

