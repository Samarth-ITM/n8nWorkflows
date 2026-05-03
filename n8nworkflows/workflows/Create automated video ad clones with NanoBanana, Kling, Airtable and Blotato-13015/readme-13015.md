Create automated video ad clones with NanoBanana, Kling, Airtable and Blotato

https://n8nworkflows.xyz/workflows/create-automated-video-ad-clones-with-nanobanana--kling--airtable-and-blotato-13015


# Create automated video ad clones with NanoBanana, Kling, Airtable and Blotato

## 1. Workflow Overview

**Purpose:** This workflow automates the full creation of “cloned” video ads from a reference/original video, using:
- **Gemini (video analysis)** to extract structured scenes (SEALCam framework)
- **OpenAI (prompt structuring)** to produce deterministic prompts per scene
- **AtlasCloud (NanoBanana + Kling)** to generate **images** then **scene videos**
- **fal.ai (ffmpeg)** to **merge** scene videos, generate **music**, and **merge audio+video**
- **Blotato** to **upload and publish** the final media (YouTube in this configuration)
- **Airtable** as the production database/state machine (Status transitions)

### 1.1 Block Structure (Logical Blocks)
1. **Step 1 — Prompt creation (manual run)**
   - Fetch “Todo” project record, analyze original video, generate structured JSON prompts, write back to Airtable (main record + per-scene records).
2. **Step 2 — Image generation (scheduled)**
   - For the active project (“In progress”), loop scene records with “Prompt ready”, call NanoBanana image generation, poll, update scene records to “Image ready”.
3. **Step 3 — Video generation (scheduled)**
   - For the active project (“In progress”), fetch scene records “Image ready”, create consecutive start/end image pairs, call Kling video generation, poll, update scene records to “Video ready”.
4. **Step 4 — Merge videos (scheduled)**
   - For the active project (“In progress”), fetch all “Video ready” scene clips, merge them into a single video, update the main project record (“Done” + “Video clone” attachment).
5. **Step 5 — Music, final merge, publish (scheduled)**
   - For a project with status “Done”, generate music from prompt, merge music with video clone, store “Final Video”, upload to Blotato, create YouTube post, mark Airtable as “Published”.

---

## 2. Block-by-Block Analysis

### Block 1 — STEP 1: Create Prompt (Manual)
**Overview:** Pulls one Airtable record in “Todo”, analyzes its original video via Gemini into SEALCam YAML, then converts that analysis into strict JSON prompts via OpenAI; stores script/music prompt on the main record and creates per-scene records with prompts.

**Nodes involved:**
- Manual Trigger
- Setup Workflow
- Search Airtable for Record
- Check if Record Found
- Analyze video (Gemini)
- Generate Creative Assets JSON (LangChain Agent)
- OpenAI Chat Model
- Structured Output Parser
- Update Main Record
- Split Out Scenes
- Write Scene Data
- Sticky Note (STEP 1 - Create Prompt)
- Sticky Note1 (large overview note)

#### Manual Trigger
- **Type/role:** `manualTrigger` – starts Step 1 on demand.
- **Outputs:** to **Setup Workflow**.
- **Edge cases:** none (manual).

#### Setup Workflow
- **Type/role:** `set` – centralizes configuration and defaults.
- **Key config values set:**
  - `maxScenes` = 3 (used by Gemini prompt to cap scenes)
  - `musicStyle` = "" (currently unused downstream)
  - `language` = "en" (currently unused downstream)
  - Airtable identifiers/placeholders:
    - `airtableBaseId`, `airtableTableId`, `airtableTableName`
  - Status driving search:
    - `statusField` = "status" (note: later nodes use `{Status}` with capital S)
    - `statusValue` = "Todo"
  - `videoUrlField` = "video" (unused; workflow uses `Original Video`)
- **Outputs:** to **Search Airtable for Record**.
- **Failure modes:** wrong Airtable IDs/table name → Airtable node errors.

#### Search Airtable for Record
- **Type/role:** `airtable` search – find a single record with status “Todo”.
- **Important expressions:**
  - Base: `={{ $('Setup Workflow').item.json.airtableBaseId }}`
  - Table (configured “mode:id” but value uses table name expression):  
    `={{ $('Setup Workflow').item.json.airtableTableName }}`
  - Filter formula:
    `={{ '{' + statusField + '}="' + statusValue + '"' }}`
- **Outputs:** to **Check if Record Found**.
- **Edge cases / risks:**
  - **Field mismatch:** formula uses `statusField="status"` but Airtable schema elsewhere uses `Status`. If your Airtable column is `Status`, this search returns nothing.
  - **Table selection mismatch:** node is in `mode: id` but expression supplies a name; ensure correct mode/value.
  - Missing attachments (Original Video / Avatar Image / Product Image) can break later expressions if not guarded.

#### Check if Record Found
- **Type/role:** `if` – verifies `{{$json.id}}` exists.
- **True path:** proceeds to **Analyze video**.
- **False path:** no downstream connection (workflow ends silently).
- **Edge cases:** if Airtable returns empty, nothing happens (no alerting).

#### Analyze video (Gemini)
- **Type/role:** `googleGemini` (LangChain Google Gemini node) – video analysis into **strict YAML** with SEALCam scene breakdown.
- **Configuration:**
  - Model: `models/gemini-2.0-flash`
  - Resource/operation: `video` → `analyze`
  - Video URL: `={{ $json['Original Video'][0].url }}`
  - Prompt includes `maxScenes` from Setup Workflow.
- **Outputs:** to **Generate Creative Assets JSON**.
- **Failure modes:**
  - Missing `Original Video` attachment or empty array → expression error.
  - Gemini may return non-YAML or malformed YAML; downstream agent expects analysis text and then structured JSON conversion.
  - Video URL inaccessible (permissions, expired signed URL) → API failure.

#### Generate Creative Assets JSON
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` – uses LLM + output parser to generate a strict JSON object containing:
  - `script`
  - `music_prompt`
  - `scenes[]`: `{scene, start_image_prompt, video_prompt}`
- **Key inputs in prompt:**
  - Gemini analysis: `{{ $json.content.parts[0].text }}`
  - User request: `{{ $('Search Airtable for Record').item.json['My Description'] }}`
  - Avatar/Product image URLs:
    - `...['Avatar Image'][0].url`, `...['Product Image'][0].url`
- **System message:** extremely strict JSON schema mapping constraints (deterministic).
- **Connected AI components:**
  - **OpenAI Chat Model** via `ai_languageModel`
  - **Structured Output Parser** via `ai_outputParser`
- **Main output:** goes to **Update Main Record** and **Split Out Scenes**.
- **Failure modes:**
  - Missing Airtable fields/attachments → expression errors.
  - LLM returns invalid JSON → parser fails, node errors.
  - Scene count mismatch vs YAML scenes → can violate constraints and fail parsing.

#### OpenAI Chat Model
- **Type/role:** `lmChatOpenAi` – LLM provider for the agent.
- **Model:** `gpt-4.1-mini`.
- **Failure modes:** invalid API key, quota, timeouts.

#### Structured Output Parser
- **Type/role:** `outputParserStructured` – enforces JSON schema (manual JSON schema provided).
- **Key behavior:** `additionalProperties:false` and required keys means any deviation fails.
- **Failure modes:** minor formatting issues cause hard failures (e.g., extra keys, markdown fences, wrong property nesting).

#### Update Main Record
- **Type/role:** `airtable` update – updates the original “project” record.
- **Writes:**
  - `Script` = `={{ $json.output.script }}`
  - `Music Prompt` = `={{ $json.output.music_prompt }}`
  - `Status` = `"In progress"`
  - Match by Airtable record `id` from the initial search.
- **Outputs:** none (end of this branch).
- **Failure modes / risks:**
  - Uses `tableId` from Setup Workflow: `airtableTableId` must be correct.
  - Assumes fields `Script`, `Music Prompt`, and option `In progress` exist.

#### Split Out Scenes
- **Type/role:** `splitOut` – iterates over `output.scenes`.
- **Field to split:** `scenes, output.scenes` (note: unusual; effectively targets `output.scenes`).
- **Outputs:** each item contains one scene object → **Write Scene Data**.
- **Failure modes:** if `output.scenes` missing/empty → node errors or no items.

#### Write Scene Data
- **Type/role:** `airtable` create – creates one Airtable record per scene.
- **Writes per scene:**
  - `Project` = original project name: `{{ $('Search Airtable for Record').item.json.Project }}`
  - `Status` = `"Prompt ready"`
  - `scene_Title` = `={{ $json['output.scenes'].scene }}`
  - `start_image_prompt`, `video_prompt` from the split scene
- **Failure modes:**
  - Creates records in the **same table** as the main project (common pattern here: project record + scene records share table). If your base separates them, adjust.
  - If `Project` is empty, later “AND({Project}=...)” searches may fail to match.

---

### Block 2 — STEP 2: Generate Images (Scheduled)
**Overview:** On a schedule, finds a project in progress, then finds all its “Prompt ready” scene records and generates one image per scene using AtlasCloud NanoBanana. Updates scene records with attachment URL and status “Image ready”.

**Nodes involved:**
- Schedule
- Setup Workflow 2
- Find In Progress Record
- Check In Progress Found
- Find Prompt Ready Records
- Loop Over Records
- Generate Image POST
- Wait 3 Minutes
- Check Prediction Status GET
- Update Record with Image
- Sticky Note 2

#### Schedule
- **Type/role:** `scheduleTrigger` – periodic trigger (interval is configured but appears empty `{}`).
- **Output:** to **Setup Workflow 2**.
- **Edge cases:** Misconfigured interval may default unexpectedly; set a concrete interval.

#### Setup Workflow 2
- **Type/role:** `set` – config for Step 2.
- **Sets:** Airtable IDs + `kpi_atlascloud` API key placeholder.
- **Output:** to **Find In Progress Record**.

#### Find In Progress Record
- **Type/role:** Airtable search – `filterByFormula: {Status}="In progress"`, `limit:1`.
- **Output:** to **Check In Progress Found**.
- **Edge cases:** multiple projects in progress: only one processed per schedule tick.

#### Check In Progress Found
- **Type/role:** `if` – checks `$json.id` exists.
- **True:** proceeds to **Find Prompt Ready Records**.

#### Find Prompt Ready Records
- **Type/role:** Airtable search – finds scene records for the same project with `Status="Prompt ready"`.
- **Formula:**
  - `AND({Project}="<project>", {Status}="Prompt ready")`
- **Output:** to **Loop Over Records**.

#### Loop Over Records
- **Type/role:** `splitInBatches` – iterates records.
- **Connection note:** Workflow uses **output index 1** (the “next batch” output) to call **Generate Image POST**. This pattern can be correct in n8n, but is easy to miswire; confirm batch size and loop behavior.
- **Output:** to **Generate Image POST** and later back into loop via **Update Record with Image**.

#### Generate Image POST
- **Type/role:** `httpRequest` – calls AtlasCloud image generation endpoint.
- **Endpoint:** `POST https://api.atlascloud.ai/api/v1/model/generateImage`
- **Auth header:** `Authorization: Bearer {{kpi_atlascloud}}`
- **JSON body highlights:**
  - `model: "google/nano-banana-pro/edit"`
  - `aspect_ratio: "9:16"`
  - `images: [avatar_url, product_url]` (uses “In progress” record attachments)
  - `prompt: Loop Over Records item.start_image_prompt`
  - async mode enabled (`enable_sync_mode:false`)
- **Output:** to **Wait 3 Minutes**.
- **Failure modes:**
  - Missing Avatar/Product images on the “In progress” record.
  - AtlasCloud returns errors for invalid prompt or model.
  - API returns an id in a different path than `data.id`.

#### Wait 3 Minutes
- **Type/role:** `wait` – delays before polling prediction.
- **Output:** to **Check Prediction Status GET**.
- **Edge cases:** fixed wait can be too short/long; no retry loop if still processing.

#### Check Prediction Status GET
- **Type/role:** `httpRequest` – poll prediction.
- **URL:** `.../prediction/{{ $('Generate Image POST').item.json.data.id }}`
- **Output:** to **Update Record with Image**.
- **Failure modes:** prediction not complete → `outputs[0]` missing.

#### Update Record with Image
- **Type/role:** Airtable update – updates the scene record (the batch record).
- **Writes:**
  - `Status` = `"Image ready"`
  - `start_image` = `={{ $json.data.outputs[0] }}` (note: Airtable attachment fields typically expect array of `{url}` objects; this may still work if the node auto-converts, but often you must wrap.)
  - `id` matched from `Loop Over Records` item
- **Output:** loops back into **Loop Over Records** to continue batches.
- **Edge cases:** If `outputs[0]` is a string URL, prefer `[{url: outputs[0]}]` for Airtable attachments.

---

### Block 3 — STEP 3: Generate Videos (Scheduled)
**Overview:** Finds “Image ready” scenes, creates start/end image pairs, generates a 5s Kling clip per scene, polls, stores result in `video_scene`, marks each scene “Video ready”.

**Nodes involved:**
- Schedule Video Generation
- Setup Workflow 3
- Find In Progress Project
- Check Project Found
- Find Image Ready Records
- Create Image Pairs (Code)
- Loop Over Pairs
- Generate Video POST
- Wait 5 Minutes
- Check Video Status GET
- Update Record with Video
- Sticky Note 3

#### Schedule Video Generation
- **Type/role:** schedule trigger (minutes interval).
- **Output:** Setup Workflow 3.

#### Setup Workflow 3
- **Type/role:** set – Airtable IDs + `atlascloudApiKey`.
- **Output:** Find In Progress Project.

#### Find In Progress Project / Check Project Found
- **Type/role:** Airtable search + if check, same as Step 2.

#### Find Image Ready Records
- **Type/role:** Airtable search – for project + `Status="Image ready"`, sorted by `scene_Title`.
- **Output:** Create Image Pairs.

#### Create Image Pairs (Code)
- **Type/role:** `code` – transforms ordered scenes into consecutive pairs:
  - For each scene record: `currentImage` = this scene’s image URL
  - `nextImage` = next scene’s image URL (null for last)
  - keeps `recordId` and `video_prompt`
- **Output:** Loop Over Pairs.
- **Edge cases:**
  - If `start_image[0].url` missing, `currentImage` becomes null → Kling request fails.
  - Last scene has `nextImage:null`; Kling start-end-frame model may require an end frame. Consider fallback: reuse current image or omit end_image for last.

#### Loop Over Pairs
- **Type/role:** splitInBatches loop.
- **Output:** to Generate Video POST (again via output index 1 in this workflow).

#### Generate Video POST
- **Type/role:** httpRequest – AtlasCloud video generation.
- **Endpoint:** `POST https://api.atlascloud.ai/api/v1/model/generateVideo`
- **Body parameters:**
  - model `kwaivgi/kling-v2.1-i2v-pro/start-end-frame`
  - duration `5`
  - image = currentImage
  - end_image = nextImage
  - guidance_scale `0.5`
  - negative_prompt = `"example_value"` (placeholder; likely should be meaningful or removed)
  - prompt = video_prompt
- **Output:** Wait 5 Minutes.
- **Failure modes:** invalid/missing images, unsupported model params, long processing time.

#### Wait 5 Minutes / Check Video Status GET
- **Type/role:** wait then poll prediction endpoint with `data.id`.
- **Output:** Update Record with Video.
- **Edge cases:** if still running, output missing.

#### Update Record with Video
- **Type/role:** Airtable update.
- **Writes:**
  - `Status` = `"Video ready"`
  - `video_scene` = `={{ [ { url: $json.data.outputs[0] } ] }}`
- **Output:** loops to Loop Over Pairs to continue.

---

### Block 4 — STEP 4: Merge Videos (Scheduled)
**Overview:** Collects all scene clips (“Video ready”) for the active project and merges them using fal.ai ffmpeg merge-videos API. Updates the main record with the merged clip and sets status “Done”.

**Nodes involved:**
- Schedule Video Generation1
- Setup Workflow 4
- Find In Progress Project1
- Check Project Found1
- Find Image Ready Records1 (actually finds Video ready)
- Collect all videos (Code)
- Merge All Videos (HTTP)
- Wait
- Update Record with Video Clone
- Sticky Note  (STEP 4)

#### Setup Workflow 4
- **Type/role:** set – Airtable IDs + `falApiKey`.

#### Find In Progress Project1 / Check Project Found1
- **Type/role:** selects one “In progress” record (main project).

#### Find Image Ready Records1
- **Type/role:** Airtable search; despite name, formula is:
  - `Status="Video ready"` and same project, sorted by `scene_Title`.
- **Output:** Collect all videos.

#### Collect all videos (Code)
- **Type/role:** code – extracts first attachment URL from each `video_scene` and outputs a single item: `{ video_urls: [...] }`.
- **Edge cases:** missing `video_scene[0].url` reduces merge inputs; might produce empty array.

#### Merge All Videos
- **Type/role:** HTTP to fal.ai endpoint.
- **Endpoint:** `POST https://fal.run/fal-ai/ffmpeg-api/merge-videos`
- **Auth:** `Authorization: key {{falApiKey}}`
- **Body:** includes `video_urls` array and output format mp4.
- **Output:** to **Wait** (2 minutes).

#### Wait
- **Type/role:** wait to allow fal processing.
- **Output:** Update Record with Video Clone.

#### Update Record with Video Clone
- **Type/role:** Airtable update of the **main project record**.
- **Writes:**
  - `Status` = `"Done"`
  - `Video clone` attachment = `[{ url: $json.video.url, filename: ... }]`
- **Edge cases:** expects response shape `video.url` from fal; if fal returns different schema, adjust mapping.

---

### Block 5 — STEP 5: Create audio and Publish (Scheduled)
**Overview:** For a “Done” project, generate music (fal prompt-to-audio), store it, merge with merged video, store final video and mark “Ready to publish”, upload to Blotato, create YouTube post, then mark “Published”.

**Nodes involved:**
- Schedule Audio Generation
- Setup Workflow 5
- Find In audio Project
- Create audio
- Wait to audio
- Update Record with audio
- Merge audio and video
- Wait final video
- Update Record with Final Video
- Upload media (Blotato)
- Create post (Blotato)
- Update Status
- Sticky Note 1 (STEP 5)

#### Find In audio Project
- **Type/role:** Airtable search – `Status="Done"`, limit 1.
- **Risk:** only processes one done project per run.

#### Create audio
- **Type/role:** HTTP fal.ai `ace-step/prompt-to-audio`
- **Body:** uses `prompt: {{ $json['Music Prompt'] }}`, `instrumental:true`, `duration:30`.
- **Output:** Wait to audio.

#### Wait to audio / Update Record with audio
- **Type/role:** wait then Airtable update storing `Music File` attachment as `[{url: $json.audio.url}]`.
- **Critical issue:** Update Record with audio uses record id from `Find In Progress Project1` (Step 4 block) instead of the current “Done” record. If Step 4 didn’t run in the same execution context, this expression can be null/wrong.
  - Current mapping: `id: ={{ $('Find In Progress Project1').first().json.id }}`
  - Should likely be: `{{ $('Find In audio Project').first().json.id }}`

#### Merge audio and video
- **Type/role:** fal.ai ffmpeg merge audio+video
- **Body references:**  
  - `audio_url`: `{{ $json.fields['Music File'][0].url }}`
  - `video_url`: `{{ $json.fields['Video clone'][0].url }}`
- **Risk:** the incoming `$json` at this point is from **Update Record with audio** output, which may or may not include `.fields` in the expected structure depending on Airtable node output. Often Airtable update returns `fields`, but confirm.
- **Also:** If “Video clone” is not present on the same record being updated, this fails.

#### Update Record with Final Video
- **Type/role:** Airtable update – stores `Final Video` and sets `Status="Ready to publish"`.
- **Critical issue:** again references `Find In Progress Project1` for id; should reference the “Done” record.
- **Also requires:** Airtable fields `Title Video` and `Caption` exist later for posting.

#### Upload media / Create post / Update Status (Blotato publishing)
- **Upload media:** uses final video URL: `={{ $('Wait final video').first().json.video.url }}`
- **Create post:** YouTube platform, accountId `8047`, title from Airtable `Title Video`, caption from Airtable `Caption`, privacy `private`.
- **Update Status:** sets Airtable status to `Published` (also incorrectly references `Find In Progress Project1` id).
- **Failure modes:**
  - Blotato credentials invalid, accountId mismatch, YouTube permissions issues.
  - Airtable fields missing: `Caption`, `Title Video`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual entrypoint for Step 1 | — | Setup Workflow | ## STEP 1 - Create Prompt |
| Setup Workflow | n8n-nodes-base.set | Global config (Step 1) | Manual Trigger | Search Airtable for Record | ## STEP 1 - Create Prompt |
| Search Airtable for Record | n8n-nodes-base.airtable | Find 1 “Todo” project record | Setup Workflow | Check if Record Found | ## STEP 1 - Create Prompt |
| Check if Record Found | n8n-nodes-base.if | Guard: stop if no record | Search Airtable for Record | Analyze video (true path) | ## STEP 1 - Create Prompt |
| Analyze video | @n8n/n8n-nodes-langchain.googleGemini | Video→SEALCam YAML analysis | Check if Record Found | Generate Creative Assets JSON | ## STEP 1 - Create Prompt |
| Generate Creative Assets JSON | @n8n/n8n-nodes-langchain.agent | Convert analysis to strict JSON prompts | Analyze video | Update Main Record; Split Out Scenes | ## STEP 1 - Create Prompt |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for agent | — (AI input) | Generate Creative Assets JSON (AI) | ## STEP 1 - Create Prompt |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema | — (AI input) | Generate Creative Assets JSON (AI) | ## STEP 1 - Create Prompt |
| Update Main Record | n8n-nodes-base.airtable | Update project: script/music + In progress | Generate Creative Assets JSON | — | ## STEP 1 - Create Prompt |
| Split Out Scenes | n8n-nodes-base.splitOut | Iterate scenes array | Generate Creative Assets JSON | Write Scene Data | ## STEP 1 - Create Prompt |
| Write Scene Data | n8n-nodes-base.airtable | Create scene records with prompts | Split Out Scenes | — | ## STEP 1 - Create Prompt |
| Schedule | n8n-nodes-base.scheduleTrigger | Scheduled entrypoint (Step 2) | — | Setup Workflow 2 | ## STEP 2 - Generate Images |
| Setup Workflow 2 | n8n-nodes-base.set | Config for Step 2 | Schedule | Find In Progress Record | ## STEP 2 - Generate Images |
| Find In Progress Record | n8n-nodes-base.airtable | Find active project (In progress) | Setup Workflow 2 | Check In Progress Found | ## STEP 2 - Generate Images |
| Check In Progress Found | n8n-nodes-base.if | Guard | Find In Progress Record | Find Prompt Ready Records | ## STEP 2 - Generate Images |
| Find Prompt Ready Records | n8n-nodes-base.airtable | Find scene records Prompt ready | Check In Progress Found | Loop Over Records | ## STEP 2 - Generate Images |
| Loop Over Records | n8n-nodes-base.splitInBatches | Batch loop over scenes | Find Prompt Ready Records; Update Record with Image | Generate Image POST (loop output) | ## STEP 2 - Generate Images |
| Generate Image POST | n8n-nodes-base.httpRequest | AtlasCloud NanoBanana image generation | Loop Over Records | Wait 3 Minutes | ## STEP 2 - Generate Images |
| Wait 3 Minutes | n8n-nodes-base.wait | Delay before polling | Generate Image POST | Check Prediction Status GET | ## STEP 2 - Generate Images |
| Check Prediction Status GET | n8n-nodes-base.httpRequest | Poll AtlasCloud prediction | Wait 3 Minutes | Update Record with Image | ## STEP 2 - Generate Images |
| Update Record with Image | n8n-nodes-base.airtable | Store image + set Image ready | Check Prediction Status GET | Loop Over Records | ## STEP 2 - Generate Images |
| Schedule Video Generation | n8n-nodes-base.scheduleTrigger | Scheduled entrypoint (Step 3) | — | Setup Workflow 3 | ## STEP 3 - Generate Videos |
| Setup Workflow 3 | n8n-nodes-base.set | Config for Step 3 | Schedule Video Generation | Find In Progress Project | ## STEP 3 - Generate Videos |
| Find In Progress Project | n8n-nodes-base.airtable | Find active project | Setup Workflow 3 | Check Project Found | ## STEP 3 - Generate Videos |
| Check Project Found | n8n-nodes-base.if | Guard | Find In Progress Project | Find Image Ready Records | ## STEP 3 - Generate Videos |
| Find Image Ready Records | n8n-nodes-base.airtable | Get Image ready scenes | Check Project Found | Create Image Pairs | ## STEP 3 - Generate Videos |
| Create Image Pairs | n8n-nodes-base.code | Create consecutive (start,end) pairs | Find Image Ready Records | Loop Over Pairs | ## STEP 3 - Generate Videos |
| Loop Over Pairs | n8n-nodes-base.splitInBatches | Batch loop over pairs | Create Image Pairs; Update Record with Video | Generate Video POST (loop output) | ## STEP 3 - Generate Videos |
| Generate Video POST | n8n-nodes-base.httpRequest | AtlasCloud Kling i2v start/end | Loop Over Pairs | Wait 5 Minutes | ## STEP 3 - Generate Videos |
| Wait 5 Minutes | n8n-nodes-base.wait | Delay before polling | Generate Video POST | Check Video Status GET | ## STEP 3 - Generate Videos |
| Check Video Status GET | n8n-nodes-base.httpRequest | Poll AtlasCloud prediction | Wait 5 Minutes | Update Record with Video | ## STEP 3 - Generate Videos |
| Update Record with Video | n8n-nodes-base.airtable | Store scene clip + Video ready | Check Video Status GET | Loop Over Pairs | ## STEP 3 - Generate Videos |
| Schedule Video Generation1 | n8n-nodes-base.scheduleTrigger | Scheduled entrypoint (Step 4) | — | Setup Workflow 4 | ## STEP 4 - Merge Videos |
| Setup Workflow 4 | n8n-nodes-base.set | Config for Step 4 | Schedule Video Generation1 | Find In Progress Project1 | ## STEP 4 - Merge Videos |
| Find In Progress Project1 | n8n-nodes-base.airtable | Find active project | Setup Workflow 4 | Check Project Found1 | ## STEP 4 - Merge Videos |
| Check Project Found1 | n8n-nodes-base.if | Guard | Find In Progress Project1 | Find Image Ready Records1 | ## STEP 4 - Merge Videos |
| Find Image Ready Records1 | n8n-nodes-base.airtable | Get Video ready scenes | Check Project Found1 | Collect all videos | ## STEP 4 - Merge Videos |
| Collect all videos | n8n-nodes-base.code | Build `video_urls[]` | Find Image Ready Records1 | Merge All Videos | ## STEP 4 - Merge Videos |
| Merge All Videos | n8n-nodes-base.httpRequest | fal.ai ffmpeg merge-videos | Collect all videos | Wait | ## STEP 4 - Merge Videos |
| Wait | n8n-nodes-base.wait | Delay for fal processing | Merge All Videos | Update Record with Video Clone | ## STEP 4 - Merge Videos |
| Update Record with Video Clone | n8n-nodes-base.airtable | Save merged clip + Done | Wait | — | ## STEP 4 - Merge Videos |
| Schedule Audio Generation | n8n-nodes-base.scheduleTrigger | Scheduled entrypoint (Step 5) | — | Setup Workflow 5 | ## STEP 5 - Create audio and Publish |
| Setup Workflow 5 | n8n-nodes-base.set | Config for Step 5 | Schedule Audio Generation | Find In audio Project | ## STEP 5 - Create audio and Publish |
| Find In audio Project | n8n-nodes-base.airtable | Find Done project | Setup Workflow 5 | Create audio | ## STEP 5 - Create audio and Publish |
| Create audio | n8n-nodes-base.httpRequest | fal.ai prompt-to-audio | Find In audio Project | Wait to audio | ## STEP 5 - Create audio and Publish |
| Wait to audio | n8n-nodes-base.wait | Delay for audio | Create audio | Update Record with audio | ## STEP 5 - Create audio and Publish |
| Update Record with audio | n8n-nodes-base.airtable | Save Music File attachment | Wait to audio | Merge audio and video | ## STEP 5 - Create audio and Publish |
| Merge audio and video | n8n-nodes-base.httpRequest | fal.ai merge-audio-video | Update Record with audio | Wait final video | ## STEP 5 - Create audio and Publish |
| Wait final video | n8n-nodes-base.wait | Delay for final merge | Merge audio and video | Update Record with Final Video | ## STEP 5 - Create audio and Publish |
| Update Record with Final Video | n8n-nodes-base.airtable | Save Final Video + Ready to publish | Wait final video | Upload media | ## STEP 5 - Create audio and Publish |
| Upload media | @blotato/n8n-nodes-blotato.blotato | Upload final video to Blotato | Update Record with Final Video | Create post | ## STEP 5 - Create audio and Publish |
| Create post | @blotato/n8n-nodes-blotato.blotato | Create YouTube post | Upload media | Update Status | ## STEP 5 - Create audio and Publish |
| Update Status | n8n-nodes-base.airtable | Mark Published | Create post | — | ## STEP 5 - Create audio and Publish |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | (content preserved below in Notes) |
| Sticky Note 2 | n8n-nodes-base.stickyNote | Comment | — | — | (content preserved below in Notes) |
| Sticky Note 3 | n8n-nodes-base.stickyNote | Comment | — | — | (content preserved below in Notes) |
| Sticky Note  | n8n-nodes-base.stickyNote | Comment | — | — | (content preserved below in Notes) |
| Sticky Note 1 | n8n-nodes-base.stickyNote | Comment | — | — | (content preserved below in Notes) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Large workflow description + links | — | — | (content preserved below in Notes) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Airtable base + table**
   1. Create one table (as in the workflow) or split into Project/Scenes tables (requires adapting filters).
   2. Add fields (at minimum, matching this workflow’s usage):
      - `Status` (single select): Todo, In progress, Prompt ready, Image ready, Video ready, Done, Ready to publish, Published
      - `Project` (text)
      - `Original Video` (attachment)
      - `Avatar Image` (attachment)
      - `Product Image` (attachment)
      - `My Description` (long text)
      - `Script` (long text)
      - `Music Prompt` (long text)
      - `scene_Title` (text)
      - `start_image_prompt` (long text)
      - `start_image` (attachment)
      - `video_prompt` (long text)
      - `video_scene` (attachment)
      - `Video clone` (attachment)
      - `Music File` (attachment)
      - `Final Video` (attachment)
      - `Title Video` (text)
      - `Caption` (long text)

2. **Create credentials**
   1. **Airtable Personal Access Token** credential in n8n.
   2. **OpenAI API** credential (for `gpt-4.1-mini`).
   3. **Google Gemini(PaLM) API** credential (Gemini video analyze).
   4. **Blotato API** credential.
   5. Prepare API keys (stored as strings in Set nodes in this workflow):
      - **AtlasCloud** bearer token
      - **fal.ai** key

3. **STEP 1 flow (manual)**
   1. Add **Manual Trigger**.
   2. Add **Set** node “Setup Workflow”:
      - `maxScenes` (number, e.g. 3)
      - `airtableBaseId`, `airtableTableId`, `airtableTableName`
      - `statusField` (recommend `Status` to match formulas)
      - `statusValue` = `Todo`
   3. Add **Airtable → Search** “Search Airtable for Record”:
      - Base: from Setup Workflow
      - Table: from Setup Workflow (use consistent “id” vs “name” mode)
      - Limit 1
      - FilterByFormula: `{Status}="Todo"` (recommended)
   4. Add **IF** “Check if Record Found”: condition `{{$json.id}} exists`.
   5. Add **Google Gemini → Analyze Video**:
      - Model: `gemini-2.0-flash`
      - Video URL: `{{$json['Original Video'][0].url}}`
      - Prompt: SEALCam analyzer with `maxScenes`.
   6. Add **LangChain Agent** “Generate Creative Assets JSON”:
      - Provide the Gemini analysis text, “My Description”, avatar/product URLs.
      - Attach **OpenAI Chat Model** node as Language Model.
      - Attach **Structured Output Parser** with the schema requiring `script`, `music_prompt`, `scenes[]`.
   7. Add **Airtable → Update** “Update Main Record”:
      - Match record by `id` from the initial project record
      - Set `Script`, `Music Prompt`, `Status="In progress"`.
   8. Add **Split Out** “Split Out Scenes”: split `output.scenes`.
   9. Add **Airtable → Create** “Write Scene Data”:
      - Create new records for each scene:
        - `Project` = original project value
        - `Status` = `Prompt ready`
        - `scene_Title`, `start_image_prompt`, `video_prompt`.

4. **STEP 2 flow (scheduled images)**
   1. Add **Schedule Trigger** (set a clear interval, e.g. every 5 minutes).
   2. Add **Set** “Setup Workflow 2”: Airtable IDs + `atlascloudKey`.
   3. Add **Airtable → Search** “Find In Progress Record”: `{Status}="In progress"`, limit 1.
   4. Add **IF** “Check In Progress Found”: `$json.id exists`.
   5. Add **Airtable → Search** “Find Prompt Ready Records”:
      - Formula: `AND({Project}="<project>", {Status}="Prompt ready")`
   6. Add **Split In Batches** “Loop Over Records” to iterate scenes.
   7. Add **HTTP Request** “Generate Image POST”:
      - POST AtlasCloud `/generateImage`
      - Authorization header `Bearer <key>`
      - Body includes model nano-banana, 9:16, images[] from Avatar/Product, prompt from scene.
   8. Add **Wait** (e.g. 3 minutes).
   9. Add **HTTP Request** “Check Prediction Status GET”: poll `/prediction/{id}`.
   10. Add **Airtable → Update** “Update Record with Image”:
       - Status = `Image ready`
       - `start_image` attachment = `[{url: <output_url>}]`
       - loop back to Split In Batches.

5. **STEP 3 flow (scheduled videos)**
   1. Add **Schedule Trigger** (e.g. every 10 minutes).
   2. Add **Set** “Setup Workflow 3”: Airtable IDs + AtlasCloud key.
   3. Find `{Status}="In progress"` project.
   4. Find scenes `AND({Project}=..., {Status}="Image ready")`, sort by `scene_Title`.
   5. Add **Code** to create consecutive `{currentImage,nextImage}` pairs.
   6. Add **Split In Batches** loop.
   7. Add **HTTP Request** POST AtlasCloud `/generateVideo` with Kling start/end frame model.
   8. Wait, then poll `/prediction/{id}`.
   9. Update Airtable scene record: `Status="Video ready"`, `video_scene=[{url: ...}]`.

6. **STEP 4 flow (merge clips)**
   1. Add **Schedule Trigger**.
   2. Add **Set** “Setup Workflow 4”: Airtable IDs + fal key.
   3. Find `{Status}="In progress"` project.
   4. Find scenes `AND({Project}=..., {Status}="Video ready")`.
   5. Add **Code** to output one item with `video_urls[]`.
   6. Add **HTTP Request** POST `https://fal.run/fal-ai/ffmpeg-api/merge-videos` with `video_urls`.
   7. Wait.
   8. Update main project record: `Status="Done"`, `Video clone=[{url: ...}]`.

7. **STEP 5 flow (music + publish)**
   1. Add **Schedule Trigger**.
   2. Add **Set** “Setup Workflow 5”: Airtable IDs + fal key.
   3. Find `{Status}="Done"` project (limit 1).
   4. POST `https://fal.run/fal-ai/ace-step/prompt-to-audio` with `prompt={{Music Prompt}}`, duration 30.
   5. Wait.
   6. Update project: `Music File=[{url: audio.url}]`.
   7. POST `https://fal.run/fal-ai/ffmpeg-api/merge-audio-video` with `audio_url` + `video_url` (from Video clone).
   8. Wait.
   9. Update project: `Final Video=[{url: ...}]`, `Status="Ready to publish"`.
   10. Blotato “Upload media” using Final Video URL.
   11. Blotato “Create post”:
       - platform: youtube
       - title: `Title Video`
       - caption: `Caption`
       - media URLs: uploaded media url
       - privacy: private
   12. Update Airtable: `Status="Published"`.

**Important fix to apply while rebuilding:** In Step 5, do **not** reference `Find In Progress Project1` for record IDs; always use the record found in Step 5’s own search node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## STEP 1 - Create Prompt” | Sticky note covering Step 1 section |
| “## STEP 2 - Generate Images” | Sticky note covering Step 2 section |
| “## STEP 3 - Generate Videos” | Sticky note covering Step 3 section |
| “## STEP 4 - Merge Videos” | Sticky note covering Step 4 section |
| “## STEP 5 - Create audio and Publish” | Sticky note covering Step 5 section |
| Required accounts: Airtable, fal.ai, AtlasCloud, Blotato | From the large overview sticky note |
| AtlasCloud referral link: https://www.atlascloud.ai?ref=8QKPJE | From the large overview sticky note |
| Blotato referral link: https://blotato.com/?ref=firas | From the large overview sticky note |
| Documentation (Notion): https://automatisation.notion.site/Clone-Video-Ads-Factory-using-NanoBanana-Kling-and-Publish-with-Blotato-2f03d6550fd980a78193e996cca37600?source=copy_link | From the large overview sticky note |
| Airtable base template link: https://airtable.com/app8c479pcQMppB9Y/shrZud9lCfD0stHoE | From the large overview sticky note |

**Disclaimer (as provided):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.