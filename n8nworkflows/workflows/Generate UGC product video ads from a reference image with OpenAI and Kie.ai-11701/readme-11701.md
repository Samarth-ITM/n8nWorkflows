Generate UGC product video ads from a reference image with OpenAI and Kie.ai

https://n8nworkflows.xyz/workflows/generate-ugc-product-video-ads-from-a-reference-image-with-openai-and-kie-ai-11701


# Generate UGC product video ads from a reference image with OpenAI and Kie.ai

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate UGC product video ads from a reference image with OpenAI and Kie.ai  
**Purpose:** Automatically generate multiple UGC-style product video ads from a single reference product image. The workflow (1) analyzes the reference image with OpenAI Vision, (2) generates structured scene prompts with an AI agent, (3) generates keyframe images via Kie.ai, (4) generates videos via Kie.ai Veo, (5) downloads and uploads results to Box and Google Drive, and (6) updates a Google Sheet log.

### 1.1 Entry & User Inputs
Manual trigger → set script text → set run parameters (image link, count, aspect ratio, model, requirements).

### 1.2 Image Analysis (Vision)
OpenAI image analysis extracts brand/colors/fonts/visual description (YAML output).

### 1.3 Prompt Generation (Agent + Structured Parsing)
LangChain agent uses the analysis + user constraints to output a JSON object with `scenes[]`, each containing an `image_prompt`, `video_prompt`, and generation settings.

### 1.4 Media Generation & Polling (Loops)
For each scene:
- Generate an image (keyframe) with Kie.ai → poll until `SUCCESS`
- Generate a video using that keyframe + prompt → poll until `msg == success`
- Fetch the final file(s)

### 1.5 Upload & Tracking
Uploads to Box and Google Drive, aggregates Drive links, fetches matching sheet rows, routes based on which speaker column matches the script, and updates the sheet with links.

---

## 2. Block-by-Block Analysis

### Block 1 — Stage 1: Setup and image analysis
**Overview:** Initializes the run configuration (script, count, aspect ratio/model, requirements, reference image URL) and performs a vision analysis of the image.  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Video script
- Setup (Image & Video settings)
- Analyze image

#### Node: When clicking ‘Execute workflow’
- **Type/role:** Manual Trigger (`n8n-nodes-base.manualTrigger`); entry point for testing/manual runs.
- **Config choices:** No parameters.
- **Outputs:** to **Video script**.
- **Edge cases:** None (manual start only).

#### Node: Video script
- **Type/role:** Set node; stores the ad script string.
- **Key fields created:**
  - `Script`: `"It’s airy, pre-mixed, ... it actually feels easy."`
- **Outputs:** to **Setup (Image & Video settings)**.
- **Edge cases:** Quotation marks inside the string can affect downstream prompt escaping if later reused without sanitization.

#### Node: Setup (Image & Video settings)
- **Type/role:** Set node; central configuration.
- **Key fields created/used:**
  - `googledrive link reference image`: Google Drive share link
  - `how many videos?`: `"7"` (string, but conceptually integer)
  - `dialouge`: expression that injects `Script` into a strict instruction block
  - `model`: `"veo3_fast"`
  - `aspec_ratio`: `"vertical"` (note spelling)
  - `any special requirements?`: long constraints (age 40–60, gender/ethnicity probabilities, no subtitles, scene distribution)
- **Outputs:** to **Analyze image**.
- **Edge cases/failures:**
  - `how many videos?` is a string; if the agent interprets it incorrectly, scene count mismatches can happen.
  - If the Drive link format differs, the later regex extraction can fail (image download URL becomes invalid).

#### Node: Analyze image
- **Type/role:** OpenAI Vision analysis (`@n8n/n8n-nodes-langchain.openAi`, resource=image, operation=analyze).
- **Config choices:**
  - Model: `chatgpt-4o-latest`
  - Prompt requests **YAML only** with fields: `brand_name`, `color_scheme[] (hex/name)`, `font_style`, `visual_description`
  - `simplify: false` (returns full OpenAI response structure)
  - `imageUrls`: built from Google Drive “uc?export=download&id=...”
    - Extracts file id with regex:
      - `match(/https:\/\/drive\.google\.com\/file\/d\/([A-Za-z0-9_-]+)/)?.[1]`
- **Outputs:** to **Generate UGC prompts with AI agent1**
- **Edge cases/failures:**
  - Invalid/permissioned Drive file → OpenAI cannot fetch image (401/403/404).
  - If OpenAI returns non-YAML despite instruction, downstream agent still uses it as text (could degrade prompt quality).
  - Model availability changes (`chatgpt-4o-latest`) could require updating.

---

### Block 2 — Stage 2: Prompt generation
**Overview:** Uses a LangChain agent with a strict system prompt and a structured output parser to generate exactly N scenes of UGC prompts.  
**Nodes involved:**  
- OpenAI Chat Model
- Structured Output Parser
- Validate prompt generation
- Generate UGC prompts with AI agent1

#### Node: OpenAI Chat Model
- **Type/role:** Chat LLM connector (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) used by the agent.
- **Config choices:**
  - Model: `gpt-4.1`
- **Connections:** Provides `ai_languageModel` input to **Generate UGC prompts with AI agent1**.
- **Edge cases:** Credential/model access errors, rate limits.

#### Node: Structured Output Parser
- **Type/role:** Structured output enforcement (`@n8n/n8n-nodes-langchain.outputParserStructured`).
- **Config choices:**
  - Provides a JSON schema example requiring:
    - root: `{ "scenes": [ ... ] }`
    - Each scene has `image_prompt`, `video_prompt`, `aspect_ratio_video`, `aspect_ratio_image`, `model`
- **Connections:** Provides `ai_outputParser` to **Generate UGC prompts with AI agent1**.
- **Edge cases:** If the agent’s output cannot be parsed to the expected structure, the node/agent run will fail.

#### Node: Validate prompt generation
- **Type/role:** Think tool (`@n8n/n8n-nodes-langchain.toolThink`) used by the agent for self-checking.
- **Connections:** Provided as `ai_tool` to **Generate UGC prompts with AI agent1**.
- **Edge cases:** None functional; only impacts reasoning/self-validation.

#### Node: Generate UGC prompts with AI agent1
- **Type/role:** LangChain agent (`@n8n/n8n-nodes-langchain.agent`) that produces structured prompts.
- **Config choices:**
  - Uses a large **system message** defining UGC style, camera keywords, dialogue rules, no double-quotes, diversity constraints, and **scene count rule** (“exactly that many scenes”).
  - User prompt injects:
    - Count: `Setup...['how many videos?']`
    - Image analysis text: `{{ $json.choices[0].message.content }}`
    - Aspect ratio/model/script/special requirements from Setup
  - `hasOutputParser: true` → structured parser is mandatory.
- **Inputs/Outputs:**
  - Input from **Analyze image**
  - Outputs to **Process image prompts1** as `output.scenes`
- **Edge cases/failures:**
  - If `choices[0].message.content` path changes (OpenAI node version/format), expression breaks.
  - Scene count mismatch: requested “7” must become 7 scenes; if the agent outputs fewer/more, parsing may still succeed but business logic breaks expectations.
  - Prompts include forbidden double quotes; later HTTP escaping attempts to mitigate but is not perfect.

---

### Block 3 — Stage 3: Media generation and polling (Image + Video loops)
**Overview:** Iterates over scenes, generates keyframe images via Kie.ai, polls until ready, then generates videos via Kie.ai Veo, polls until ready, then retrieves downloadable files.  
**Nodes involved:**  
- Process image prompts1  
- Call image generation API1  
- Wait for image generation1  
- Retrieve generated image1  
- Check if image is ready1  
- Call video generation API1  
- Wait for video generation1  
- Retrieve generated video1  
- Check if video is ready1  
- Process video files1  
- Retrieve video file list1

#### Node: Process image prompts1
- **Type/role:** Split Out (`n8n-nodes-base.splitOut`) to iterate scenes.
- **Config choices:** Splits `output.scenes` into separate items (one per scene).
- **Outputs:** to **Call image generation API1**
- **Edge cases:** If `output.scenes` missing/not array → node fails.

#### Node: Call image generation API1
- **Type/role:** HTTP Request to Kie.ai image generation.
- **Endpoint:** `POST https://api.kie.ai/api/v1/gpt4o-image/generate`
- **Auth:** Generic Credential → HTTP Bearer Auth (API key required).
- **Body highlights:**
  - `filesUrl`: array with reference image download URL (Drive `uc?export=download&id=...`)
  - `prompt`: uses `$json.image_prompt` with escaping:
    - replaces `\"` and converts newlines to literal `\\n`
  - `size`: `$json.aspect_ratio_image` (expected `2:3` or `3:2`)
  - `nVariants: 1`
- **Batching:** batchSize=1, interval=5000ms (rate limiting).
- **Outputs:** to **Wait for image generation1**
- **Edge cases/failures:**
  - Invalid bearer token → 401
  - Kie.ai errors/timeouts
  - Prompt escaping may still break JSON if unexpected characters appear.

#### Node: Wait for image generation1
- **Type/role:** Wait node; delays before polling.
- **Config:** unit=minutes (amount not shown; default behavior in UI is important).
- **Outputs:** to **Retrieve generated image1**
- **Edge cases:** Too-short wait → excessive polling; too-long → slow runs.

#### Node: Retrieve generated image1
- **Type/role:** HTTP Request poll for image status.
- **Endpoint:** `https://api.kie.ai/api/v1/gpt4o-image/record-info`
- **Auth:** Bearer.
- **Query:** `taskId = {{ $('Call image generation API1').item.json.data.taskId }}`
- **Outputs:** to **Check if image is ready1**
- **Edge cases:** If `taskId` missing (API failed), expression fails.

#### Node: Check if image is ready1
- **Type/role:** IF node; loops until image generation is complete.
- **Condition:** `$json.data.status == "SUCCESS"`
- **True output:** to **Call video generation API1**
- **False output:** to **Wait for image generation1** (poll loop)
- **Edge cases:**
  - If API uses other terminal statuses (FAILED, ERROR), this loops forever unless additional conditions are added.

#### Node: Call video generation API1
- **Type/role:** HTTP Request to Kie.ai Veo video generation.
- **Endpoint:** `POST https://api.kie.ai/api/v1/veo/generate`
- **Auth:** Bearer.
- **Body (raw JSON):**
  - `prompt`: from `video_prompt` (taken from the current scene item via `$('Process image prompts1').item...`) with newline and quote escaping
  - `model`: scene model (`veo3` or `veo3_fast`)
  - `aspectRatio`: `9:16` or `16:9`
  - `imageUrls`: first result URL from the generated image
- **Outputs:** to **Wait for video generation1**
- **Edge cases:**
  - If `Retrieve generated image1` returns empty `resultUrls`, video generation fails.
  - Some APIs expect `imageUrls` as array, but here it’s sent as a string (depends on Kie.ai spec).

#### Node: Wait for video generation1
- **Type/role:** Wait node for video completion.
- **Config:** `amount: 800` (unit not shown in JSON; UI determines whether seconds/minutes).
- **Outputs:** to **Retrieve generated video1**
- **Edge cases:** Same polling concerns as image.

#### Node: Retrieve generated video1
- **Type/role:** HTTP Request poll for video status.
- **Endpoint:** `https://api.kie.ai/api/v1/veo/record-info`
- **Query:** `taskId = {{ $json.data.taskId }}` (from prior video generation call chain)
- **Outputs:** to **Check if video is ready1**
- **Edge cases:** If upstream item structure differs, `data.taskId` missing.

#### Node: Check if video is ready1
- **Type/role:** IF node; loops until video is ready.
- **Condition:** `$json.msg == "success"`
- **True output:** to **Process video files1**
- **False output:** to **Wait for video generation1**
- **Edge cases:**
  - Terminal failures are not handled; could loop indefinitely on “failed” messages.

#### Node: Process video files1
- **Type/role:** Split Out; attempts to split file URLs.
- **Field:** `data.response.resultUrls[0]`
- **Outputs:** to **Retrieve video file list1**
- **Edge cases/concerns:**
  - Splitting a single URL string is unusual; if `resultUrls` is an array, you typically split `data.response.resultUrls`. As configured, it may not iterate multiple URLs correctly.

#### Node: Retrieve video file list1
- **Type/role:** HTTP Request to download the actual video file.
- **URL:** `={{ $json['data.response.resultUrls[0]'] }}`
- **Outputs:** to both **Upload a file** (Box) and **Upload file** (Google Drive).
- **Edge cases:**
  - If the URL requires auth/signed URL expires, download fails.
  - Ensure “Download” / “Response format” in node is configured to return **binary** if Box upload expects binary (not visible in JSON; important in UI).

---

### Block 4 — Stage 4: Upload and tracking
**Overview:** Uploads the downloaded video to Box and Google Drive, aggregates Drive links, finds target rows in Google Sheets, routes by which speaker column matches the script, and updates sheet columns with video links.  
**Nodes involved:**  
- Upload a file (Box)
- Upload file (Google Drive)
- Put files together
- Fetch video generation logs1
- Route based on output type1
- Log image generation status1
- Log video generation status1
- Update final results1

#### Node: Upload a file (Box)
- **Type/role:** Box upload node (`n8n-nodes-base.box`).
- **Config choices:**
  - `binaryData: true`
  - `fileName: {{ $('Retrieve generated video1').item.json.data.taskId }}.mp4`
- **Inputs:** from **Retrieve video file list1**
- **Outputs:** Not connected further in this workflow.
- **Edge cases:**
  - Requires Box OAuth2 credentials (not included in JSON).
  - If incoming data is not binary, upload fails.

#### Node: Upload file (Google Drive)
- **Type/role:** Google Drive upload (`n8n-nodes-base.googleDrive`).
- **Config choices:**
  - Drive: “My Drive”
  - Folder: `1AyXY9wyFsiVbgdXfVB8Rhv1ksb64M_LO` (“8 sec Script Vids Bonsai”)
  - `inputDataFieldName: =data`
  - `name`: `={{ $json['data.response.resultUrls[0]'] }} ` (this likely sets the file name to the URL string; probably not intended)
- **Credentials:** Google Drive OAuth2 (`Jannik.hiller02`)
- **Outputs:** to **Put files together**
- **Edge cases:**
  - File naming: using a URL as filename can create messy names or be rejected.
  - If the node expects binary data in a specific field, ensure “Binary Property” is configured in UI.

#### Node: Put files together
- **Type/role:** Aggregate node; collects output links from Drive uploads.
- **Config:** Aggregates field `webViewLink` into an array `webViewLink[]`.
- **Outputs:** to **Fetch video generation logs1**
- **Edge cases:** If Drive upload doesn’t return `webViewLink`, aggregation results are empty.

#### Node: Fetch video generation logs1
- **Type/role:** Google Sheets node; fetches rows filtered by `Status = "for production"`.
- **Spreadsheet:** “UGC Ad Center - Bonsai Soil” (`17eJzHuXXpUr4i3QSk6S48gYToNdos1VLUOu18ndrDVU`)
- **Sheet tab:** “Orchid” (gid `107294555`)
- **Outputs:** to **Route based on output type1**
- **Edge cases:**
  - If multiple rows match “for production”, routing/updates may apply to multiple rows unexpectedly.

#### Node: Route based on output type1
- **Type/role:** Switch node; routes depending on which column matches the current script.
- **Rules (each output renamed):**
  - Output “Speaker 1” if `Video script.Script == row['Hook - Speaker 1']`
  - Output “Speaker 2” if `Video script.Script == row['Speaker 2']`
  - Output “Speaker 3” if `Video script.Script == row['Speaker 3']`
- **Outputs:**
  - Speaker 1 → **Log image generation status1**
  - Speaker 2 → **Log video generation status1**
  - Speaker 3 → **Update final results1**
- **Edge cases:**
  - If the script doesn’t match any column exactly (spacing/quotes), nothing routes and no updates occur.
  - If it matches multiple rows, multiple updates occur.

#### Node: Log image generation status1 (Google Sheets Update)
- **Type/role:** Updates rows where `Status` matches (matchingColumns = `["Status"]`).
- **Writes:**
  - `Status = "for production"` (unchanged)
  - `Video 1` = concatenation of `Put files together.webViewLink[0..6]` separated by blank lines
- **Edge cases:**
  - Updating by `Status` is unsafe if many rows share “for production”; it can overwrite multiple rows.
  - Assumes exactly 7 links exist; if fewer, expressions return `undefined`.

#### Node: Log video generation status1 (Google Sheets Update)
- Same pattern as above, but writes into `Video 2`.

#### Node: Update final results1 (Google Sheets Update)
- Same pattern, writes into `Video 3`.
- **Schema note:** The provided schema includes duplicate “Feedback/ Notes” entries; may cause mapping ambiguity depending on n8n behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry point | — | Video script |  |
| Video script | set | Define ad script text | When clicking ‘Execute workflow’ | Setup (Image & Video settings) | ## Stage 1: Setup and image analysis… |
| Setup (Image & Video settings) | set | Define run settings (image link, count, constraints) | Video script | Analyze image | ## Stage 1: Setup and image analysis… |
| Analyze image | openAi (image analyze) | Vision analysis of reference image | Setup (Image & Video settings) | Generate UGC prompts with AI agent1 | ## Stage 1: Setup and image analysis… |
| OpenAI Chat Model | lmChatOpenAi | LLM for agent prompt generation | — | Generate UGC prompts with AI agent1 (ai_languageModel) | ## Stage 2: Prompt generation… |
| Structured Output Parser | outputParserStructured | Enforce JSON structure (`scenes[]`) | — | Generate UGC prompts with AI agent1 (ai_outputParser) | ## Stage 2: Prompt generation… |
| Validate prompt generation | toolThink | Agent self-check tool | — | Generate UGC prompts with AI agent1 (ai_tool) | ## Stage 2: Prompt generation… |
| Generate UGC prompts with AI agent1 | agent | Generate UGC prompts per scene | Analyze image (+ AI model/parser/tool) | Process image prompts1 | ## Stage 2: Prompt generation… |
| Process image prompts1 | splitOut | Iterate over scenes | Generate UGC prompts with AI agent1 | Call image generation API1 | ## Image Generation Loop… |
| Call image generation API1 | httpRequest | Kie.ai keyframe image generation | Process image prompts1 | Wait for image generation1 | ## Image Generation Loop… |
| Wait for image generation1 | wait | Delay between poll attempts | Call image generation API1 / Check if image is ready1 (false) | Retrieve generated image1 | ## Image Generation Loop… |
| Retrieve generated image1 | httpRequest | Poll Kie.ai for image status/result URL | Wait for image generation1 | Check if image is ready1 | ## Image Generation Loop… |
| Check if image is ready1 | if | Loop control (image SUCCESS?) | Retrieve generated image1 | Call video generation API1 / Wait for image generation1 | ## Image Generation Loop… |
| Call video generation API1 | httpRequest | Kie.ai Veo video generation | Check if image is ready1 (true) | Wait for video generation1 | ## Video Generation Loop… |
| Wait for video generation1 | wait | Delay between video poll attempts | Call video generation API1 / Check if video is ready1 (false) | Retrieve generated video1 | ## Video Generation Loop… |
| Retrieve generated video1 | httpRequest | Poll Kie.ai for video status/result URLs | Wait for video generation1 | Check if video is ready1 | ## Video Generation Loop… |
| Check if video is ready1 | if | Loop control (video ready?) | Retrieve generated video1 | Process video files1 / Wait for video generation1 | ## Video Generation Loop… |
| Process video files1 | splitOut | Prepare iterating over result URLs | Check if video is ready1 (true) | Retrieve video file list1 | ## Video Generation Loop… |
| Retrieve video file list1 | httpRequest | Download final video file | Process video files1 | Upload a file / Upload file | ## Video Generation Loop… |
| Upload a file | box | Upload MP4 to Box | Retrieve video file list1 | — | ## Stage 4: Upload and tracking… |
| Upload file | googleDrive | Upload MP4 to Google Drive folder | Retrieve video file list1 | Put files together | ## Stage 4: Upload and tracking… |
| Put files together | aggregate | Collect Drive `webViewLink`s | Upload file | Fetch video generation logs1 | ## Stage 4: Upload and tracking… |
| Fetch video generation logs1 | googleSheets | Find target rows by Status filter | Put files together | Route based on output type1 | ## Stage 4: Upload and tracking… |
| Route based on output type1 | switch | Choose which column (speaker) to update | Fetch video generation logs1 | Log image generation status1 / Log video generation status1 / Update final results1 | ## Stage 4: Upload and tracking… |
| Log image generation status1 | googleSheets (update) | Update “Video 1” links | Route based on output type1 | — | ## Stage 4: Upload and tracking… |
| Log video generation status1 | googleSheets (update) | Update “Video 2” links | Route based on output type1 | — | ## Stage 4: Upload and tracking… |
| Update final results1 | googleSheets (update) | Update “Video 3” links | Route based on output type1 | — | ## Stage 4: Upload and tracking… |

Additional sticky note (covers general area):  
- **📋 Overview: Scale product videos with AI-generated UGC content** (applies broadly to the whole workflow)

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3) **Add Set node for the script**
   - Node: **Set**
   - Name: `Video script`
   - Add field:
     - `Script` (String) → your ad script text
   - Connect: Manual Trigger → Video script

4) **Add Set node for settings**
   - Node: **Set**
   - Name: `Setup (Image & Video settings)`
   - Add fields:
     - `googledrive link reference image` (String) → Drive share URL like `https://drive.google.com/file/d/<ID>/view?...`
     - `how many videos?` (String or Number; recommended Number) → e.g. `7`
     - `dialouge` (String, expression) → include strict instruction + `{{$json.Script}}`
     - `model` (String) → `veo3_fast` (or `veo3`)
     - `aspec_ratio` (String) → `vertical` or `horizontal`
     - `any special requirements?` (String) → your constraints
   - Connect: Video script → Setup

5) **Add OpenAI Vision analysis**
   - Node: **OpenAI** (LangChain OpenAI node with image analyze capability)
   - Name: `Analyze image`
   - Resource: **Image**
   - Operation: **Analyze**
   - Model: `chatgpt-4o-latest` (or equivalent vision-capable model)
   - Image URL expression: build a direct download link from Drive:
     - `https://drive.google.com/uc?export=download&id=<FILE_ID>`
     - Extract `<FILE_ID>` via regex from the Setup URL (as in workflow)
   - Prompt: request YAML-only output with brand/colors/fonts/description.
   - **Credentials:** OpenAI API key in n8n credentials.
   - Connect: Setup → Analyze image

6) **Add LLM, parser, and Think tool for the agent**
   - Node: **OpenAI Chat Model** (`lmChatOpenAi`)
     - Name: `OpenAI Chat Model`
     - Model: `gpt-4.1`
     - Credentials: OpenAI
   - Node: **Structured Output Parser**
     - Name: `Structured Output Parser`
     - Provide JSON schema example with root `scenes[]` and required fields.
   - Node: **Think Tool**
     - Name: `Validate prompt generation`

7) **Add Agent node to generate prompts**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `Generate UGC prompts with AI agent1`
   - System message: paste your UGC rules (camera keywords, no double quotes, scene count rule, dialogue exactness).
   - User message: include expressions referencing:
     - number of videos from Setup
     - analysis content from Analyze image output
     - aspect ratio, model, dialogue, special requirements from Setup
   - Enable structured output parsing and attach:
     - Language Model: `OpenAI Chat Model`
     - Output Parser: `Structured Output Parser`
     - Tool: `Validate prompt generation`
   - Connect: Analyze image → Agent

8) **Split scenes**
   - Node: **Split Out**
   - Name: `Process image prompts1`
   - Field to split out: `output.scenes`
   - Connect: Agent → Split Out

9) **Image generation (Kie.ai)**
   - Node: **HTTP Request**
   - Name: `Call image generation API1`
   - Method: POST
   - URL: `https://api.kie.ai/api/v1/gpt4o-image/generate`
   - Auth: **HTTP Bearer Auth** (create credential “Kie.ai” with API key)
   - Send JSON body with:
     - reference image URL (Drive direct download)
     - `prompt` = current item `image_prompt` (escape newlines/quotes)
     - `size` = `aspect_ratio_image`
     - `nVariants = 1`
   - (Optional) enable batching/rate limit similar to JSON.
   - Connect: Split Out → Call image generation API1

10) **Polling loop for image**
   - Add **Wait** node `Wait for image generation1` (e.g., 1–2 minutes)
   - Add **HTTP Request** node `Retrieve generated image1`
     - GET/POST as required by Kie.ai
     - URL: `https://api.kie.ai/api/v1/gpt4o-image/record-info`
     - Query: `taskId = {{$('Call image generation API1').item.json.data.taskId}}`
     - Bearer auth
   - Add **IF** node `Check if image is ready1`
     - Condition: `{{$json.data.status}} equals SUCCESS`
     - True → proceed
     - False → back to Wait
   - Connect: Call image → Wait → Retrieve → IF, and IF(false) → Wait

11) **Video generation (Kie.ai Veo)**
   - Add **HTTP Request** node `Call video generation API1`
     - POST `https://api.kie.ai/api/v1/veo/generate`
     - Bearer auth
     - Raw JSON body:
       - `prompt` from `video_prompt`
       - `model` from scene
       - `aspectRatio` from scene
       - `imageUrls` from retrieved image result URL
   - Connect: IF(true) → Call video generation API1

12) **Polling loop for video**
   - Add **Wait** node `Wait for video generation1` (choose a realistic duration)
   - Add **HTTP Request** node `Retrieve generated video1`
     - GET `https://api.kie.ai/api/v1/veo/record-info`
     - Query `taskId` from the video generation response
     - Bearer auth
   - Add **IF** node `Check if video is ready1`
     - Condition: `{{$json.msg}} equals success`
     - False → back to Wait
     - True → continue
   - Connect: Call video → Wait → Retrieve → IF, and IF(false) → Wait

13) **Download the final video file**
   - Prefer this setup (more robust than the provided split field):
     - Add **Split Out** on `data.response.resultUrls` (array)
   - Add **HTTP Request** `Retrieve video file list1`
     - URL: current item URL
     - Set response to **File/Binary** (so uploads work)
   - Connect: IF(true) → Split Out → Retrieve video file

14) **Upload to Box**
   - Add **Box** node `Upload a file`
   - Configure upload from binary property produced by the download node
   - File name: `<taskId>.mp4` (or something meaningful)
   - Set Box OAuth2 credentials

15) **Upload to Google Drive**
   - Add **Google Drive** node `Upload file`
   - Select Drive + Folder
   - Upload binary
   - Set a clean file name (recommended: `<taskId>.mp4` rather than URL)
   - Set Google Drive OAuth2 credentials

16) **Aggregate Drive links**
   - Add **Aggregate** node `Put files together`
   - Aggregate field: `webViewLink`
   - Connect: Google Drive upload → Aggregate

17) **Fetch rows to update in Google Sheets**
   - Add **Google Sheets** node `Fetch video generation logs1`
   - Operation: Read/Get Many with filter:
     - Column `Status` equals `"for production"`
   - Set credentials and select the document + sheet tab.

18) **Route to the correct update path**
   - Add **Switch** node `Route based on output type1`
   - Rules compare your `Video script.Script` to one of the columns in the fetched sheet row:
     - `Hook - Speaker 1`, `Speaker 2`, `Speaker 3`
   - Connect: Fetch rows → Switch

19) **Update the sheet with video links**
   - Add 3 **Google Sheets Update** nodes:
     - `Log image generation status1` → writes `Video 1`
     - `Log video generation status1` → writes `Video 2`
     - `Update final results1` → writes `Video 3`
   - Each should update the correct row (recommended improvement):
     - Match by a unique row identifier (e.g., `row_number`), not by `Status`.
   - Connect: Switch outputs → respective update nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Scale product videos with AI-generated UGC content… analyzes a reference image… calls external APIs… uploads… logs results to Google Sheets” | Sticky note: 📋 Overview: Scale product videos with AI-generated UGC content |
| Image Generation Loop: “Generates keyframe images using Kie.ai API. Each image serves as the first frame for video generation.” | Sticky note: Image Generation Loop |
| Video Generation Loop: “Generates AI videos using the keyframes and dialogue. Downloads final video files.” | Sticky note: Video Generation Loop |
| Stages 1–4 descriptions (setup/analysis → prompt generation → media generation/polling → upload/tracking) | Sticky notes: Stage 1/2/3/4 |

If you want, I can also propose “hardening” changes (fail-fast on FAILED statuses, correct splitting of resultUrls, safer Google Sheets row matching, and consistent binary handling for uploads).