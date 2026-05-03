Create tutorial videos from documentation URLs with Claude, Google TTS, Remotion and YouTube

https://n8nworkflows.xyz/workflows/create-tutorial-videos-from-documentation-urls-with-claude--google-tts--remotion-and-youtube-12515


# Create tutorial videos from documentation URLs with Claude, Google TTS, Remotion and YouTube

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow turns a **documentation URL** into a **fully rendered training video** by extracting page content, having an LLM generate a structured tutorial plan, generating narration instructions, assembling scene specs for Remotion, rendering a video via Remotion API, then distributing it (YouTube upload + Google Drive backup) and returning a JSON response to the caller.

**Primary use cases**
- Convert technical docs/blog posts into video lessons with code/terminal/diagram scenes.
- Automated video generation pipeline driven by a single webhook call.

### 1.1 Input Reception
Receives a POST request containing `documentationUrl` and starts the pipeline.

### 1.2 Content Extraction
Fetches the documentation HTML and parses it into cleaned text, headings, and code blocks.

### 1.3 AI Analysis & Planning
An agent node (labeled “Claude AI…”) uses a connected chat model to produce a JSON tutorial outline; then the outline is validated and enriched with metadata.

### 1.4 Audio & Visual Plan Generation
Creates narration segments (intro/section/outro) and scene definitions (code/terminal/diagram/text overlays).

### 1.5 Video Rendering & Distribution
Sends render request to Remotion, checks status, then uploads to YouTube and backs up to Google Drive, compiles a final response, and responds to the webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Exposes an HTTP endpoint that accepts a documentation URL and holds the response until the workflow finishes.

**Nodes involved**
- Webhook Trigger

#### Node: Webhook Trigger
- **Type / role:** `n8n-nodes-base.webhook` — entry point; receives POST requests.
- **Key configuration:**
  - **Path:** `create-video`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (response is sent by “Respond to Webhook” later).
- **Expected input payload:** JSON body like:
  ```json
  { "documentationUrl": "https://docs.example.com/guide" }
  ```
- **Outputs:** Forwards the request data to “Fetch Documentation”.
- **Edge cases / failures:**
  - Missing/invalid `body.documentationUrl` will break later expressions.
  - If workflow errors before Respond node, caller may get timeout or generic error.

---

### Block 2 — Content Extraction
**Overview:** Downloads the HTML for the given URL and extracts usable text structure for AI analysis.

**Nodes involved**
- Fetch Documentation
- Parse Documentation Content

#### Node: Fetch Documentation
- **Type / role:** `n8n-nodes-base.httpRequest` — HTTP GET to fetch the documentation HTML.
- **Key configuration:**
  - **URL expression:** `={{ $json.body.documentationUrl }}`
  - Response is stored under the node output (in this workflow, the parser expects it in `data`).
- **Connections:** Input from Webhook Trigger; output to Parse Documentation Content.
- **Edge cases / failures:**
  - 4xx/5xx, redirects, anti-bot protection, or HTML behind auth.
  - Large pages may exceed memory/time; parsing truncates later.
  - If response structure differs (e.g., not in `json.data`), parser will fail.

#### Node: Parse Documentation Content
- **Type / role:** `n8n-nodes-base.code` — transforms HTML into: `textContent`, `codeBlocks`, `headings`, plus samples.
- **Key logic / variables:**
  - Reads HTML from: `const html = $input.item.json.data;`
  - Removes `<script>`, `<style>`, `<nav>`, `<header>`, `<footer>` with regex.
  - Extracts plain text by stripping tags; collapses whitespace.
  - Extracts code blocks via regex looking for `<code class="language-...">`.
  - Extracts headings via regex `<h1..h6>`.
  - Truncation:
    - `textContent.substring(0, 8000)` (LLM input limit)
    - `rawHtml.substring(0, 5000)`
  - Sets `documentationUrl` from either `$input.item.json.url` or webhook body.
- **Outputs:** JSON with keys: `documentationUrl`, `textContent`, `codeBlocks`, `headings`, `rawHtml`.
- **Edge cases / failures:**
  - If Fetch Documentation returns HTML in a different field than `data`, `html` will be `undefined`.
  - Regex parsing is fragile with non-standard markup (e.g., code blocks not matching the `language-xxx` pattern).
  - HTML entities in code blocks may not be decoded.
  - Very long documents are truncated, possibly losing critical context.

---

### Block 3 — AI Analysis & Planning
**Overview:** Sends extracted content to an AI agent to produce a structured tutorial outline (as JSON), then validates and enriches it.

**Nodes involved**
- Mistral Cloud Chat Model
- Claude AI to analyze documentation
- Structure Tutorial Outline

#### Node: Mistral Cloud Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatMistralCloud` — provides the language model to the downstream agent.
- **Key configuration:** default options (no explicit model parameters shown).
- **Credentials:** Mistral Cloud API credential (`Mistral Cloud - test`).
- **Connections:** Supplies `ai_languageModel` to “Claude AI to analyze documentation”.
- **Edge cases / failures:**
  - Credential invalid/expired.
  - Model availability, rate limiting, or token limits.

#### Node: Claude AI to analyze documentation
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that should generate a tutorial outline JSON.
- **Key configuration:** options empty in JSON (important prompts/tools are not shown here; likely default agent behavior).
- **Inputs:**
  - **Main input:** from “Parse Documentation Content”
  - **Language model input:** from “Mistral Cloud Chat Model”
- **Outputs:** Expected to include a text field containing JSON (later parsed by Structure Tutorial Outline).
- **Edge cases / failures:**
  - Despite its name, it is not configured with an Anthropic/Claude model; it is wired to Mistral.
  - If agent returns non-JSON, JSON wrapped in markdown, or partial JSON, the next node fails.
  - Missing prompt/tooling configuration can yield inconsistent structure.

#### Node: Structure Tutorial Outline
- **Type / role:** `n8n-nodes-base.code` — parses AI output, validates schema, computes metadata.
- **Key logic:**
  - Reads AI response from:
    - `$input.item.json.response` **or**
    - `$input.item.json.content?.[0]?.text`
  - Strips markdown fences ```json / ``` and parses as JSON.
  - Validates: `sections` must exist and be an array.
  - Enriches:
    - `createdAt` ISO timestamp
    - `sourceUrl` from `$('Parse Documentation Content').item.json.documentationUrl`
    - `totalSections`
    - `totalDurationSeconds` and `totalDurationMinutes` (each section duration default `30`)
- **Connections:** From agent → to “Generate Narration Script”.
- **Edge cases / failures:**
  - Node throws explicit errors:
    - “No AI response received”
    - “Failed to parse AI response…”
    - “Invalid tutorial outline: missing sections array”
  - If sections have non-numeric `duration`, defaults may skew total duration.
  - Hard dependency on node name “Parse Documentation Content” via `$('...')`.

---

### Block 4 — Audio & Visual Plan Generation
**Overview:** Generates narration segments and then triggers an “agent” intended for Google TTS, finally builds scene specs for Remotion.

**Nodes involved**
- Generate Narration Script
- Google Gemini Chat Model
- Generate Audio with Google TTS1
- Create Visual Scenes

#### Node: Generate Narration Script
- **Type / role:** `n8n-nodes-base.code` — converts outline sections into timestamped narration segments with voice settings.
- **Key outputs:**
  - `videoTitle`
  - `narrationSegments`: `[intro, ...sectionSegments, outro]`
  - Each segment includes: `startTime`, `duration`, `narrationText`, `voiceSettings` (voice `en-US-Neural2-J`, speed/pitch 1.0)
- **Important behavior:**
  - Intro fixed duration: 10s at `startTime: 0`
  - Section start times computed as sum of prior section durations (but do **not** shift for intro duration)
  - Outro `startTime: outline.totalDurationSeconds` (also not including intro)
- **Connections:** Output to “Generate Audio with Google TTS1”.
- **Edge cases / failures:**
  - Timing alignment issue: intro overlaps section 0 unless Remotion composition accounts for it separately.
  - If outline lacks `videoTitle` or section fields (`narration`, `sectionTitle`), output may be incomplete.

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides a model to the TTS agent node.
- **Credentials:** Google Gemini (PaLM) API credential.
- **Connections:** Supplies `ai_languageModel` to “Generate Audio with Google TTS1”.
- **Edge cases / failures:** API limits, credential issues, regional restrictions.

#### Node: Generate Audio with Google TTS1
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — labeled as Google TTS generation step.
- **Configuration shown:** options empty; no explicit Google TTS API node exists in this JSON.
- **Inputs:**
  - Main input from “Generate Narration Script”
  - Language model input from “Google Gemini Chat Model”
- **Outputs:** Feeds “Create Visual Scenes”.
- **Important integration gap:**
  - No actual Google Cloud Text-to-Speech node is present; unless the agent is configured with tools to call TTS externally, this won’t generate real audio binaries.
- **Edge cases / failures:**
  - Agent may only “describe” audio generation instead of producing audio URLs/binaries unless tool integration exists.
  - If Remotion render expects audio assets, render may fail later.

#### Node: Create Visual Scenes
- **Type / role:** `n8n-nodes-base.code` — builds `scenes[]` for Remotion.
- **Key expressions / dependencies:**
  - Reads outline using: `const outline = $('Structure Tutorial Outline').item.json;` (hard dependency by node name)
  - Builds `scenes` per section with `visualType` switch:
    - `code` → code editor scene (theme dracula, typewriter effect)
    - `terminal` → terminal command animation
    - `diagram` → flowchart progressive reveal
    - `screen` → screen recording placeholder
    - default → text overlay with bullets
  - `detectLanguage()` heuristic for JS/Python/PHP/Java else plaintext
  - Sets video settings:
    - `resolution: 1920x1080`, `fps: 30`, `codec: h264`
    - `totalDuration: outline.totalDurationSeconds + 18` (intro/outro allowance)
- **Connections:** Output to “Render Video with Remotion”.
- **Edge cases / failures:**
  - If `visualType` is absent/unknown, defaults to text overlay.
  - `section.codeSnippet` / `terminalCommands` may be missing, yielding empty content.
  - Duration mismatch possible vs narration timing.

---

### Block 5 — Video Rendering & Distribution
**Overview:** Requests Remotion rendering, checks completion, then uploads/backs up and returns a summary response.

**Nodes involved**
- Render Video with Remotion
- Check Render Status
- Upload to Youtube
- Backup to Google Drive
- Compile Response
- Respond to Webhook

#### Node: Render Video with Remotion
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Remotion Render API.
- **Key configuration:**
  - **POST** `https://api.remotion.dev/v1/render`
  - **Headers:** `Content-Type: application/json`
  - **Auth:** generic header auth credential named `n8n` (details not shown; likely API key/bearer)
  - **Body parameters:**
    - `composition = "TutorialVideo"`
    - `inputProps = {{ JSON.stringify($json) }}` (sends scenes/settings as JSON string)
    - `codec = "h264"`
- **Connections:** Input from Create Visual Scenes; output to Check Render Status.
- **Edge cases / failures:**
  - Auth misconfiguration, invalid API key.
  - Remotion expects a different payload shape (often includes `serveUrl`, `composition`, `inputProps` etc.); this workflow may be incomplete depending on Remotion API requirements.
  - Render is usually asynchronous; immediate status “completed” is unlikely without polling.

#### Node: Check Render Status
- **Type / role:** `n8n-nodes-base.if` — gates next steps on render completion.
- **Condition:** `$json.status == "completed"` (case-insensitive config disabled but operation uses equals).
- **Connections:** If true → both Backup to Google Drive and Upload to Youtube.
- **Edge cases / failures:**
  - If Remotion returns `status: "queued"` or provides only `renderId`, this IF will route nowhere (no “false” branch configured), causing execution to end prematurely.
  - No polling/wait loop implemented.

#### Node: Upload to Youtube
- **Type / role:** `n8n-nodes-base.youTube` — uploads a video.
- **Key configuration shown:**
  - Operation: `video.upload`
  - Title: `"your_title"` (hardcoded placeholder)
  - CategoryId: expression-like string `=nji876tfcxswe34567yhgnmklp09` (looks invalid as a YouTube category ID)
  - RegionCode: `IN`
- **Credentials:** YouTube OAuth2 (`YouTube - test`).
- **Connections:** From Check Render Status; output to Compile Response.
- **Critical missing piece:** No binary/video file is mapped into this node. YouTube upload requires binary data (or a file reference depending on node capabilities/version).
- **Edge cases / failures:**
  - OAuth scope missing (`youtube.upload`).
  - Missing binary property → upload fails.
  - Invalid category ID → API error.
  - Privacy status not set (defaults apply).

#### Node: Backup to Google Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads a file to Drive as backup.
- **Key configuration:**
  - File name: `={{ $('Structure Tutorial Outline').item.json.videoTitle }}.mp4`
  - Drive: My Drive, Folder: root
- **Credentials:** Google Drive OAuth2.
- **Critical missing piece:** No binary/video file mapped here either; node typically needs a binary property containing file data.
- **Edge cases / failures:**
  - Missing binary input → upload fails.
  - Permission issues on Drive account or folder.

#### Node: Compile Response
- **Type / role:** `n8n-nodes-base.code` — builds final JSON response with metadata + YouTube/Drive info.
- **Key dependencies / expressions:**
  - `const outline = $('Structure Tutorial Outline').item.json;`
  - `const youtubeData = $('Upload to YouTube').item.json;` **(BUG: node name mismatch)**
  - `const driveData = $('Backup to Google Drive').item.json;`
  - Builds response:
    - `success`, `message`
    - `video` metadata (title/duration/sections/sourceUrl/createdAt)
    - `youtube` object with URL if `youtubeData.id`
    - `backup` object with Drive id/link
    - `metadata` (prerequisites, learningOutcomes)
- **Connections:** Output to Respond to Webhook.
- **Edge cases / failures:**
  - **Node reference bug:** actual node is named **“Upload to Youtube”** (lowercase “tube”), but code references **“Upload to YouTube”**. This will throw an error at runtime.
  - If YouTube/Drive nodes don’t run (render not completed), references may fail.

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — sends JSON response back to caller.
- **Key configuration:**
  - Respond with: JSON
  - Body: `={{ JSON.stringify($json, null, 2) }}`
- **Edge cases / failures:**
  - If earlier nodes throw, this node will never execute (caller waits until timeout unless error workflow is configured).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Entry point receiving documentation URL | — | Fetch Documentation | ## Content Extraction\nFetches documentation and extracts text, code blocks, and structure |
| Fetch Documentation | n8n-nodes-base.httpRequest | Download HTML from URL | Webhook Trigger | Parse Documentation Content | ## Content Extraction\nFetches documentation and extracts text, code blocks, and structure |
| Parse Documentation Content | n8n-nodes-base.code | Clean HTML; extract text, headings, code blocks | Fetch Documentation | Claude AI to analyze documentation | ## Content Extraction\nFetches documentation and extracts text, code blocks, and structure |
| Mistral Cloud Chat Model | @n8n/n8n-nodes-langchain.lmChatMistralCloud | LLM provider for analysis agent | — | Claude AI to analyze documentation (ai_languageModel) | ## AI Analysis & Planning\nAnalyzes content and creates structured tutorial outline |
| Claude AI to analyze documentation | @n8n/n8n-nodes-langchain.agent | Agent to generate tutorial outline JSON | Parse Documentation Content; Mistral Cloud Chat Model | Structure Tutorial Outline | ## AI Analysis & Planning\nAnalyzes content and creates structured tutorial outline |
| Structure Tutorial Outline | n8n-nodes-base.code | Parse/validate outline JSON; compute metadata | Claude AI to analyze documentation | Generate Narration Script | ## AI Analysis & Planning\nAnalyzes content and creates structured tutorial outline |
| Generate Narration Script | n8n-nodes-base.code | Build timestamped narration segments + voice settings | Structure Tutorial Outline | Generate Audio with Google TTS1 | ## Audio & Visual Generation\nGenerates narration audio and creates visual scenes with code/terminal animations |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for TTS agent | — | Generate Audio with Google TTS1 (ai_languageModel) | ## Audio & Visual Generation\nGenerates narration audio and creates visual scenes with code/terminal animations |
| Generate Audio with Google TTS1 | @n8n/n8n-nodes-langchain.agent | Intended TTS generation step (agent-based) | Generate Narration Script; Google Gemini Chat Model | Create Visual Scenes | ## Audio & Visual Generation\nGenerates narration audio and creates visual scenes with code/terminal animations |
| Create Visual Scenes | n8n-nodes-base.code | Generate Remotion scene specifications | Generate Audio with Google TTS1 | Render Video with Remotion | ## Audio & Visual Generation\nGenerates narration audio and creates visual scenes with code/terminal animations |
| Render Video with Remotion | n8n-nodes-base.httpRequest | Request Remotion render | Create Visual Scenes | Check Render Status | ## Video Rendering & Distribution\nRenders final video and uploads to YouTube and Google Drive |
| Check Render Status | n8n-nodes-base.if | Gate distribution on render completion | Render Video with Remotion | Backup to Google Drive; Upload to Youtube | ## Video Rendering & Distribution\nRenders final video and uploads to YouTube and Google Drive |
| Upload to Youtube | n8n-nodes-base.youTube | Upload rendered video to YouTube | Check Render Status | Compile Response | ## Video Rendering & Distribution\nRenders final video and uploads to YouTube and Google Drive |
| Backup to Google Drive | n8n-nodes-base.googleDrive | Upload backup copy to Drive | Check Render Status | Compile Response | ## Video Rendering & Distribution\nRenders final video and uploads to YouTube and Google Drive |
| Compile Response | n8n-nodes-base.code | Build final API response JSON | Backup to Google Drive; Upload to Youtube | Respond to Webhook | ## Video Rendering & Distribution\nRenders final video and uploads to YouTube and Google Drive |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return response to caller | Compile Response | — | ## Video Rendering & Distribution\nRenders final video and uploads to YouTube and Google Drive |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / setup notes | — | — | ## AI Documentation to Training Video Creator\n\nAutomatically transforms any technical documentation or blog post into a professional tutorial video with code highlighting, terminal animations, and AI-generated narration.\n\n### How it works\n\nThe workflow accepts a documentation URL via webhook, fetches and parses the HTML content to extract text, code blocks, and structure. Claude AI analyzes the content and creates a detailed tutorial outline with sections, narration scripts, and visual types. The workflow generates audio narration using Google Text-to-Speech, creates visual scenes with code highlights and terminal animations, and renders the complete video using Remotion. Finally, it uploads the video to YouTube and backs it up to Google Drive.\n\n### Setup steps\n\n1. Configure Claude AI credentials (Anthropic API key) in the \"Analyze with Claude AI\" node\n2. Set up Google Cloud Text-to-Speech API credentials with service account authentication\n3. Configure Remotion API credentials for video rendering (requires Remotion account)\n4. Add YouTube OAuth2 credentials for video uploads with upload permissions\n5. Add Google Drive OAuth2 credentials for backup storage\n6. Activate the workflow to enable the webhook endpoint\n7. Send POST requests to the webhook URL with JSON body: {\"documentationUrl\": \"https://docs.example.com/guide\"}\n8. Monitor executions and check YouTube for published videos\n\n### Customization\n\nModify narration voice settings in \"Generate Narration Script\" node. Adjust video resolution, FPS, and codec in \"Create Visual Scenes\" node. Customize code editor themes and terminal styles. Change YouTube category and privacy settings. Add custom intro/outro animations or branding overlays. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section label | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section label | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section label | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section label | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named: `AI Documentation to Training Video Creator`.

2) **Add “Webhook Trigger” (Webhook node)**
   - Method: `POST`
   - Path: `create-video`
   - Response: `Using 'Respond to Webhook' node`
   - Save and note the Test/Production URL.

3) **Add “Fetch Documentation” (HTTP Request)**
   - Method: `GET` (default)
   - URL: `{{$json.body.documentationUrl}}`
   - Response: ensure the node returns the HTML body in a JSON field (align with the parser; in this workflow the code expects `json.data`).
   - Connect: **Webhook Trigger → Fetch Documentation**

4) **Add “Parse Documentation Content” (Code node)**
   - Mode: `Run Once for Each Item`
   - Paste the parsing JS logic (clean HTML, extract text/code/headings, truncation).
   - Ensure it reads the correct field from the HTTP node output (adjust `const html = ...` if needed).
   - Connect: **Fetch Documentation → Parse Documentation Content**

5) **Add “Mistral Cloud Chat Model” (Mistral Chat Model)**
   - Create **Mistral Cloud API credentials** in n8n (API key).
   - Select the credential in the node.

6) **Add “Claude AI to analyze documentation” (AI Agent)**
   - In the agent node:
     - Provide a **system/prompt** (not present in JSON) that instructs it to output **strict JSON** with fields like:
       - `videoTitle`, `videoDescription`, `prerequisites[]`, `learningOutcomes[]`
       - `sections[]` with `sectionTitle`, `duration`, `narration`, `visualType`, plus optional `codeSnippet`, `terminalCommands[]`, `keyPoints[]`
     - Connect the **Language Model** input to **Mistral Cloud Chat Model**.
   - Connect main: **Parse Documentation Content → Claude AI to analyze documentation**

7) **Add “Structure Tutorial Outline” (Code node)**
   - Paste validation/parsing JS (strip ```json fences, `JSON.parse`, validate `sections[]`, add metadata).
   - Connect: **Claude AI to analyze documentation → Structure Tutorial Outline**

8) **Add “Generate Narration Script” (Code node)**
   - Paste narration segment creation JS (intro + per-section + outro).
   - Connect: **Structure Tutorial Outline → Generate Narration Script**

9) **Add “Google Gemini Chat Model” (Gemini Chat Model)**
   - Create **Google Gemini (PaLM) API credentials** and select them.

10) **Add “Generate Audio with Google TTS1” (AI Agent)**
   - If you truly want Google TTS audio generation, the most robust approach is to use a dedicated **Google Cloud Text-to-Speech** integration (HTTP request to Google TTS or an n8n node if available).  
   - If keeping the agent approach:
     - Configure agent tools to call TTS and return **audio file URLs or binary data** per narration segment (not shown in JSON, but required for real rendering).
     - Connect **Language Model** input from **Google Gemini Chat Model**.
   - Connect main: **Generate Narration Script → Generate Audio with Google TTS1**

11) **Add “Create Visual Scenes” (Code node)**
   - Paste scene building JS.
   - Ensure the output includes everything Remotion needs (scenes, duration, fps, codec, and references to audio assets if required).
   - Connect: **Generate Audio with Google TTS1 → Create Visual Scenes**

12) **Add “Render Video with Remotion” (HTTP Request)**
   - Method: `POST`
   - URL: `https://api.remotion.dev/v1/render`
   - Authentication: Header Auth (create an **HTTP Header Auth** credential holding Remotion API key/bearer token)
   - Header: `Content-Type: application/json`
   - Body fields:
     - `composition: TutorialVideo`
     - `inputProps: {{JSON.stringify($json)}}`
     - `codec: h264`
   - Connect: **Create Visual Scenes → Render Video with Remotion**

13) **Add “Check Render Status” (IF node)**
   - Condition: `{{$json.status}}` equals `completed`
   - Connect: **Render Video with Remotion → Check Render Status**
   - (Recommended fix) Add a polling loop (Wait + status-check request) if Remotion is async.

14) **Add “Upload to Youtube” (YouTube node)**
   - Operation: `Video → Upload`
   - Configure **YouTube OAuth2** credentials with upload scope/permissions.
   - Set Title dynamically (recommended): `{{$('Structure Tutorial Outline').item.json.videoTitle}}`
   - Provide the **binary video file** (this workflow JSON does not show how the MP4 is obtained from Remotion).
   - Connect from the **true** output of Check Render Status.

15) **Add “Backup to Google Drive” (Google Drive node)**
   - Operation: Upload (file)
   - OAuth2 credentials: Google Drive
   - File name: `{{$('Structure Tutorial Outline').item.json.videoTitle}}.mp4`
   - Folder: select destination
   - Provide the same **binary MP4**
   - Connect from the **true** output of Check Render Status.

16) **Add “Compile Response” (Code node)**
   - Build a final JSON response combining outline + YouTube + Drive outputs.
   - **Important:** Reference the exact node names (e.g., `$('Upload to Youtube')`, not `$('Upload to YouTube')`).
   - Connect inputs from both distribution nodes (as in the JSON) or use a Merge node for clean join.

17) **Add “Respond to Webhook” (Respond to Webhook node)**
   - Respond with: JSON
   - Body: `{{ JSON.stringify($json, null, 2) }}`
   - Connect: **Compile Response → Respond to Webhook**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Documentation to Training Video Creator” sticky note includes setup steps and customization ideas; however, the current workflow wiring uses **Mistral** for the “Claude…” agent and does not include a concrete Google TTS node. | Workflow canvas note (global) |
| Known runtime bug: Compile Response references `$('Upload to YouTube')` but actual node name is `Upload to Youtube`. | Fix required for successful execution |
| Remotion render completion is checked only once; most Remotion renders are async and require polling via renderId/status endpoint. | Reliability/production hardening |
| YouTube/Drive nodes require an MP4 binary input; workflow does not show download of rendered video from Remotion to binary. | Distribution integration gap |

