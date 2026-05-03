Generate UGC-style video ads of any length with Kie, Claude, GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/generate-ugc-style-video-ads-of-any-length-with-kie--claude--gpt-4o-and-google-sheets-14316


# Generate UGC-style video ads of any length with Kie, Claude, GPT-4o and Google Sheets

# 1. Workflow Overview

This workflow automates the end-to-end production of UGC-style AI video ads from rows stored in Google Sheets. It reads pending video ideas, generates a reference image, analyzes that image, turns the script into multiple 8-second scenes, generates one video clip per scene, uploads assets to Google Drive, stitches the clips into a final video, and writes progress and output links back to Google Sheets.

It is designed for batch or scheduled production of ad creatives where each input row contains:
- a script,
- a character/setting description,
- an aspect ratio,
- and a creation status flag.

The workflow has **three entry points**:
1. **Schedule Trigger1**: main intake pipeline, every 30 minutes.
2. **When Executed by Another Workflow**: alternate programmatic entry into the same intake pipeline.
3. **Schedule Trigger**: regeneration/retry pipeline for scene clips marked `Redo` in the scene-tracking sheet.

## 1.1 Intake and Job Selection
The workflow reads rows from the **Videos** sheet where `LAUNCH CREATION = Create`, normalizes fields, generates a unique run ID, and immediately marks the source row as `Processing`.

## 1.2 Reference Image Prompting and Generation
It uses a LangChain agent with OpenRouter Claude to build a structured image prompt, then submits an image-generation task to Kie.

## 1.3 Image Polling, Download, and Sharing
The workflow polls Kie until the reference image task succeeds or fails. On success, it downloads the image, uploads it to Google Drive, shares it publicly, and exposes both a view link and downloadable link.

## 1.4 Scene Planning from Script + Visual Reference
It analyzes the generated reference image with OpenAI GPT-4o-mini, then uses Claude Opus to split the script into ordered 8-second scenes with a fixed visual description and per-scene movement instructions.

## 1.5 Scene Row Creation and Source Row Update
Each generated scene is appended into the **Video Data** sheet, with identifiers such as `ID`, `VIDEO ID`, `SCENE NO`, `TOTAL SCENES`, serialized scene JSON, and the shared image URL. The source **Videos** row is also updated with the generated run ID and processing status.

## 1.6 Scene Video Generation and Per-Clip Storage
For each scene row in **Video Data**, the workflow submits a VEO3 video generation request to Kie, polls until completion, downloads the resulting clip, uploads it to Google Drive, shares it, and updates that scene row with clip links or failure status.

## 1.7 Completion Check and Final Stitching
After a clip completes, the workflow checks whether all scenes for the same parent `ID` are completed. If yes, it sorts the clips by `SCENE NO`, aggregates their URLs, merges them with fal.ai FFmpeg, downloads the merged video, uploads it to Drive, shares it, and updates the source **Videos** row to `Completed`.

## 1.8 Regeneration / Retry Path
A separate scheduled path runs every 15 minutes to find rows in **Video Data** marked `Redo`, groups them by `ID`, prefers the least-complete set, and re-runs scene video generation for those rows.

---

# 2. Block-by-Block Analysis

## Block 1 — Intake and Source Row Preparation

### Overview
This block selects source requests from the **Videos** sheet and prepares the normalized payload used throughout the rest of the workflow. It also updates the source row so operators can see that generation has started.

### Nodes Involved
- Schedule Trigger1
- When Executed by Another Workflow
- Get Video Ideas
- UpdateSheet6
- Input

### Node Details

#### Schedule Trigger1
- **Type / role:** `n8n-nodes-base.scheduleTrigger`; periodic entry point for the main creation pipeline.
- **Configuration:** Runs every 30 minutes.
- **Input / output:** No input; outputs to **Get Video Ideas**.
- **Version-specific notes:** Type version `1.3`.
- **Failure/edge cases:** Timezone behavior depends on workflow instance settings.

#### When Executed by Another Workflow
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger`; alternate entry point for parent workflows.
- **Configuration:** `inputSource = passthrough`, so inbound data can be forwarded.
- **Input / output:** Trigger node; outputs to **Get Video Ideas**.
- **Sub-workflow reference:** This workflow can be invoked by another workflow, but it does not itself call a sub-workflow.
- **Failure/edge cases:** Upstream workflow must pass compatible data if relying on passthrough behavior.

#### Get Video Ideas
- **Type / role:** Google Sheets node; fetches a source row from **Videos**.
- **Configuration choices:**
  - Document: `Tax Relief - Final AI Videos (n8n)`
  - Sheet/tab: `Videos`
  - Filter: `LAUNCH CREATION = Create`
  - `returnFirstMatch = true`
  - Retries enabled with 5-second wait
- **Key variables:** none beyond sheet filter.
- **Input / output connections:** Triggered by **Schedule Trigger1** and **When Executed by Another Workflow**; outputs to **Input** and **UpdateSheet6**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - Missing or renamed sheet/column headers
  - No matching rows found
  - Google auth/permission errors
  - `returnFirstMatch` means only one eligible row is processed per run

#### UpdateSheet6
- **Type / role:** Google Sheets update; marks the source row as processing immediately.
- **Configuration choices:**
  - Target sheet: `Videos`
  - Match column: `row_number`
  - Writes:
    - `row_number = {{$json.row_number}}`
    - `LAUNCH CREATION = Processing`
  - Executes once
- **Input / output connections:** Input from **Get Video Ideas**; no downstream connection.
- **Key variables:** `{{$json.row_number}}`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - `row_number` must exist from Google Sheets output
  - Failure here does not stop the main branch because the data also goes to **Input**

#### Input
- **Type / role:** Set node; normalizes source sheet columns into internal field names.
- **Configuration choices:** Creates:
  - `character_description = {{$json['CHARACTER/SETTING DESCRIPTION']}}`
  - `demo_script = {{$json.SCRIPT}}`
  - `aspect_ratio = {{$json['ASPECT RATIO'] || "9:16"}}`
  - `id = {{$execution.id + '_' + Date.now().toString() + Math.random().toString(36).substring(2, 10)}}`
  - `row_number = {{$json.row_number}}`
- **Input / output connections:** Input from **Get Video Ideas**; output to **Nano Banana Prompt Agent**.
- **Key variables:** `$execution.id`, `Date.now()`, random suffix generation.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failure types:**
  - Missing source fields create empty strings
  - Generated `id` is probabilistically unique, not globally guaranteed
  - Default aspect ratio is `9:16`

---

## Block 2 — Image Prompt Construction and Submission

### Overview
This block transforms the human description and script into a structured image-generation prompt, then submits a Kie image-generation task.

### Nodes Involved
- Prompt Agent LLM
- Prompt Output Parser
- Nano Banana Prompt Agent
- Generate UGC Image
- Image Gen Response Check

### Node Details

#### Prompt Agent LLM
- **Type / role:** OpenRouter chat model; language model backend for the agent.
- **Configuration choices:** Model `anthropic/claude-opus-4.5`.
- **Input / output connections:** Feeds **Nano Banana Prompt Agent** via `ai_languageModel`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failure types:**
  - OpenRouter credential or quota issues
  - Model availability changes

#### Prompt Output Parser
- **Type / role:** Structured output parser for the prompt agent.
- **Configuration choices:** Manual JSON schema requiring:
  - `prompt`
  - `model`
  - `aspectRatio`
  - `resolution`
- **Input / output connections:** Feeds **Nano Banana Prompt Agent** via `ai_outputParser`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failure types:**
  - LLM output not matching schema
  - Missing required fields

#### Nano Banana Prompt Agent
- **Type / role:** LangChain agent; generates a clean prompt package for image creation.
- **Configuration choices:**
  - Consumes `character_description`, `demo_script`, `aspect_ratio`
  - Strong system prompt enforcing photorealistic iPhone-style selfie framing, blurred background, realistic skin texture, non-studio lighting, and no visible phone
  - `hasOutputParser = true`
- **Key expressions:**
  - `{{ $('Input').item.json.character_description }}`
  - `{{ $('Input').item.json.demo_script }}`
  - `{{ $json.aspect_ratio }}`
- **Input / output connections:** Main input from **Input**; LLM input from **Prompt Agent LLM**; parser input from **Prompt Output Parser**; outputs to **Generate UGC Image**.
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases/failure types:**
  - Model may return malformed JSON despite parser
  - Long descriptions may degrade prompt quality
  - Prompt text says “Nano Banana Pro API JSON body” while downstream API actually uses Kie with model `nano-banana-2`

#### Generate UGC Image
- **Type / role:** HTTP Request; submits image generation task to Kie.
- **Configuration choices:**
  - POST `https://api.kie.ai/api/v1/jobs/createTask`
  - Uses generic header auth
  - JSON body includes:
    - `model: "nano-banana-2"`
    - placeholder callback URL
    - prompt from `{{$json.output.prompt}}`
    - aspect ratio from `Input`
    - resolution `2K`
    - output format `png`
- **Key expressions:**
  - `{{ JSON.stringify($json.output.prompt) }}`
  - `{{ $('Input').item.json.aspect_ratio }}`
- **Input / output connections:** Input from **Nano Banana Prompt Agent**; output to **Image Gen Response Check**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases/failure types:**
  - Invalid auth header
  - Placeholder callback URL may be ignored by API or may be invalid operationally
  - If prompt is missing, API may reject the request
  - Retry enabled

#### Image Gen Response Check
- **Type / role:** Switch; validates immediate Kie submission response.
- **Configuration choices:**
  - `Success` when `code == 200`
  - `Error` when `code != 200`
  - fallback `Unexpected`
- **Input / output connections:**
  - Success → **Poll Image Status**
  - Error / Unexpected → **UpdateSheet5**
- **Key variables:** `{{$json.code}}`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failure types:**
  - APIs that return non-200 success semantics would be misclassified
  - Unexpected response shape causes expression issues

---

## Block 3 — Image Polling, Storage, and Failure Handling

### Overview
This block monitors the asynchronous Kie image task, loops until completion, and either stores the result in Drive or marks the source row as failed.

### Nodes Involved
- Poll Image Status
- Image Status Check
- Wait 10s
- Download Image
- Upload file
- Share file
- Workflow Configuration3
- UpdateSheet5
- Image Poll Error

### Node Details

#### Poll Image Status
- **Type / role:** HTTP Request; polls Kie image task status.
- **Configuration choices:**
  - GET `https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json.data.taskId }}`
  - Generic header auth
- **Input / output connections:** Input from **Image Gen Response Check** and loop from **Wait 10s**; output to **Image Status Check**.
- **Key expressions:** `{{$json.data.taskId}}`
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases/failure types:**
  - Missing `taskId`
  - Rate limits during repeated polling
  - Auth/network failures

#### Image Status Check
- **Type / role:** Switch; branches on image task state.
- **Configuration choices:**
  - `Success` if `successFlag == 1` or `state == success`
  - `Failed` if `successFlag >= 2` or `state == failed`
  - `Processing` if `successFlag == 0` or state in `waiting|processing|generating`
  - fallback `Unknown Status`
- **Input / output connections:**
  - Success → **Download Image**
  - Failed → **UpdateSheet5**
  - Processing → **Wait 10s**
  - Unknown → **UpdateSheet5**
- **Key variables:** `$json.data.successFlag`, `$json.data.state`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failure types:**
  - Unknown state values are treated as failure path
  - If `data` is missing, expressions may fail

#### Wait 10s
- **Type / role:** Wait; poll delay.
- **Configuration choices:** `amount = 60` with no unit explicitly shown in the export; in n8n this generally means default wait-unit behavior, so it should be verified in UI.
- **Input / output connections:** From **Image Status Check** processing branch; back to **Poll Image Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases/failure types:**
  - Ambiguity over wait unit from export; verify manually
  - Long-running waits can accumulate executions

#### Download Image
- **Type / role:** HTTP Request; downloads the finished generated image.
- **Configuration choices:** Downloads from `{{$json.data.resultJson.parseJson().resultUrls[0]}}`
- **Input / output connections:** From **Image Status Check** success branch; output to **Upload file**.
- **Key expressions:** `.parseJson().resultUrls[0]`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases/failure types:**
  - `resultJson` must be valid JSON string
  - Missing `resultUrls[0]`
  - Binary download settings are implicit and should be verified in UI if rebuilding

#### Upload file
- **Type / role:** Google Drive upload; stores the reference image.
- **Configuration choices:**
  - File name: `{{$('Input').item.json.id}}.png`
  - Folder ID: `11kfsPSUmo3vzh5J3evGxEFa6aRrNPh5R`
  - Drive: `My Drive`
- **Input / output connections:** From **Download Image**; output to **Share file**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failure types:**
  - Binary property must exist
  - Folder permissions or invalid folder ID
  - Google Drive quota issues

#### Share file
- **Type / role:** Google Drive share; makes the image publicly readable.
- **Configuration choices:**
  - Operation `share`
  - `role = reader`
  - `type = anyone`
  - File ID from upload result
- **Input / output connections:** From **Upload file**; output to **Workflow Configuration3**.
- **Key expressions:** `{{$json.id}}`
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failure types:**
  - Share restrictions in workspace policies
  - Missing `id` from upload step

#### Workflow Configuration3
- **Type / role:** Set; packages downstream fields for analysis and scene generation.
- **Configuration choices:** Creates:
  - `referenceImage = {{$('Upload file').item.json.webViewLink}}`
  - `download_image = {{$('Upload file').item.json.webContentLink}}`
  - `character_description`, `demo_script`, `aspect_ratio` from **Input**
- **Input / output connections:** From **Share file**; output to **Analyze UGC Image3**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failure types:**
  - Depends on upload output rather than share output, which is valid but slightly fragile if node behavior changes

#### UpdateSheet5
- **Type / role:** Google Sheets update; marks source video request as failed.
- **Configuration choices:**
  - Match on `ID`
  - Writes `LAUNCH CREATION = Failed`
  - `ID = {{$('Input').first().json.id}}`
- **Input / output connections:** From **Image Gen Response Check** error/unexpected and **Image Status Check** failed/unknown; output to **Image Poll Error**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - If **UpdateSheet1** has not yet written the `ID` back to the source row, matching by ID may fail
  - This is a sequencing risk in the design

#### Image Poll Error
- **Type / role:** Stop and Error; terminates execution with a detailed image-generation error.
- **Configuration choices:** Error message includes state, success flag, error message, and task ID.
- **Input / output connections:** From **UpdateSheet5**; terminal node.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failure types:** Safe final failure node; depends on expected Kie response shape.

---

## Block 4 — Visual Analysis and Scene Planning

### Overview
This block derives a visual consistency description from the generated image and converts the original script into a structured scene list for downstream video generation.

### Nodes Involved
- Analyze UGC Image3
- Script Agent LLM
- Scene Output Parser
- Generate Scene Scripts3
- Split Scenes

### Node Details

#### Analyze UGC Image3
- **Type / role:** OpenAI image analysis node; extracts subject and setting details from the reference image.
- **Configuration choices:**
  - Resource: `image`
  - Operation: `analyze`
  - Model: `gpt-4o-mini`
  - Detail: `high`
  - Image URL from `download_image`
- **Input / output connections:** From **Workflow Configuration3**; output to **Generate Scene Scripts3**.
- **Key expressions:** `{{$json.download_image}}`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases/failure types:**
  - OpenAI auth or quota failures
  - Public image URL must be accessible
  - Response structure is assumed later as `$json['0'].content[0].text`

#### Script Agent LLM
- **Type / role:** OpenRouter chat model; LLM backend for scene generation.
- **Configuration choices:** Model `anthropic/claude-opus-4.5`.
- **Input / output connections:** Feeds **Generate Scene Scripts3** via `ai_languageModel`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failure types:** Same OpenRouter issues as above.

#### Scene Output Parser
- **Type / role:** Structured parser; enforces schema for `scenes[]`.
- **Configuration choices:** Manual JSON schema requiring:
  - `scene_number`
  - `script`
  - `visual_description`
  - `movement`
  - `duration_seconds`
  - optional `audio_description`
- **Input / output connections:** Feeds **Generate Scene Scripts3** via `ai_outputParser`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failure types:**
  - Scene schema mismatch
  - LLM may produce invalid JSON or omit required keys

#### Generate Scene Scripts3
- **Type / role:** LangChain agent; creates structured scenes from script plus image analysis.
- **Configuration choices:**
  - Input text combines:
    - character description
    - generated image analysis
    - original user script
  - System prompt enforces:
    - identical `visual_description` across all scenes
    - roughly 20–27 words / ~8 seconds per scene
    - `duration_seconds = 8`
    - movement logic based on scene setting
    - conversational script style
  - `hasOutputParser = true`
- **Key expressions:**
  - `{{ $('Workflow Configuration3').item.json.character_description }}`
  - `{{ $json['0'].content[0].text }}`
  - `{{ $('Workflow Configuration3').item.json.demo_script }}`
- **Input / output connections:** Main data from **Analyze UGC Image3**; LLM from **Script Agent LLM**; parser from **Scene Output Parser**; output to **Split Scenes**.
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases/failure types:**
  - Strong dependency on exact OpenAI response structure
  - Very short scripts may yield one scene; long scripts may yield many scenes
  - Prompt says “No background noice” in sample JSON, typo not functionally harmful

#### Split Scenes
- **Type / role:** Split Out; turns the scenes array into one item per scene.
- **Configuration choices:** `fieldToSplitOut = output.scenes`
- **Input / output connections:** From **Generate Scene Scripts3**; output to **UpdateSheet**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failure types:**
  - If parser output is not nested under `output.scenes`, split fails
  - Empty scene array produces no items

---

## Block 5 — Scene Tracking Rows and Source Row ID Update

### Overview
This block writes each scene into the **Video Data** sheet and updates the source **Videos** row with the generated run ID and status.

### Nodes Involved
- UpdateSheet
- Wait
- UpdateSheet1

### Node Details

#### UpdateSheet
- **Type / role:** Google Sheets append; stores per-scene generation metadata in **Video Data**.
- **Configuration choices:**
  - Appends columns:
    - `ID = {{$('Input').first().json.id}}`
    - `LAUNCH = Processing`
    - `SCRIPT = {{JSON.stringify($json)}}`
    - `SCENE NO = {{$json.scene_number}}`
    - `VIDEO ID = {{$('Input').first().json.id}}_{{$itemIndex}}`
    - `IMAGE URL = {{$('Workflow Configuration3').first().json.download_image}}`
    - `TOTAL SCENES = {{$('Generate Scene Scripts3').first().json.output.scenes.length}}`
- **Input / output connections:** From **Split Scenes**; output to **Wait**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - `VIDEO ID` uses `$itemIndex`, which is zero-based and may not match `scene_number`
  - `SCRIPT` is stored as serialized scene JSON, and downstream expects to parse it
  - Column names must match exactly

#### Wait
- **Type / role:** Wait; adds delay before updating the source sheet.
- **Configuration choices:** `amount = 60`; verify unit in UI.
- **Input / output connections:** From **UpdateSheet**; output to **UpdateSheet1**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases/failure types:** Same wait-unit ambiguity as above.

#### UpdateSheet1
- **Type / role:** Google Sheets update; writes generated `ID` and marks source row as processing.
- **Configuration choices:**
  - Match on `row_number`
  - Writes:
    - `ID = {{$('Input').first().json.id}}`
    - `SCRIPT = {{$('Input').item.json.demo_script}}`
    - `row_number = {{$('Input').item.json.row_number}}`
    - `LAUNCH CREATION = Processing`
  - Executes once
- **Input / output connections:** From **Wait**; output to **Get row(s) in sheet**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - This update happens after scene rows are appended, so any earlier failure path trying to match source row by `ID` can miss
  - `.item` and `.first()` mixed usage should be tested carefully if multiple items are present

---

## Block 6 — Per-Scene Video Generation

### Overview
This block reads scene rows from **Video Data**, submits a VEO3 generation request for each scene, polls status, and branches into success or failure updates.

### Nodes Involved
- Get row(s) in sheet
- videoInput
- UpdateSheet10
- HTTP Request12
- HTTP Request13
- Video Status Check
- Wait6
- HTTP Request1
- Upload file1
- Share file1
- UpdateSheet2
- UpdateSheet4
- Wait1
- UpdateSheet9

### Node Details

#### Get row(s) in sheet
- **Type / role:** Google Sheets read; fetches scene rows by parent `ID`.
- **Configuration choices:** Filters **Video Data** where `ID = {{$('Input').item.json.id}}`; executes once.
- **Input / output connections:** From **UpdateSheet1**; output to **videoInput**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - If no rows were appended, nothing continues
  - Executes once may limit item processing expectations

#### videoInput
- **Type / role:** Set; maps scene row columns into internal names for video generation.
- **Configuration choices:** Creates:
  - `ID`
  - `VIDEO ID`
  - `SCRIPT`
  - `IMAGE URL`
- **Input / output connections:** From **Get row(s) in sheet** and regeneration block; outputs to **HTTP Request12** and **UpdateSheet10**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failure types:** Depends on exact sheet headers and values.

#### UpdateSheet10
- **Type / role:** Google Sheets update; marks each scene row in **Video Data** as processing.
- **Configuration choices:**
  - Match on `VIDEO ID`
  - Writes:
    - `LAUNCH = Processing`
    - `VIDEO ID = {{$json['VIDEO ID']}}`
  - Executes once
- **Input / output connections:** From **videoInput**; no downstream connection.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:** `executeOnce = true` may only update one item despite multiple scene items; verify actual runtime semantics.

#### HTTP Request12
- **Type / role:** HTTP Request; submits VEO3 video generation task to Kie.
- **Configuration choices:**
  - POST `https://api.kie.ai/api/v1/veo/generate`
  - Body:
    - `prompt = {{ JSON.stringify($json.SCRIPT.parseJson()) }}`
    - `imageUrls = [IMAGE URL, IMAGE URL]`
    - `model = veo3_fast`
    - `aspect_ratio = Auto`
    - `generationType = FIRST_AND_LAST_FRAMES_2_VIDEO`
  - Header auth
- **Input / output connections:** From **videoInput**; output to **HTTP Request13**.
- **Key expressions:** `$json.SCRIPT.parseJson()`, `$json['IMAGE URL']`
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases/failure types:**
  - Assumes `SCRIPT` is valid JSON string from prior append
  - Uses same image as both first and last frame
  - Aspect ratio is forced to `Auto`, not source aspect ratio
  - API contract must accept prompt object/string as provided

#### HTTP Request13
- **Type / role:** HTTP Request; polls scene video task status.
- **Configuration choices:** GET `https://api.kie.ai/api/v1/veo/record-info?taskId={{ $json.data.taskId }}`
- **Input / output connections:** From **HTTP Request12** and loop from **Wait6**; output to **Video Status Check**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases/failure types:** Missing `taskId`, rate limits, auth issues.

#### Video Status Check
- **Type / role:** Switch; classifies video task result.
- **Configuration choices:**
  - `Success` if `successFlag == 1` or `state == success`
  - `Failed` if `successFlag >= 2` or `state == failed`
  - `Processing` if in waiting/processing/generating or successFlag 0
  - fallback `Unknown Status`
- **Input / output connections:**
  - Success → **HTTP Request1**
  - Failed → **UpdateSheet4**
  - Processing → **Wait6**
  - Unknown → **UpdateSheet4**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failure types:** Same pattern risks as image polling.

#### Wait6
- **Type / role:** Wait; long poll delay for video generation.
- **Configuration choices:** `unit = minutes`, `amount = 10`
- **Input / output connections:** From **Video Status Check** processing branch; back to **HTTP Request13**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases/failure types:** Long-running executions; storage/timeout policies in self-hosted deployments.

#### HTTP Request1
- **Type / role:** HTTP Request; downloads completed scene clip.
- **Configuration choices:** URL from `{{$('Video Status Check').item.json.data.response.resultUrls[0]}}`
- **Input / output connections:** From **Video Status Check** success branch; output to **Upload file1**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases/failure types:**
  - Assumes exact Kie response nesting
  - URL expiration possible

#### Upload file1
- **Type / role:** Google Drive upload; stores per-scene clip.
- **Configuration choices:**
  - File name `{{$('videoInput').item.json['VIDEO ID']}}.mp4`
  - Folder ID `1HlQRqwXfdYEm8zIsxQ0NsEYRAUEaFvMD`
- **Input / output connections:** From **HTTP Request1**; output to **Share file1**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failure types:** Binary handling and Drive permissions.

#### Share file1
- **Type / role:** Google Drive share; exposes clip publicly.
- **Configuration choices:** reader / anyone.
- **Input / output connections:** From **Upload file1**; output to **UpdateSheet2**.
- **Version-specific requirements:** Type version `3`.

#### UpdateSheet2
- **Type / role:** Google Sheets update; marks a scene clip as completed.
- **Configuration choices:**
  - Match on `VIDEO ID`
  - Writes:
    - `Error = No Error`
    - `LAUNCH = Completed`
    - `VIDEO ID`
    - `VIDEO CLIP LINK = {{$('Upload file1').item.json.webContentLink}}`
    - `VIDEO VIEW CLIP = {{$('Upload file1').item.json.webViewLink.split("?")[0]}}`
- **Input / output connections:** From **Share file1**; output to **Get row(s) for Stitch**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:** Assumes upload node exposes both Drive links.

#### UpdateSheet4
- **Type / role:** Google Sheets update; marks a scene clip as failed.
- **Configuration choices:**
  - Match on `VIDEO ID`
  - Writes:
    - `Error = {{$json.data.errorMessage}}`
    - `LAUNCH = Failed`
    - `VIDEO ID`
- **Input / output connections:** From **Video Status Check** failed/unknown; output to **Wait1**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:** If `errorMessage` missing, field may be blank.

#### Wait1
- **Type / role:** Wait; delay before marking the parent source row failed.
- **Configuration choices:** default wait parameters; verify UI.
- **Input / output connections:** From **UpdateSheet4**; output to **UpdateSheet9**.
- **Version-specific requirements:** Type version `1.1`.

#### UpdateSheet9
- **Type / role:** Google Sheets update; marks the parent source video as failed after a scene clip failure.
- **Configuration choices:**
  - Match on `ID`
  - Writes:
    - `ID = {{$('videoInput').item.json['VIDEO ID'].slice(0, -2)}}`
    - `LAUNCH CREATION = Failed`
- **Input / output connections:** From **Wait1**; terminal in this branch.
- **Key expressions:** `slice(0, -2)`
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:**
  - This assumes every `VIDEO ID` ends with exactly two characters like `_0`, `_1`
  - It breaks for scene indexes with two digits, e.g. `_10`
  - This is a real design weakness

---

## Block 7 — Completion Check, Sorting, and Final Stitching

### Overview
After each successful clip, this block checks whether all expected scenes for the parent video are completed. If all are ready, it merges them in order and updates the source row with final links.

### Nodes Involved
- Get row(s) for Stitch
- Code in JavaScript
- If
- Sort
- Aggregate Tempfile URLs2
- Generate media using AI model1
- download-video8
- Upload file2
- Share file2
- UpdateSheet3

### Node Details

#### Get row(s) for Stitch
- **Type / role:** Google Sheets read; fetches completed scene rows for a parent video.
- **Configuration choices:**
  - Filters:
    - `ID = {{$('videoInput').item.json.ID}}`
    - `LAUNCH = Completed`
  - Executes once
- **Input / output connections:** From **UpdateSheet2**; output to **Code in JavaScript**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:** Only completed rows are fetched, so completeness must be inferred against `TOTAL SCENES`.

#### Code in JavaScript
- **Type / role:** Code node; determines whether all scenes are complete.
- **Configuration choices:** 
  - Counts returned items
  - Reads `TOTAL SCENES` from first item
  - Sets `isComplete = totalItems >= totalScenes` on each item
- **Input / output connections:** From **Get row(s) for Stitch**; output to **If**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases/failure types:**
  - Fails if no items returned
  - Assumes all rows share the same `TOTAL SCENES`
  - Uses `>=` rather than strict equality

#### If
- **Type / role:** If node; proceeds only when all scenes are complete.
- **Configuration choices:** `isComplete == true`
- **Input / output connections:**
  - True → **Sort**
  - False → **Image Gen Error**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases/failure types:**
  - False branch uses **Image Gen Error**, which is semantically mismatched and error text is unrelated to stitching

#### Sort
- **Type / role:** Sort; orders scenes by `SCENE NO`.
- **Configuration choices:** Ascending sort on `SCENE NO`
- **Input / output connections:** From **If** true branch; output to **Aggregate Tempfile URLs2**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failure types:** `SCENE NO` stored as string may sort lexicographically if not coerced.

#### Aggregate Tempfile URLs2
- **Type / role:** Aggregate; builds an array of clip URLs.
- **Configuration choices:** Aggregates `VIDEO CLIP LINK`
- **Input / output connections:** From **Sort**; output to **Generate media using AI model1**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failure types:** Empty links or private links will break merge.

#### Generate media using AI model1
- **Type / role:** fal.ai node; merges clips using FFmpeg API.
- **Configuration choices:**
  - Model: `fal-ai/ffmpeg-api/merge-videos`
  - Waits for completion
  - `maxWaitTime = 600`
  - `pollInterval = 5`
  - Parameter `video_urls = {{ JSON.stringify($json['VIDEO CLIP LINK']) }}`
- **Input / output connections:** From **Aggregate Tempfile URLs2**; output to **download-video8**.
- **Version-specific requirements:** fal.ai node type version `1`.
- **Edge cases/failure types:**
  - fal.ai credentials required
  - Merge may fail on inconsistent codec/container inputs
  - Serialized array format must match model expectations

#### download-video8
- **Type / role:** HTTP Request; downloads merged final video.
- **Configuration choices:** URL from `{{$json.video.url}}`
- **Input / output connections:** From **Generate media using AI model1**; output to **Upload file2**.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases/failure types:** Temporary URL expiry.

#### Upload file2
- **Type / role:** Google Drive upload; stores final stitched video.
- **Configuration choices:**
  - File name `{{$('videoInput').item.json.ID}}.mp4`
  - Same target folder as clip uploads: `1HlQRqwXfdYEm8zIsxQ0NsEYRAUEaFvMD`
- **Input / output connections:** From **download-video8**; output to **Share file2**.
- **Version-specific requirements:** Type version `3`.

#### Share file2
- **Type / role:** Google Drive share; public final video access.
- **Configuration choices:** reader / anyone.
- **Input / output connections:** From **Upload file2**; output to **UpdateSheet3**.
- **Version-specific requirements:** Type version `3`.

#### UpdateSheet3
- **Type / role:** Google Sheets update; final completion update for source row in **Videos**.
- **Configuration choices:**
  - Match on `ID`
  - Writes:
    - `ID = {{$('videoInput').first().json.ID}}`
    - `VIDEO LINK = {{$('Upload file2').item.json.webContentLink}}`
    - `VIDEO VIEW LINK = {{$('Upload file2').item.json.webViewLink.split("?")[0]}}`
    - `LAUNCH CREATION = Completed`
- **Input / output connections:** From **Share file2**; terminal in this success branch.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:** Source row must already contain the generated `ID`.

---

## Block 8 — Regeneration / Retry of Scene Clips

### Overview
This block periodically looks for scene rows marked `Redo` and re-runs the clip generation path. It tries to choose the least-complete parent group when multiple IDs are present.

### Nodes Involved
- Schedule Trigger
- Get row(s) in sheet1
- Code in JavaScript2
- videoInput

### Node Details

#### Schedule Trigger
- **Type / role:** `scheduleTrigger`; retry entry point.
- **Configuration choices:** Runs every 15 minutes.
- **Input / output connections:** Outputs to **Get row(s) in sheet1**.
- **Version-specific requirements:** Type version `1.3`.

#### Get row(s) in sheet1
- **Type / role:** Google Sheets read; finds `Redo` rows in **Video Data**.
- **Configuration choices:**
  - Filter: `LAUNCH = Redo`
  - Executes once
- **Input / output connections:** From **Schedule Trigger**; output to **Code in JavaScript2**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases/failure types:** No redo rows means no action.

#### Code in JavaScript2
- **Type / role:** Code node; groups redo rows by `ID`.
- **Configuration choices:**
  - If all rows share one `ID`, returns all
  - Otherwise returns group(s) with minimum row count
- **Input / output connections:** From **Get row(s) in sheet1**; output to **videoInput**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases/failure types:**
  - “Minimum entry count” is only a proxy for least-complete video
  - Mixed data quality may produce surprising selections

#### videoInput
- Already described in Block 6.
- **Sub-workflow reference:** none; it re-enters the same scene generation path.

---

## Block 9 — Sticky Notes / Embedded Documentation

### Overview
These nodes are purely descriptive and do not participate in execution, but they provide important operator context about setup, purpose, and customization.

### Nodes Involved
- Sticky Note1
- Sticky Note
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details
All are `n8n-nodes-base.stickyNote`, type version `1`, with no runtime behavior.

- **Sticky Note1:** Overall workflow summary, setup checklist, and customization tips.
- **Sticky Note:** Intake + Image Generation block description.
- **Sticky Note2:** Script Generation block description.
- **Sticky Note3 / Sticky Note4:** Video Generation block descriptions.
- **Sticky Note5:** Regeneration section label.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Video Ideas | n8n-nodes-base.googleSheets | Read first source row from Videos where `LAUNCH CREATION = Create` | Schedule Trigger1; When Executed by Another Workflow | Input; UpdateSheet6 | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| HTTP Request12 | n8n-nodes-base.httpRequest | Submit VEO3 scene video generation task | videoInput | HTTP Request13 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| HTTP Request13 | n8n-nodes-base.httpRequest | Poll VEO3 task status | HTTP Request12; Wait6 | Video Status Check | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Wait6 | n8n-nodes-base.wait | Delay between video status polls | Video Status Check | HTTP Request13 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation |  |  | # UGC Ads Factory  **Automated AI pipeline for UGC-style video ads**  `Create row → Reference image → Scene plan → Scene clips → Stitch final video → Publish links`  ### How it works  - The trigger reads rows in **Videos** where `LAUNCH CREATION = Create`, normalizes fields, and marks each row `Processing`.  - Kie creates a reference image. The workflow polls until completion, then uploads and shares the image in Google Drive.  - Claude-4o-mini analyzes visual details. Claude Opus converts the script into ordered 8-second scenes with consistent visual context and movement prompts.  - Each scene is appended to **Video Data** using `VIDEO ID`, `SCENE NO`, `SCRIPT`, and `TOTAL SCENES`.  - VEO3 generates one clip per scene. Successful clips are uploaded and linked; failures are logged with error state.  - After all scenes complete, clips are sorted and merged via fal.ai FFmpeg.  - Final video links are written back to **Videos**, and campaign status becomes `Completed`.  ### Setup  - Configure credentials: Google Sheets, Google Drive, Kie, OpenRouter, OpenAI, fal.ai.  - Confirm sheet/tab names and column headers exactly match workflow mapping.  - Set destination folder IDs for image clips and final stitched videos.  ### Customization  - Tune scene length, polling intervals, model defaults, retry policy, and failure-routing behavior. |
| Video Status Check | n8n-nodes-base.switch | Branch on video generation status | HTTP Request13 | HTTP Request1; UpdateSheet4; Wait6; UpdateSheet4 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Prompt Agent LLM | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM backend for image prompt agent |  | Nano Banana Prompt Agent | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Prompt Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for image prompt JSON |  | Nano Banana Prompt Agent | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Image Gen Error | n8n-nodes-base.stopAndError | Terminal error node | If |  | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Script Agent LLM | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM backend for scene generation |  | Generate Scene Scripts3 | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Scene Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for scenes array |  | Generate Scene Scripts3 | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Nano Banana Prompt Agent | @n8n/n8n-nodes-langchain.agent | Create structured photorealistic image prompt | Input; Prompt Agent LLM; Prompt Output Parser | Generate UGC Image | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Generate UGC Image | n8n-nodes-base.httpRequest | Submit image-generation task to Kie | Nano Banana Prompt Agent | Image Gen Response Check | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Image Gen Response Check | n8n-nodes-base.switch | Validate immediate Kie image task response | Generate UGC Image | Poll Image Status; UpdateSheet5; UpdateSheet5 | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Workflow Configuration3 | n8n-nodes-base.set | Package shared image links and source fields | Share file | Analyze UGC Image3 | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Split Scenes | n8n-nodes-base.splitOut | Emit one item per generated scene | Generate Scene Scripts3 | UpdateSheet | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Analyze UGC Image3 | @n8n/n8n-nodes-langchain.openAi | Analyze generated reference image for scene consistency | Workflow Configuration3 | Generate Scene Scripts3 | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Generate Scene Scripts3 | @n8n/n8n-nodes-langchain.agent | Convert script into structured scene plan | Analyze UGC Image3; Script Agent LLM; Scene Output Parser | Split Scenes | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Input | n8n-nodes-base.set | Normalize source row and assign unique ID | Get Video Ideas | Nano Banana Prompt Agent | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| videoInput | n8n-nodes-base.set | Normalize scene row fields for clip generation | Get row(s) in sheet; Code in JavaScript2 | HTTP Request12; UpdateSheet10 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| HTTP Request1 | n8n-nodes-base.httpRequest | Download completed scene clip | Video Status Check | Upload file1 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Wait 10s | n8n-nodes-base.wait | Delay image polling loop | Image Status Check | Poll Image Status | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Image Poll Error | n8n-nodes-base.stopAndError | Stop execution after permanent image failure | UpdateSheet5 |  | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Poll Image Status | n8n-nodes-base.httpRequest | Poll image task status | Image Gen Response Check; Wait 10s | Image Status Check | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Image Status Check | n8n-nodes-base.switch | Branch on image generation status | Poll Image Status | Download Image; UpdateSheet5; Wait 10s; UpdateSheet5 | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Upload file | n8n-nodes-base.googleDrive | Upload generated reference image to Drive | Download Image | Share file | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Download Image | n8n-nodes-base.httpRequest | Download final image from Kie result URL | Image Status Check | Upload file | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Share file | n8n-nodes-base.googleDrive | Make reference image public | Upload file | Workflow Configuration3 | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| UpdateSheet | n8n-nodes-base.googleSheets | Append scene rows to Video Data | Split Scenes | Wait | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| UpdateSheet1 | n8n-nodes-base.googleSheets | Write generated ID and processing state to source row | Wait | Get row(s) in sheet | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Fetch scene rows by parent ID | UpdateSheet1 | videoInput | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Upload file1 | n8n-nodes-base.googleDrive | Upload scene clip to Drive | HTTP Request1 | Share file1 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Share file1 | n8n-nodes-base.googleDrive | Make scene clip public | Upload file1 | UpdateSheet2 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| UpdateSheet2 | n8n-nodes-base.googleSheets | Mark scene row completed and store clip links | Share file1 | Get row(s) for Stitch | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Get row(s) for Stitch | n8n-nodes-base.googleSheets | Read completed scene rows for merge eligibility | UpdateSheet2 | Code in JavaScript | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Sort | n8n-nodes-base.sort | Order completed clips by scene number | If | Aggregate Tempfile URLs2 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Code in JavaScript | n8n-nodes-base.code | Compute completeness against total scenes | Get row(s) for Stitch | If | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| If | n8n-nodes-base.if | Allow stitching only when all scenes are complete | Code in JavaScript | Sort; Image Gen Error | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| download-video8 | n8n-nodes-base.httpRequest | Download stitched final video | Generate media using AI model1 | Upload file2 |  |
| Upload file2 | n8n-nodes-base.googleDrive | Upload stitched final video to Drive | download-video8 | Share file2 |  |
| Share file2 | n8n-nodes-base.googleDrive | Make final video public | Upload file2 | UpdateSheet3 |  |
| UpdateSheet3 | n8n-nodes-base.googleSheets | Mark source request completed and store final links | Share file2 |  |  |
| UpdateSheet4 | n8n-nodes-base.googleSheets | Mark scene clip failed and store error | Video Status Check | Wait1 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| UpdateSheet5 | n8n-nodes-base.googleSheets | Mark source request failed on image-generation failure | Image Gen Response Check; Image Status Check | Image Poll Error | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Get row(s) in sheet1 | n8n-nodes-base.googleSheets | Fetch redo scene rows | Schedule Trigger | Code in JavaScript2 | ## Regeneration of the video |
| UpdateSheet9 | n8n-nodes-base.googleSheets | Mark parent source request failed after scene failure | Wait1 |  | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| UpdateSheet10 | n8n-nodes-base.googleSheets | Mark scene row processing before regeneration/submission | videoInput |  | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Code in JavaScript2 | n8n-nodes-base.code | Group redo rows and select least-complete set | Get row(s) in sheet1 | videoInput | ## Regeneration of the video |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Retry pipeline trigger every 15 minutes |  | Get row(s) in sheet1 | ## Regeneration of the video |
| Schedule Trigger1 | n8n-nodes-base.scheduleTrigger | Main pipeline trigger every 30 minutes |  | Get Video Ideas | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Generate media using AI model1 | @fal-ai/n8n-nodes-fal.falAi | Merge completed scene clips into final video | Aggregate Tempfile URLs2 | download-video8 |  |
| Aggregate Tempfile URLs2 | n8n-nodes-base.aggregate | Build array of clip URLs for merge | Sort | Generate media using AI model1 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Wait | n8n-nodes-base.wait | Delay before updating source row ID | UpdateSheet | UpdateSheet1 | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Wait1 | n8n-nodes-base.wait | Delay before marking parent source row failed | UpdateSheet4 | UpdateSheet9 | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Programmatic entry point from another workflow |  | Get Video Ideas | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| UpdateSheet6 | n8n-nodes-base.googleSheets | Immediately mark source row as processing by row number | Get Video Ideas |  | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Intake + Image Generation  Pulls `Create` rows from **Videos**, normalizes inputs, assigns run IDs, and sets `LAUNCH CREATION` to `Processing`. Builds Nano Banana prompt, submits Kie image task, polls status, and stores a shared Drive image URL for downstream nodes. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Script Generation  Analyzes the reference image for visual consistency, then uses Claude Opus to split the source script into ordered 8-second scenes. Each scene includes speech text, fixed visual description, movement guidance, and scene metadata for tracking. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Video Generation  Writes scenes to **Video Data** and generates one VEO3 clip per scene using `VIDEO ID` and scene JSON. Polls until terminal status, uploads completed clips to Drive, and updates each scene row with clip links, state, or error details. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation |  |  | ## Regeneration of the video |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the source Google Sheet structure**
   - Create a spreadsheet named as desired.
   - Add a tab named **Videos** with at least these columns:
     - `ID`
     - `SCRIPT`
     - `CHARACTER/SETTING DESCRIPTION`
     - `ASPECT RATIO`
     - `VIDEO VIEW LINK`
     - `VIDEO LINK`
     - `LAUNCH CREATION`
     - `row_number`
   - Add another tab named **Video Data** with at least:
     - `ID`
     - `VIDEO ID`
     - `IMAGE URL`
     - `TOTAL SCENES`
     - `SCENE NO`
     - `SCRIPT`
     - `VIDEO VIEW CLIP`
     - `VIDEO CLIP LINK`
     - `LAUNCH`
     - `Error`
     - `row_number`

2. **Create credentials**
   - Google Sheets credential
   - Google Drive credential
   - OpenRouter credential for LangChain OpenRouter nodes
   - OpenAI credential for the image analysis node
   - HTTP Header Auth credential for Kie API
   - fal.ai credential for the merge node

3. **Create entry triggers**
   - Add **Schedule Trigger1**:
     - every 30 minutes
   - Add **When Executed by Another Workflow**:
     - input source `passthrough`
   - Add **Schedule Trigger**:
     - every 15 minutes

4. **Add Get Video Ideas**
   - Google Sheets node
   - Operation: read rows/filter rows
   - Sheet: **Videos**
   - Filter: `LAUNCH CREATION = Create`
   - Enable `returnFirstMatch`
   - Connect **Schedule Trigger1** and **When Executed by Another Workflow** to it

5. **Add UpdateSheet6**
   - Google Sheets update node
   - Target: **Videos**
   - Match column: `row_number`
   - Set:
     - `row_number = {{$json.row_number}}`
     - `LAUNCH CREATION = Processing`
   - Connect from **Get Video Ideas**

6. **Add Input**
   - Set node
   - Add fields:
     - `character_description = {{$json['CHARACTER/SETTING DESCRIPTION']}}`
     - `demo_script = {{$json.SCRIPT}}`
     - `aspect_ratio = {{$json['ASPECT RATIO'] || '9:16'}}`
     - `id = {{$execution.id + '_' + Date.now().toString() + Math.random().toString(36).substring(2, 10)}}`
     - `row_number = {{$json.row_number}}`
   - Connect from **Get Video Ideas**

7. **Add Prompt Agent LLM**
   - LangChain OpenRouter chat model
   - Model: `anthropic/claude-opus-4.5`

8. **Add Prompt Output Parser**
   - Structured output parser
   - Manual JSON schema requiring:
     - `prompt`
     - `model`
     - `aspectRatio`
     - `resolution`

9. **Add Nano Banana Prompt Agent**
   - LangChain agent
   - Prompt type: define
   - Main prompt references:
     - character description
     - script
     - aspect ratio
   - Add the system prompt from the workflow logic:
     - iPhone selfie framing
     - extreme background blur
     - realistic skin
     - natural lighting
     - no visible phone
     - output only valid JSON
   - Connect:
     - **Input** → main input
     - **Prompt Agent LLM** → AI language model
     - **Prompt Output Parser** → AI output parser

10. **Add Generate UGC Image**
    - HTTP Request
    - Method: POST
    - URL: `https://api.kie.ai/api/v1/jobs/createTask`
    - Authentication: generic header auth
    - Body as JSON:
      - model `nano-banana-2`
      - callback URL placeholder or your real callback
      - input.prompt from `{{$json.output.prompt}}`
      - input.aspect_ratio from `{{$('Input').item.json.aspect_ratio}}`
      - input.resolution `2K`
      - input.output_format `png`
    - Connect from **Nano Banana Prompt Agent**

11. **Add Image Gen Response Check**
    - Switch node
    - Output 1 `Success`: `{{$json.code}} == 200`
    - Output 2 `Error`: `{{$json.code}} != 200`
    - Fallback: `Unexpected`
    - Connect from **Generate UGC Image**

12. **Add Poll Image Status**
    - HTTP Request
    - URL: `https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json.data.taskId }}`
    - Auth: same Kie header auth
    - Connect success branch from **Image Gen Response Check**

13. **Add Image Status Check**
    - Switch node
    - Success when:
      - `data.successFlag == 1` OR `data.state == success`
    - Failed when:
      - `data.successFlag >= 2` OR `data.state == failed`
    - Processing when:
      - `data.successFlag == 0` OR `data.state` in `waiting`, `processing`, `generating`
    - Fallback: `Unknown Status`
    - Connect from **Poll Image Status**

14. **Add Wait 10s**
    - Wait node
    - Set the intended poll delay manually in the UI
    - Connect `Processing` from **Image Status Check** back into **Poll Image Status**

15. **Add UpdateSheet5**
    - Google Sheets update node
    - Target: **Videos**
    - Match on `ID`
    - Set:
      - `ID = {{$('Input').first().json.id}}`
      - `LAUNCH CREATION = Failed`
    - Connect error branches from:
      - **Image Gen Response Check**
      - **Image Status Check** failed and fallback

16. **Add Image Poll Error**
    - Stop and Error node
    - Use an error message including:
      - state
      - success flag
      - error message
      - task ID
    - Connect from **UpdateSheet5**

17. **Add Download Image**
    - HTTP Request
    - URL: `{{$json.data.resultJson.parseJson().resultUrls[0]}}`
    - Configure to download binary content
    - Connect from **Image Status Check** success

18. **Add Upload file**
    - Google Drive upload
    - Folder ID: your image folder
    - File name: `{{$('Input').item.json.id}}.png`
    - Connect from **Download Image**

19. **Add Share file**
    - Google Drive share
    - File ID: uploaded file ID
    - Permission: anyone / reader
    - Connect from **Upload file**

20. **Add Workflow Configuration3**
    - Set node with:
      - `referenceImage = {{$('Upload file').item.json.webViewLink}}`
      - `download_image = {{$('Upload file').item.json.webContentLink}}`
      - `character_description = {{$('Input').item.json.character_description}}`
      - `demo_script = {{$('Input').item.json.demo_script}}`
      - `aspect_ratio = {{$('Input').item.json.aspect_ratio}}`
    - Connect from **Share file**

21. **Add Analyze UGC Image3**
    - OpenAI node
    - Resource: image
    - Operation: analyze
    - Model: `gpt-4o-mini`
    - Detail: high
    - Image URL: `{{$json.download_image}}`
    - Connect from **Workflow Configuration3**

22. **Add Script Agent LLM**
    - LangChain OpenRouter chat model
    - Model: `anthropic/claude-opus-4.5`

23. **Add Scene Output Parser**
    - Structured output parser
    - Schema with root `scenes` array
    - Each item contains:
      - `scene_number`
      - `script`
      - `visual_description`
      - `audio_description`
      - `movement`
      - `duration_seconds`

24. **Add Generate Scene Scripts3**
    - LangChain agent
    - Feed:
      - original character prompt
      - image analysis
      - source script
    - System rules:
      - split into ~8-second scene chunks
      - same visual description in every scene
      - duration always 8 seconds
      - movement depends on environment
      - return valid JSON only
    - Connect:
      - **Analyze UGC Image3** → main
      - **Script Agent LLM** → AI model
      - **Scene Output Parser** → parser

25. **Add Split Scenes**
    - Split Out node
    - Field: `output.scenes`
    - Connect from **Generate Scene Scripts3**

26. **Add UpdateSheet**
    - Google Sheets append
    - Target sheet: **Video Data**
    - Map:
      - `ID = {{$('Input').first().json.id}}`
      - `LAUNCH = Processing`
      - `SCRIPT = {{JSON.stringify($json)}}`
      - `SCENE NO = {{$json.scene_number}}`
      - `VIDEO ID = {{$('Input').first().json.id}}_{{$itemIndex}}`
      - `IMAGE URL = {{$('Workflow Configuration3').first().json.download_image}}`
      - `TOTAL SCENES = {{$('Generate Scene Scripts3').first().json.output.scenes.length}}`
    - Connect from **Split Scenes**

27. **Add Wait**
    - Wait node
    - Add a small delay similar to exported workflow
    - Connect from **UpdateSheet**

28. **Add UpdateSheet1**
    - Google Sheets update
    - Target: **Videos**
    - Match on `row_number`
    - Set:
      - `ID = {{$('Input').first().json.id}}`
      - `SCRIPT = {{$('Input').item.json.demo_script}}`
      - `row_number = {{$('Input').item.json.row_number}}`
      - `LAUNCH CREATION = Processing`
    - Connect from **Wait**

29. **Add Get row(s) in sheet**
    - Google Sheets read
    - Sheet: **Video Data**
    - Filter `ID = {{$('Input').item.json.id}}`
    - Execute once
    - Connect from **UpdateSheet1**

30. **Add videoInput**
    - Set node
    - Create:
      - `ID = {{$json.ID}}`
      - `VIDEO ID = {{$json['VIDEO ID']}}`
      - `SCRIPT = {{$json.SCRIPT}}`
      - `IMAGE URL = {{$json['IMAGE URL']}}`
    - Connect from **Get row(s) in sheet**

31. **Add UpdateSheet10**
    - Google Sheets update
    - Target: **Video Data**
    - Match on `VIDEO ID`
    - Set:
      - `LAUNCH = Processing`
      - `VIDEO ID = {{$json['VIDEO ID']}}`
    - Connect from **videoInput**

32. **Add HTTP Request12**
    - HTTP Request
    - Method: POST
    - URL: `https://api.kie.ai/api/v1/veo/generate`
    - Auth: Kie header auth
    - JSON body:
      - prompt = `{{$json.SCRIPT.parseJson()}}` serialized appropriately
      - imageUrls = `[IMAGE URL, IMAGE URL]`
      - model = `veo3_fast`
      - aspect_ratio = `Auto`
      - generationType = `FIRST_AND_LAST_FRAMES_2_VIDEO`
    - Connect from **videoInput**

33. **Add HTTP Request13**
    - HTTP Request
    - URL: `https://api.kie.ai/api/v1/veo/record-info?taskId={{ $json.data.taskId }}`
    - Connect from **HTTP Request12**

34. **Add Video Status Check**
    - Switch node with the same success / failed / processing logic used for image polling
    - Connect from **HTTP Request13**

35. **Add Wait6**
    - Wait node
    - Unit: minutes
    - Amount: 10
    - Connect `Processing` branch back to **HTTP Request13**

36. **Add HTTP Request1**
    - HTTP Request
    - Download from `{{$('Video Status Check').item.json.data.response.resultUrls[0]}}`
    - Configure binary download
    - Connect from success branch of **Video Status Check**

37. **Add Upload file1**
    - Google Drive upload
    - File name: `{{$('videoInput').item.json['VIDEO ID']}}.mp4`
    - Folder: your clip folder
    - Connect from **HTTP Request1**

38. **Add Share file1**
    - Google Drive share
    - anyone / reader
    - Connect from **Upload file1**

39. **Add UpdateSheet2**
    - Google Sheets update
    - Sheet: **Video Data**
    - Match on `VIDEO ID`
    - Set:
      - `Error = No Error`
      - `LAUNCH = Completed`
      - `VIDEO ID = {{$('videoInput').item.json['VIDEO ID']}}`
      - `VIDEO CLIP LINK = {{$('Upload file1').item.json.webContentLink}}`
      - `VIDEO VIEW CLIP = {{$('Upload file1').item.json.webViewLink.split('?')[0]}}`
    - Connect from **Share file1**

40. **Add UpdateSheet4**
    - Google Sheets update
    - Match on `VIDEO ID`
    - Set:
      - `Error = {{$json.data.errorMessage}}`
      - `LAUNCH = Failed`
      - `VIDEO ID = {{$('videoInput').item.json['VIDEO ID']}}`
    - Connect failed and fallback branches from **Video Status Check**

41. **Add Wait1**
    - Wait node
    - Connect from **UpdateSheet4**

42. **Add UpdateSheet9**
    - Google Sheets update
    - Target: **Videos**
    - Match on `ID`
    - Set:
      - `ID = {{$('videoInput').item.json['VIDEO ID'].slice(0, -2)}}`
      - `LAUNCH CREATION = Failed`
    - Connect from **Wait1**
   - Recommended improvement: replace this expression with a safer parent-ID extractor.

43. **Add Get row(s) for Stitch**
    - Google Sheets read
    - Sheet: **Video Data**
    - Filters:
      - `ID = {{$('videoInput').item.json.ID}}`
      - `LAUNCH = Completed`
    - Execute once
    - Connect from **UpdateSheet2**

44. **Add Code in JavaScript**
    - Code node
    - Use logic:
      - count rows
      - read `TOTAL SCENES`
      - set `isComplete = totalItems >= totalScenes`
    - Connect from **Get row(s) for Stitch**

45. **Add If**
    - If node
    - Condition: `{{$json.isComplete}} == true`
    - Connect from **Code in JavaScript**

46. **Add Sort**
    - Sort node
    - Sort by `SCENE NO`
    - Connect true branch from **If**

47. **Add Aggregate Tempfile URLs2**
    - Aggregate node
    - Aggregate field `VIDEO CLIP LINK`
    - Connect from **Sort**

48. **Add Generate media using AI model1**
    - fal.ai node
    - Model: `fal-ai/ffmpeg-api/merge-videos`
    - Enable wait for completion
    - Set:
      - max wait 600
      - poll interval 5
      - `video_urls = {{ JSON.stringify($json['VIDEO CLIP LINK']) }}`
    - Connect from **Aggregate Tempfile URLs2**

49. **Add download-video8**
    - HTTP Request
    - Download from `{{$json.video.url}}`
    - Configure binary download
    - Connect from **Generate media using AI model1**

50. **Add Upload file2**
    - Google Drive upload
    - File name: `{{$('videoInput').item.json.ID}}.mp4`
    - Folder: your final video folder
    - Connect from **download-video8**

51. **Add Share file2**
    - Google Drive share
    - anyone / reader
    - Connect from **Upload file2**

52. **Add UpdateSheet3**
    - Google Sheets update
    - Target: **Videos**
    - Match on `ID`
    - Set:
      - `ID = {{$('videoInput').first().json.ID}}`
      - `VIDEO LINK = {{$('Upload file2').item.json.webContentLink}}`
      - `VIDEO VIEW LINK = {{$('Upload file2').item.json.webViewLink.split('?')[0]}}`
      - `LAUNCH CREATION = Completed`
    - Connect from **Share file2**

53. **Build the regeneration path**
    - Add **Get row(s) in sheet1**
      - Sheet: **Video Data**
      - Filter `LAUNCH = Redo`
      - Execute once
      - Connect from **Schedule Trigger**
    - Add **Code in JavaScript2**
      - Group rows by `ID`
      - If only one group, return it
      - Otherwise return group(s) with minimum item count
      - Connect from **Get row(s) in sheet1**
    - Connect **Code in JavaScript2** to **videoInput**

54. **Optional but recommended corrections before production**
    - Replace placeholder callback URL in image generation
    - Verify all Wait node time units
    - Replace `slice(0, -2)` in **UpdateSheet9**
    - Consider strict numeric sorting for `SCENE NO`
    - Consider using scene number instead of `$itemIndex` for `VIDEO ID`
    - Add explicit no-data guards for Google Sheets read nodes

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| UGC Ads Factory — Automated AI pipeline for UGC-style video ads | Internal sticky note summary in workflow |
| High-level process: `Create row → Reference image → Scene plan → Scene clips → Stitch final video → Publish links` | Workflow overview note |
| Required credentials: Google Sheets, Google Drive, Kie, OpenRouter, OpenAI, fal.ai | Setup guidance from workflow note |
| Confirm sheet/tab names and column headers exactly match workflow mapping | Setup guidance |
| Set destination folder IDs for image clips and final stitched videos | Setup guidance |
| Tune scene length, polling intervals, model defaults, retry policy, and failure-routing behavior | Customization guidance |
| Intake + Image Generation block note: pulls `Create` rows, normalizes inputs, sets processing state, builds prompt, submits Kie image task, polls, and stores shared Drive image URL | Contextual sticky note |
| Script Generation block note: analyzes reference image and uses Claude Opus to generate ordered 8-second scenes with consistent visual description and movement guidance | Contextual sticky note |
| Video Generation block note: writes scenes to Video Data, generates VEO3 clip per scene, polls until terminal status, uploads clips, and updates per-scene row with links/errors | Contextual sticky note |
| Regeneration of the video | Contextual sticky note |

## Additional implementation cautions
- The workflow contains a few sequencing and identifier assumptions that should be reviewed before production use:
  - source-row failure updates depend on `ID` being written back,
  - parent ID extraction from `VIDEO ID` is fragile,
  - some wait-node units are ambiguous in the export,
  - `If` false branch currently points to an image-related error node rather than a stitching-specific failure node.
- There are **no Sub-Workflow invocation nodes** in this design, but there is an **Execute Workflow Trigger** entry point, so this workflow can act as a callable child workflow.