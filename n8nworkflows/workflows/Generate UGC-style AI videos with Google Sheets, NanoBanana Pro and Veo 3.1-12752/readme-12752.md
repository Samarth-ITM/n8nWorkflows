Generate UGC-style AI videos with Google Sheets, NanoBanana Pro and Veo 3.1

https://n8nworkflows.xyz/workflows/generate-ugc-style-ai-videos-with-google-sheets--nanobanana-pro-and-veo-3-1-12752


# Generate UGC-style AI videos with Google Sheets, NanoBanana Pro and Veo 3.1

## 1. Workflow Overview

**Title:** Generate UGC-style AI videos with Google Sheets, NanoBanana Pro and Veo 3.1  
**Purpose:** This workflow automatically takes “tasks” from a Google Sheet, **composes a UGC-style selfie image** using 3 references (product + character + background) via **AtlasCloud (NanoBanana Pro Edit)**, then **turns the resulting image into an 8‑second vertical video** via **AtlasCloud (Veo 3.1 image-to-video)**. It writes results (prompts, outputs, analysis, errors) back to the same Google Sheet.

**Target use cases:** TikTok/Reels-style product videos, UGC ad creatives, product demos, rapid content generation pipelines driven by Google Sheets.

### 1.1 Logical Blocks
1. **(A) Image Editing Pipeline (hourly)**
   - Pull “Ready” rows from Google Sheets
   - Generate a concise selfie image prompt (LLM agent)
   - Upload 3 reference images to a temporary host and normalize URLs
   - Call AtlasCloud NanoBanana Pro Edit to generate the edited selfie image
   - If success: analyze image with OpenAI Vision and update Sheet as “Edited”
   - If error: update Sheet status “Error”

2. **(B) Video Generation Pipeline (hourly)**
   - Pull “Edited” rows from Google Sheets
   - Generate a Veo-compatible UGC video prompt (LLM agent)
   - Call AtlasCloud Veo 3.1 image-to-video to start generation
   - Poll job status until completed/failed
   - Update Sheet as “Finished” (or “Error” with message)

3. **(C) Shared AI Model Configuration**
   - Primary LLM: **OpenAI GPT-5-mini** (via LangChain ChatOpenAI node)
   - Fallback LLM: **Groq openai/gpt-oss-120b** (via LangChain Groq node)
   - OpenAI Vision model used for image analysis: **gpt-4o-mini**

---

## 2. Block-by-Block Analysis

### Block A — Edit Images: from “Ready” tasks to “Edited” tasks
**Overview:** Runs on a schedule, fetches all rows with `Status = Ready`, generates a short image prompt, rehosts the 3 input images to temporary direct-download URLs, sends them to AtlasCloud NanoBanana Pro Edit, then updates Google Sheets based on success/failure.

**Nodes involved:**
- Schedule Trigger Edit Images
- Get Ready Task
- Image Prompt Agent
- Get Image Array
- Split Out
- Download Images
- Get Temp Url
- Get Clean Data
- NanoBanana Pro Edit
- If Image Success
- Analyze image
- Update Edit Task [SUCCESS]
- Update Edit Task [ERROR]

#### 2.1 Schedule Trigger Edit Images
- **Type / role:** Schedule Trigger; initiates the image-edit pipeline periodically.
- **Config:** Every hour (`interval: hours`).
- **Outputs:** Triggers `Get Ready Task`.
- **Edge cases:** None typical; schedule timing depends on n8n instance uptime/timezone.

#### 2.2 Get Ready Task
- **Type / role:** Google Sheets node; reads rows to process.
- **Config choices:**
  - Filters: `Status` equals **Ready**
  - Returns potentially multiple matches (`returnFirstMatch: false`)
  - Uses a specific sheet tab named **Task**
- **Key fields expected per row:** `Task ID`, `Product ID`, `Product Photo`, `Character`, `Background`, `Product Name`, `Video Scene`, (and other optional columns).
- **Outputs:** Each matching row becomes an item, passed to `Image Prompt Agent`.
- **Failure types:** OAuth token expired, sheet URL not updated after duplication, missing columns, Google API quota.

#### 2.3 Image Prompt Agent
- **Type / role:** LangChain Agent node; produces an image-generation prompt.
- **Inputs:** The task row JSON.
- **Prompt composition (`text`):**
  - `Product: {{ Product Name }}`
  - `Scene: {{ Video Scene }}`
- **System message highlights:**
  - Must output **English only**
  - Concise (max 100 words), selfie-style, match references
  - Avoid detailed hand descriptions (“holding product naturally”)
- **Model routing:**
  - Uses **GPT-5-Mini** as primary language model (via `GPT-5-Mini` node).
  - Uses **GPT-OSS-120b** as fallback (via `GPT-OSS-120b` node).
- **Outputs:** Typically `output` field containing the final prompt; forwarded to `Get Image Array`.
- **Edge cases:** Empty/invalid agent output (handled later by `Get Clean Data` fallback), model auth failure, safety refusal.

#### 2.4 Get Image Array
- **Type / role:** Code node; builds an `images[]` array from the original sheet row(s).
- **Key logic:**
  - Reads **all items** from `Get Ready Task`
  - For each task row, returns:
    - `images: [Product Photo, Character, Background]`
- **Outputs:** Items each containing `json.images` (3 URLs).
- **Connections:** Feeds `Split Out`.
- **Edge cases:**
  - If any of the 3 columns is empty, downstream HTTP download/upload may fail.
  - It maps over *all* “Ready” tasks, which can increase run time if many rows match.

#### 2.5 Split Out
- **Type / role:** Split Out node; converts the `images` array into individual items.
- **Config:** `fieldToSplitOut = images`
- **Effect:** 3 items per task, each containing one `images` URL value.
- **Edge cases:** If `images` isn’t an array, it errors; if array length ≠ 3, later validation fails.

#### 2.6 Download Images
- **Type / role:** HTTP Request; downloads each image URL into binary for upload.
- **Config:** URL = `{{ $json.images }}` (the split-out URL).
- **Outputs:** Binary data (commonly in field `data` depending on n8n defaults).
- **Failure types:** 403/404 image URLs, timeouts, non-image content, very large images.

#### 2.7 Get Temp Url
- **Type / role:** HTTP Request; uploads binaries to tmpfiles.org to get a public URL.
- **Config choices:**
  - POST `https://tmpfiles.org/api/v1/upload`
  - `multipart-form-data` with form field `file` from binary input field `data`
  - **onError:** `continueErrorOutput` (workflow continues even if upload fails)
  - **retryOnFail:** false
- **Outputs:** JSON including `data.url` when successful.
- **Connections:** Sends output to both `Get Clean Data` **and** `Get Image Array` (second connection is unusual; see edge case below).
- **Edge cases / concerns:**
  - If upload fails for any image, `Get Clean Data` will throw when it doesn’t see 3 URLs.
  - The extra connection to `Get Image Array` can create a **loop-like dependency chain** in graph structure (though actual execution depends on item flow). It is not required for the pipeline and can cause confusion/maintenance risk.

#### 2.8 Get Clean Data
- **Type / role:** Code node; assembles final request payload for NanoBanana edit.
- **Inputs:**
  - All upload results from `Get Temp Url`
  - Original sheet data from `Get Ready Task` (first item)
  - Prompt from `Image Prompt Agent` (first item)
- **Key operations:**
  - Collect URLs: `item.json.data.url`
  - Normalize tmpfiles URLs:
    - `tmpfiles.org/` → `tmpfiles.org/dl/` (direct download)
    - `http://` → `https://`
  - Validate count: **must be exactly 3** or throw `Expected 3 images...`
  - Extract prompt from agent (`output || text || content || ''`)
  - Fallback prompt if empty
  - Enforce max **80 words** (note: system asked 100, code enforces 80)
  - Build output:
    - `json.images` (3 temp URLs)
    - `json.prompt`
    - `json.metadata` with `taskId`, `productId`, `rowNumber`, `productName`
- **Outputs:** One item for the downstream image generation request.
- **Failure types:** Missing upload URLs (due to tmpfiles failure), missing sheet fields, agent output missing, word-splitting edge cases.

#### 2.9 NanoBanana Pro Edit
- **Type / role:** HTTP Request to AtlasCloud; generates edited selfie image using 3 references + prompt.
- **Config choices:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
  - Bearer auth (credential: **AtlasCloud**)
  - JSON body key settings:
    - `model`: `google/nano-banana-pro/edit`
    - `aspect_ratio`: `9:16`
    - `resolution`: `2k`
    - `enable_sync_mode`: true
    - `enable_base64_output`: false
    - `output_format`: `png`
    - `images`: `{{ $json.images.toJsonString() }}`
    - `prompt`: sanitized: remove line breaks and quotes
- **Important edge case:** The JSON body prompt expression appears to include an extra `}} }}` at the end:
  - `"... replace(/[“”]/g, '') }} }}"`  
  This can create malformed JSON or unintended prompt text. Same pattern also appears in the Veo node.
- **Outputs:** AtlasCloud response with `code`, `data.outputs` (image URL list).
- **Failure types:** Auth invalid, request schema mismatch, model errors, content moderation rejection, timeout.

#### 2.10 If Image Success
- **Type / role:** If node; routes based on AtlasCloud response code.
- **Condition:** `{{ $json.code }} == 200`
- **True path:** `Analyze image`
- **False path:** `Update Edit Task [ERROR]`
- **Edge cases:** If `code` missing/non-string, loose validation may pass unexpectedly.

#### 2.11 Analyze image
- **Type / role:** OpenAI (LangChain OpenAI) “image analyze” operation; describes the generated image.
- **Config:**
  - Resource: `image`, Operation: `analyze`
  - Model: `gpt-4o-mini`
  - `imageUrls`: `{{ $json.data.outputs[0] }}`
  - Text prompt: requests a concise paragraph describing subject action/expression, environment, object held (including readable text quoted), and composition.
- **Outputs:** Typically structured content; later referenced as `$json[0].content[0].text` in the Sheets update.
- **Failure types:** OpenAI credential issues, invalid image URL, safety filters, response shape differences (breaking downstream expression).

#### 2.12 Update Edit Task [SUCCESS]
- **Type / role:** Google Sheets update; marks the task edited and stores outputs.
- **Matching:** Updates row where `Task ID` matches `$('Get Clean Data').item.json.metadata.taskId`
- **Writes:**
  - `Status` = `Edited`
  - `Image Prompt` = prompt from `Get Clean Data`
  - `Image Result` = `$('If Image Success').item.json.data.outputs.first()`
  - `Analyze Image` = `{{ $json[0].content[0].text }}`
- **Edge cases:**
  - If `Analyze image` output format differs, `Analyze Image` write will fail expression evaluation.
  - If multiple tasks are processed per run, using `.item`/`.first()` can mismatch prompts to rows unless item linking is correct.

#### 2.13 Update Edit Task [ERROR]
- **Type / role:** Google Sheets update; marks task as error if image generation failed.
- **Matching:** `Task ID` from `Get Clean Data` metadata.
- **Writes:** `Status` = `Error`
- **Edge cases:** If failure occurs *before* `Get Clean Data` (e.g., tmpfiles upload fails and `Get Clean Data` throws), this node may never be reached because the error happens upstream.

---

### Block B — Make Videos: from “Edited” tasks to “Finished” tasks
**Overview:** Runs on a schedule, fetches all rows with `Status = Edited`, generates a Veo 3.1 prompt, submits an image-to-video job to AtlasCloud, polls until completion, then updates Google Sheets.

**Nodes involved:**
- Schedule Trigger Make Videos
- Get Edited Task
- Video Prompt Agent
- Veo 3.1
- Wait
- Get a Video
- Switch
- Update Video Task [SUCCESS]
- Update Video Task [ERROR]

#### 2.14 Schedule Trigger Make Videos
- **Type / role:** Schedule Trigger; initiates the video pipeline.
- **Config:** Every hour.
- **Outputs:** Triggers `Get Edited Task`.

#### 2.15 Get Edited Task
- **Type / role:** Google Sheets read; pulls tasks ready for video generation.
- **Filter:** `Status` equals **Edited**
- **Outputs:** Each row becomes an item passed to `Video Prompt Agent`.
- **Edge cases:** Same as other Sheets reads (auth, quotas, wrong spreadsheet URL, missing columns).

#### 2.16 Video Prompt Agent
- **Type / role:** LangChain Agent; creates a single Veo-style prompt.
- **Inputs:** Sheet row fields:
  - Product Name, Video Scene, Target Market, Product Description, Analyze Image
- **System message highlights:**
  - Output in English only
  - Dialogue must be in **Indonesian by default**, unless target market specifies otherwise
  - Subject must be an adult (20+), avoid sensitive descriptions
  - 9:16 handheld selfie, 8 seconds, no overlays/watermarks, do not mention reference image
  - Exactly 1 prompt block, dialogue in quotes
- **Model routing:** same as Image Prompt Agent (primary GPT-5-mini, fallback Groq).
- **Outputs:** `output` used by Veo node.
- **Edge cases:** Missing Analyze Image text (if previous block didn’t write), refusal/safety behavior, overly long prompt.

#### 2.17 Veo 3.1
- **Type / role:** HTTP Request to AtlasCloud; starts video generation job.
- **Config:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateVideo`
  - Bearer auth AtlasCloud
  - JSON body:
    - `model`: `google/veo3.1/image-to-video`
    - `aspect_ratio`: `9:16`
    - `duration`: 8
    - `generate_audio`: true
    - `image`: from `Get Edited Task` → `Image Result`
    - `prompt`: from agent output, sanitized (line breaks and quotes removed)
    - `resolution`: `1080p`
    - `seed`: 1
- **Important edge case:** The same suspicious trailing `}} }}` appears in the prompt string which can break JSON.
- **Outputs:** Job object containing `data.id` for polling.
- **Failure types:** Auth errors, invalid image URL, moderation/safety rejection, async job start failures.

#### 2.18 Wait
- **Type / role:** Wait node; delays before polling.
- **Config:** Waits `amount: 10` (seconds by default).
- **Outputs:** Then calls `Get a Video`.
- **Edge cases:** Polling interval might be too short/long depending on typical job durations and AtlasCloud rate limits.

#### 2.19 Get a Video
- **Type / role:** HTTP Request; polls AtlasCloud prediction endpoint.
- **URL:** `https://api.atlascloud.ai/api/v1/model/prediction/{{ $json.data.id }}`
- **Auth:** Bearer AtlasCloud
- **Outputs:** Includes `data.status` and if completed, likely `data.outputs`.
- **Failure types:** 404 (bad id), 429 rate limit, transient 5xx.

#### 2.20 Switch
- **Type / role:** Switch on job status; routes polling loop.
- **Rules (renamed outputs):**
  - `completed` if `data.status == completed` → `Update Video Task [SUCCESS]`
  - `failed` if `data.status == failed` → `Update Video Task [ERROR]`
  - `processing` if `data.status == processing` → `Wait` (loop)
- **Edge cases:** Other statuses (e.g., queued) are not handled; would fall through and produce no output (silent stall).

#### 2.21 Update Video Task [SUCCESS]
- **Type / role:** Google Sheets update; writes final video URL and marks complete.
- **Matching:** `Task ID` from `Get Edited Task`
- **Writes:**
  - `Status` = `Finished`
  - `Video Prompt` = `$('Video Prompt Agent').item.json.output`
  - `Video Result` = `$('Get a Video').item.json.data.outputs.first()`
- **Edge cases:** If `outputs` missing or not an array, `.first()` may fail.

#### 2.22 Update Video Task [ERROR]
- **Type / role:** Google Sheets update; records failure.
- **Matching:** `Task ID` from `Get Edited Task`
- **Writes:**
  - `Status` = `Error`
  - `Video Prompt` = agent output
  - `Error Message` = `{{ $json.data.error }}`
- **Edge cases:** If API error shape differs (no `data.error`), message cell becomes blank or expression errors.

---

### Block C — AI Models and Notes (non-execution helpers)
**Overview:** Two LLM provider nodes supply the agents. Sticky notes provide setup instructions and links.

**Nodes involved:**
- GPT-5-Mini
- GPT-OSS-120b
- Sticky Note, Sticky Note5, Sticky Note1, Sticky Note2

#### 2.23 GPT-5-Mini
- **Type / role:** LangChain ChatOpenAI model node; primary LLM for both agents.
- **Model:** `openai/gpt-5-mini`
- **Connections:** Supplies `ai_languageModel` to both `Image Prompt Agent` and `Video Prompt Agent`.
- **Failure types:** OpenAI credential invalid, model unavailable in account/region.

#### 2.24 GPT-OSS-120b
- **Type / role:** LangChain Groq model node; fallback LLM for both agents.
- **Model:** `openai/gpt-oss-120b` (served via Groq)
- **Connections:** Secondary `ai_languageModel` input for both agents.
- **Failure types:** Groq credential invalid, model name mismatch, rate limits.

#### 2.25 Sticky Notes (documentation only)
- **Sticky Note1 content:** “## Combine 3 Images into 1 (Product, Character, Background)”
- **Sticky Note2 content:** “## Make Video from Images”
- **Sticky Note (Quick Setup Guide):** Creator info, setup steps, links (Google Cloud Console, template copy, API providers, Atlas Cloud link), test procedure, what it does.
- **Sticky Note5 (Follow channels + video link):** Social links and YouTube video link/thumbnail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger Edit Images | scheduleTrigger | Triggers image edit pipeline hourly | — | Get Ready Task | ## Combine 3 Images into 1 (Product, Character, Background) |
| Get Ready Task | googleSheets | Fetch tasks with Status=Ready | Schedule Trigger Edit Images | Image Prompt Agent | ## Combine 3 Images into 1 (Product, Character, Background) |
| GPT-5-Mini | lmChatOpenAi | Primary LLM for both agents | — | Image Prompt Agent; Video Prompt Agent |  |
| GPT-OSS-120b | lmChatGroq | Fallback LLM for both agents | — | Image Prompt Agent; Video Prompt Agent |  |
| Image Prompt Agent | langchain.agent | Generate concise selfie image prompt | Get Ready Task; GPT-5-Mini/GPT-OSS-120b | Get Image Array | ## Combine 3 Images into 1 (Product, Character, Background) |
| Get Image Array | code | Build array of 3 image URLs per task | Image Prompt Agent; (also Get Temp Url connection) | Split Out | ## Combine 3 Images into 1 (Product, Character, Background) |
| Split Out | splitOut | Split `images[]` into separate items | Get Image Array | Download Images | ## Combine 3 Images into 1 (Product, Character, Background) |
| Download Images | httpRequest | Download each image URL to binary | Split Out | Get Temp Url | ## Combine 3 Images into 1 (Product, Character, Background) |
| Get Temp Url | httpRequest | Upload image binary to tmpfiles.org | Download Images | Get Clean Data; Get Image Array | ## Combine 3 Images into 1 (Product, Character, Background) |
| Get Clean Data | code | Validate 3 URLs, normalize, attach prompt + metadata | Get Temp Url | NanoBanana Pro Edit | ## Combine 3 Images into 1 (Product, Character, Background) |
| NanoBanana Pro Edit | httpRequest | AtlasCloud image edit (NanoBanana Pro Edit) | Get Clean Data | If Image Success | ## Combine 3 Images into 1 (Product, Character, Background) |
| If Image Success | if | Route based on response code==200 | NanoBanana Pro Edit | Analyze image; Update Edit Task [ERROR] | ## Combine 3 Images into 1 (Product, Character, Background) |
| Analyze image | langchain.openAi (image analyze) | Describe generated image (vision) | If Image Success | Update Edit Task [SUCCESS] | ## Combine 3 Images into 1 (Product, Character, Background) |
| Update Edit Task [SUCCESS] | googleSheets | Write Edited status + image prompt/result + analysis | Analyze image | — | ## Combine 3 Images into 1 (Product, Character, Background) |
| Update Edit Task [ERROR] | googleSheets | Write Error status for edit stage | If Image Success (false) | — | ## Combine 3 Images into 1 (Product, Character, Background) |
| Schedule Trigger Make Videos | scheduleTrigger | Triggers video generation pipeline hourly | — | Get Edited Task | ## Make Video from Images |
| Get Edited Task | googleSheets | Fetch tasks with Status=Edited | Schedule Trigger Make Videos | Video Prompt Agent | ## Make Video from Images |
| Video Prompt Agent | langchain.agent | Generate Veo-style UGC video prompt | Get Edited Task; GPT-5-Mini/GPT-OSS-120b | Veo 3.1 | ## Make Video from Images |
| Veo 3.1 | httpRequest | Start AtlasCloud Veo 3.1 image-to-video job | Video Prompt Agent | Wait | ## Make Video from Images |
| Wait | wait | Delay before polling | Veo 3.1; Switch (processing) | Get a Video | ## Make Video from Images |
| Get a Video | httpRequest | Poll AtlasCloud prediction status/results | Wait | Switch | ## Make Video from Images |
| Switch | switch | Route based on job status (completed/failed/processing) | Get a Video | Update Video Task [SUCCESS]; Update Video Task [ERROR]; Wait | ## Make Video from Images |
| Update Video Task [SUCCESS] | googleSheets | Write Finished status + video prompt/result | Switch (completed) | — | ## Make Video from Images |
| Update Video Task [ERROR] | googleSheets | Write Error status + error message | Switch (failed) | — | ## Make Video from Images |
| Sticky Note1 | stickyNote | Comment: combine 3 images | — | — | ## Combine 3 Images into 1 (Product, Character, Background) |
| Sticky Note2 | stickyNote | Comment: make video from images | — | — | ## Make Video from Images |
| Sticky Note | stickyNote | Setup guide & links | — | — | ## 🛠️ Quick Setup Guide  / **Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre) / Google Cloud Console link / Template copy link / Provider links / Atlas Cloud link / Test steps / Purpose |
| Sticky Note5 | stickyNote | Social + YouTube video link | — | — | YouTube/Instagram/TikTok links + https://youtu.be/wOdtH54A8iE |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Google Sheet**
   1) Duplicate the provided template (or create your own with matching columns).  
   2) Ensure a tab/sheet named **Task** includes at least:  
      `Task ID`, `Status`, `Product Photo`, `Character`, `Background`, `Product Name`, `Video Scene`, `Product Description`, `Target Market`, `Image Prompt`, `Image Result`, `Analyze Image`, `Video Prompt`, `Video Result`, `Error Message`.

2. **Create credentials**
   1) **Google Sheets OAuth2** in n8n (Google Cloud Console OAuth client, add redirect URL from n8n).  
   2) **HTTP Bearer Auth** credential named (e.g.) `AtlasCloud` containing your AtlasCloud API key.  
   3) **OpenAI API** credential for:
      - GPT-5-mini node (ChatOpenAI)
      - Analyze image node (gpt-4o-mini vision)
   4) (Optional fallback) **Groq API** credential for the Groq model node.

3. **Add nodes for Image Editing pipeline**
   1) Add **Schedule Trigger** named `Schedule Trigger Edit Images`, set interval to **every hour**.  
   2) Add **Google Sheets** node `Get Ready Task`:
      - Operation: **Read/Get Many** (as available)
      - Filter: `Status` equals `Ready`
      - Select your spreadsheet + `Task` sheet
   3) Add **LM model** node `GPT-5-Mini` (LangChain ChatOpenAI):
      - Model: `openai/gpt-5-mini`
      - Attach OpenAI credentials
   4) Add **LM model** node `GPT-OSS-120b` (LangChain Groq):
      - Model: `openai/gpt-oss-120b`
      - Attach Groq credentials
   5) Add **Agent** node `Image Prompt Agent`:
      - Text: `Product: {{$json["Product Name"]}}\nScene: {{$json["Video Scene"]}}`
      - System message: the UGC photo rules (English only, short, holding product naturally, etc.)
      - Connect `GPT-5-Mini` to agent’s **AI Language Model** input (index 0)
      - Connect `GPT-OSS-120b` to agent’s **AI Language Model** input (index 1) as fallback
   6) Add **Code** node `Get Image Array` to output `images: [Product Photo, Character, Background]` for each task.
   7) Add **Split Out** node `Split Out` splitting field `images`.
   8) Add **HTTP Request** node `Download Images`:
      - URL: `={{ $json.images }}`
      - Ensure it downloads as binary (n8n defaults typically attach binary)
   9) Add **HTTP Request** node `Get Temp Url`:
      - POST `https://tmpfiles.org/api/v1/upload`
      - Send multipart form-data with `file` from binary property (commonly `data`)
      - (Optional) Set **On Error** to “Continue” if you want partial outputs; otherwise prefer failing fast.
   10) Add **Code** node `Get Clean Data`:
       - Collect 3 tmpfiles URLs, convert to direct `/dl/` links, force https
       - Pull prompt from `Image Prompt Agent`
       - Build `{ images: [url1,url2,url3], prompt, metadata: { taskId,... } }`
   11) Add **HTTP Request** node `NanoBanana Pro Edit`:
       - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
       - Auth: Bearer (AtlasCloud)
       - JSON body containing: model `google/nano-banana-pro/edit`, aspect_ratio `9:16`, resolution `2k`, images array, prompt string
       - **Important:** Ensure your JSON body is valid and does not include stray braces after the prompt expression.
   12) Add **IF** node `If Image Success`:
       - Condition: `{{$json.code}} equals 200`
   13) Add **OpenAI Image Analyze** node `Analyze image`:
       - Model: `gpt-4o-mini`
       - Image URL: `={{ $json.data.outputs[0] }}`
       - Prompt text as in the workflow (describe subject, environment, object, composition)
   14) Add **Google Sheets Update** node `Update Edit Task [SUCCESS]`:
       - Match on `Task ID`
       - Set `Status=Edited`, `Image Prompt`, `Image Result`, `Analyze Image`
   15) Add **Google Sheets Update** node `Update Edit Task [ERROR]`:
       - Match on `Task ID`
       - Set `Status=Error`

4. **Add nodes for Video Generation pipeline**
   1) Add **Schedule Trigger** named `Schedule Trigger Make Videos` (every hour).
   2) Add **Google Sheets** node `Get Edited Task` filtering `Status=Edited`.
   3) Add **Agent** node `Video Prompt Agent`:
      - Text includes `Product Name`, `Video Scene`, `Target Market`, `Product Description`, `Analyze Image`
      - System message: Veo prompt rules (English output, Indonesian dialogue default, adult subject, 9:16 selfie, 8 seconds, no overlays)
      - Attach the same LLM model inputs as above (GPT-5-mini primary, Groq fallback)
   4) Add **HTTP Request** node `Veo 3.1`:
      - POST `https://api.atlascloud.ai/api/v1/model/generateVideo`
      - JSON: model `google/veo3.1/image-to-video`, aspect_ratio `9:16`, duration `8`, generate_audio `true`, image from `Image Result`, prompt from agent output, resolution `1080p`, seed `1`
      - **Important:** keep JSON valid; avoid stray braces in prompt field.
   5) Add **Wait** node `Wait` set to ~10 seconds.
   6) Add **HTTP Request** node `Get a Video`:
      - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{ $json.data.id }}`
      - Bearer auth AtlasCloud
   7) Add **Switch** node `Switch` on `{{$json.data.status}}`:
      - `completed` → success update
      - `failed` → error update
      - `processing` → loop back to `Wait`
      - (Recommended) add a default output for unknown statuses (e.g., queued) to loop as well.
   8) Add **Google Sheets Update** `Update Video Task [SUCCESS]`:
      - Match by `Task ID`
      - Set `Status=Finished`, `Video Prompt`, `Video Result`
   9) Add **Google Sheets Update** `Update Video Task [ERROR]`:
      - Match by `Task ID`
      - Set `Status=Error`, `Video Prompt`, `Error Message`

5. **Wire connections in order**
   - Edit pipeline: Trigger → Get Ready Task → Image Prompt Agent → Get Image Array → Split Out → Download Images → Get Temp Url → Get Clean Data → NanoBanana → If → (true) Analyze → Update SUCCESS; (false) Update ERROR
   - Video pipeline: Trigger → Get Edited Task → Video Prompt Agent → Veo → Wait → Get a Video → Switch → (completed) Update SUCCESS; (failed) Update ERROR; (processing) Wait

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Created by Kristian Ekachandra | https://yapp.ink/aichandre |
| Google Sheets OAuth2 setup via Google Cloud Console | https://console.cloud.google.com/ |
| Duplicate the Google Sheets template and update all node URLs | https://docs.google.com/spreadsheets/d/1wVz-tvvAuYi9sHtHj40i4yeOFZMbdwmuaQR5N5wRg08/copy |
| OpenAI platform | https://platform.openai.com |
| Groq | https://groq.com |
| Gemini API key (alternative) | https://aistudio.google.com/app/apikey |
| Atlas Cloud (API key used in NanoBanana + Veo nodes) | https://goto.atlascloud.ai/Kristian?ref=TM2L4K |
| Full video link | https://youtu.be/wOdtH54A8iE |
| Socials: YouTube/Instagram/TikTok @aichandre | https://www.youtube.com/@aichandre / https://www.instagram.com/aichandre / https://www.tiktok.com/@aichandre |

**Disclaimer (provided):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… (content is legal/public and compliant).