Generate images on Telegram from text and voice using Grok Imagine via Kie AI

https://n8nworkflows.xyz/workflows/generate-images-on-telegram-from-text-and-voice-using-grok-imagine-via-kie-ai-13367


# Generate images on Telegram from text and voice using Grok Imagine via Kie AI

## 1. Workflow Overview

**Title:** Generate images on Telegram from text and voice using Grok Imagine via Kie AI  
**Workflow name (in JSON):** *Image Grok Agent with Telegram*  

This workflow implements an **AI-powered Telegram bot** that accepts **text messages, voice notes, and uploaded images**, then uses an **LLM orchestrator (Grok 4.1 Fast via OpenRouter)** to decide whether to run:
- **Text-to-image** generation (Grok Imagine via **Kie AI**), or
- **Image-to-image** transformation (Grok Imagine via **Kie AI**) using an uploaded image URL.

It also supports:
- **User authorization** by Telegram user ID (simple allow-list),
- **Voice transcription** (OpenAI Whisper),
- **Image upload to public hosting** via FTP (e.g., BunnyCDN) to produce a URL usable by Kie AI,
- **Asynchronous job execution** using Kie AI task IDs + **Wait (webhook resume)** + job status retrieval.

### 1.1 Input Reception & Authorization (Telegram)
Receives Telegram updates and blocks unauthorized users before any costly processing.

### 1.2 Input Normalization (Text vs Voice vs Photo)
Routes incoming Telegram messages by modality:
- Text → direct prompt
- Voice → fetch audio → transcribe → prompt
- Photo → fetch file → upload to FTP → build public image URL (+ optional caption prompt)

### 1.3 AI Orchestration (Agent + Memory + Tools)
A LangChain Agent (powered by Grok) reads the normalized input (prompt + optional image URL), decides which tool to call (text-to-image or image-to-image), and produces a final text response for Telegram.

### 1.4 Kie AI Execution (Async + Polling)
The “tool workflows” are wired to an Execute Workflow Trigger inside the same workflow. Based on `tool_name`, the workflow calls the appropriate Kie AI endpoint to create a task, waits for callback, then queries task results.

### 1.5 Delivery back to Telegram
Sends the agent’s output back to the user as a Telegram text message (note: this workflow, as provided, does **not** send generated images back as Telegram photos—only text).

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Trigger & Authorization Gate
**Overview:** Receives Telegram messages and allows only a specific Telegram user ID to proceed.  
**Nodes involved:** `Get Message`, `Code`, `Switch2` (first hop)

#### Node: Get Message
- **Type / role:** Telegram Trigger (`telegramTrigger`) — entry point.
- **Configuration (interpreted):**
  - Listens to `message` updates.
- **Outputs:** Raw Telegram update payload under `json.message...`
- **Failure/edge cases:**
  - Telegram credential invalid/revoked.
  - Bot not configured with webhook/polling correctly.
  - Message types other than `message` ignored.

#### Node: Code
- **Type / role:** Code node — authorization filter.
- **Configuration (interpreted):**
  - Checks `message.from.id` against a hard-coded allowed ID (`XXX` placeholder).
  - If not matching, returns `{ unauthorized: true }`, else returns original items.
- **Key variables/expressions:**
  - `$input.first().json.message.from.id`
- **Outputs:**
  - Authorized path: unchanged Telegram payload
  - Unauthorized path: an item with `unauthorized: true` (but still continues to `Switch2` in this design)
- **Failure/edge cases:**
  - If incoming payload lacks `message.from.id` (rare but possible for certain update types), code may throw.
  - Unauthorized flow is not explicitly stopped; it will reach `Switch2` but likely won’t match any rule, resulting in no further processing (silent drop).

---

### Block 2 — Input Type Routing (Text / Voice / Image)
**Overview:** Splits processing depending on whether the message contains text, a voice note, or a photo.  
**Nodes involved:** `Switch2`, `Get Text`, `Get voice message`, `Transcribe recording`, `Get input text from voice`, `Get image file`, `Upload image`, `Set Image Url`

#### Node: Switch2
- **Type / role:** Switch — modality router.
- **Configuration (interpreted):**
  - Output “Text” if `message.text` exists.
  - Output “Audio” if `message.voice.file_id` exists.
  - Output “Immagine” if `message.photo[0]` exists.
- **Key expressions:**
  - `={{ $json.message.text }}`
  - `={{ $json.message.voice.file_id }}`
  - `={{ $json.message.photo[0] }}`
- **Outputs:**
  - Text → `Get Text`
  - Audio → `Get voice message`
  - Image → `Get image file`
- **Failure/edge cases:**
  - Messages can contain both photo and caption; photo route will be used (caption handled later).
  - Photo indexing assumptions (see “Get image file”).

#### Node: Get Text
- **Type / role:** Set node — normalize text prompt + session.
- **Configuration (interpreted):**
  - `chatInput = message.text`
  - `sessionId = message.from.id`
- **Outputs:** `{ chatInput, sessionId }` to agent.
- **Edge cases:** empty string text or commands; agent still runs.

#### Node: Get voice message
- **Type / role:** Telegram node (resource: file) — fetches the voice file binary.
- **Configuration (interpreted):**
  - `fileId = Get Message.message.voice.file_id`
- **Outputs:** Binary audio data (typically in `$binary.data`).
- **Failure/edge cases:**
  - Missing `voice.file_id` (route mismatch).
  - Telegram file retrieval errors, large audio, network issues.

#### Node: Transcribe recording
- **Type / role:** OpenAI (LangChain OpenAI node) — audio transcription (Whisper).
- **Configuration (interpreted):**
  - Operation: `audio.transcribe`
  - Language: Italian (`it`)
- **Inputs:** audio binary from previous node.
- **Outputs:** transcription text in `json.text` (typical).
- **Failure/edge cases:**
  - OpenAI credential invalid.
  - Unsupported audio format/encoding.
  - Large file/timeouts.
  - Language forced to `it` may reduce accuracy for other languages.

#### Node: Get input text from voice
- **Type / role:** Set node — normalize transcription into `chatInput`.
- **Configuration:**
  - `chatInput = $json.text`
  - `sessionId = Get Message.message.from.id`
- **Outputs:** `{ chatInput, sessionId }` to agent.
- **Edge cases:** transcription empty; agent may ask clarifying questions.

#### Node: Get image file
- **Type / role:** Telegram node (resource: file) — fetches a photo file binary.
- **Configuration (interpreted):**
  - `fileId = $json.message.photo[2].file_id`
- **Important edge case:** Telegram photo sizes array length varies; `[2]` may not exist. Many messages provide 1–4 sizes. Safer logic would pick the last element.
- **Outputs:** Binary image in `$binary.data`.

#### Node: Upload image
- **Type / role:** FTP node — uploads Telegram image binary to hosting.
- **Configuration (interpreted):**
  - Operation: upload
  - Remote path: `/XXX/{{ $binary.data.fileName }}` (replace `/XXX/` with your directory)
- **Credentials:** FTP BunnyCDN (example)
- **Failure/edge cases:**
  - FTP credentials, permissions, wrong path.
  - Filename collisions (same `fileName` overwriting).
  - Binary property name mismatch (expects `$binary.data`).

#### Node: Set Image Url
- **Type / role:** Set node — constructs public URL + prompt context for agent.
- **Configuration:**
  - `image_url = https://XXX/{{ $binary.data.fileName }}` (replace `https://XXX/` with your public CDN domain/path)
  - `chatInput = Get Message.message.caption || ""`
  - `sessionId = Get Message.message.from.id`
- **Edge cases:**
  - Caption may be empty; agent must infer intent from image alone.
  - Public URL must be truly reachable by Kie AI; private FTP-only URLs will fail.

---

### Block 3 — Orchestrator Agent (LLM + Memory + Tooling)
**Overview:** A Grok-powered agent interprets the prompt (and optional uploaded image URL), chooses the correct generation tool, and prepares user-facing output.  
**Nodes involved:** `Grok Imagine Agent`, `Grok 4.1 Fast`, `Simple Memory1`, `Run text to image`, `Run image to image`, `Send a text message`

#### Node: Grok 4.1 Fast
- **Type / role:** OpenRouter Chat Model node (`lmChatOpenRouter`) — provides the LLM to the agent.
- **Configuration:**
  - Model: `x-ai/grok-4.1-fast`
- **Credentials:** OpenRouter API key.
- **Failure/edge cases:**
  - OpenRouter quota/model availability.
  - Policy refusals based on prompt content.
  - Latency/timeouts.

#### Node: Simple Memory1
- **Type / role:** Memory Buffer Window — session-based short-term conversation memory.
- **Configuration:**
  - Session key: `{{$json.sessionId}}`
  - Session type: custom key
- **Edge cases:**
  - If `sessionId` missing, memory may not group properly.
  - Memory window size defaults (not explicitly set) may limit context.

#### Node: Run text to image
- **Type / role:** Tool Workflow (`toolWorkflow`) — tool callable by the agent.
- **Configuration (interpreted):**
  - Exposes tool inputs: `tool_name`, `query`, `duration`, `video_url`, `image_url`
  - Populates `query` via `$fromAI('query', ...)`
  - Sets `tool_name` to `"Run text to image"`
  - **WorkflowId points to the same workflow ID** (self-reference) and cached name shows “Video Grok Agent with Telegram” (mismatch).
- **Critical note:** The Italian description says it should be called only to edit an existing video—this conflicts with node name and intended function. The switch later relies on `tool_name`.
- **Failure/edge cases:**
  - Misconfigured tool name mismatch vs agent system message (“run_text_to_image” vs “Run text to image”).
  - Self-referencing workflow tool can create recursion if not carefully separated by trigger types (here it routes through Execute Workflow Trigger, so it can work, but naming/intent confusion is high risk).

#### Node: Run image to image
- **Type / role:** Tool Workflow — tool callable by the agent for img2img.
- **Configuration:**
  - Requires `image_url` via `$fromAI('image_url', ...)`
  - `tool_name` set to `"Run image to image"`
- **Failure/edge cases:**
  - Same tool name mismatch risk with system prompt (“run_image_to_image” vs “Run image to image”).
  - If `image_url` not publicly accessible, Kie AI job will fail.

#### Node: Grok Imagine Agent
- **Type / role:** LangChain Agent — orchestration and tool calling.
- **Configuration (interpreted):**
  - Prompt text fed to agent:
    - `{{$json.chatInput ?? ''}}`
    - `Image Url (if exist): {{$json.image_url ?? ''}}`
  - System message: long instruction block describing two tools:
    - `run_text_to_image`
    - `run_image_to_image`
- **Key issue:** Tool names in system message are **snake_case**, but actual tools are named **“Run text to image”** and **“Run image to image”** (title case). This can prevent correct tool invocation unless the model guesses.
- **Connections:**
  - Uses `Grok 4.1 Fast` as language model input.
  - Uses `Simple Memory1` for memory.
  - Uses both tool nodes as available tools.
- **Outputs:** Agent result in `json.output` (used by Telegram send node).
- **Failure/edge cases:**
  - Tool selection failure due to naming mismatch.
  - Model may respond with plain text instead of calling tools.
  - Missing `image_url` but user intent is img2img → agent should ask, but your tool schema allows `image_url` optional; runtime failure later if absent.

#### Node: Send a text message
- **Type / role:** Telegram node — sends agent output back to user.
- **Configuration:**
  - `chatId = Get Message.message.from.id`
  - `text = {{$json.output}}`
- **Edge cases:**
  - Very long messages may exceed Telegram limits.
  - If the agent produces structured JSON but not user-friendly text, user experience suffers.

---

### Block 4 — Tool Execution Backend (Kie AI Trigger + Task Routing)
**Overview:** Receives tool invocations (as “workflow inputs”), routes to the correct Kie AI endpoint based on `tool_name`.  
**Nodes involved:** `Run Kei AI`, `Switch`

#### Node: Run Kei AI
- **Type / role:** Execute Workflow Trigger — entry point for tool executions (called by Tool Workflow nodes).
- **Configuration:**
  - Accepts workflow inputs: `tool_name`, `query`, `duration`, `video_url`, `image_url`
- **Outputs:** Passed to `Switch`.
- **Failure/edge cases:**
  - If tool workflow invocation doesn’t provide `tool_name`, routing fails.
  - Because this is inside the same workflow, ensure you understand that **Telegram trigger** and **Execute Workflow Trigger** are distinct entry points.

#### Node: Switch
- **Type / role:** Switch — routes tool request to the right HTTP call.
- **Configuration:**
  - If `tool_name == "Run text to image"` → `Run text to image1`
  - If `tool_name == "Run image to image"` → `Run image to image1`
- **Failure/edge cases:**
  - Any mismatch in `tool_name` causes silent no-output.
  - Case sensitivity matters.

---

### Block 5 — Kie AI: Text-to-Image Async Flow
**Overview:** Creates a Kie AI text-to-image task, waits for callback to resume, then queries task record info.  
**Nodes involved:** `Run text to image1`, `Wait1`, `Result text to image`

#### Node: Run text to image1
- **Type / role:** HTTP Request — create Kie AI task.
- **Configuration:**
  - POST `https://api.kie.ai/api/v1/jobs/createTask`
  - Bearer auth (generic HTTP Bearer credential “Kie AI”)
  - JSON body:
    - `model: "grok-imagine/text-to-image"`
    - `callBackUrl: {{$execution.resumeUrl}}`
    - `input.prompt: {{$json.query}}`
    - `input.aspect_ratio: "3:2"`
- **Outputs:** Expects `json.data.taskId`
- **Failure/edge cases:**
  - Kie API key invalid.
  - `query` empty → poor or rejected generation.
  - Callback URL must be reachable from Kie AI (public internet).
  - If n8n is behind firewall/private network, callback won’t arrive.

#### Node: Wait1
- **Type / role:** Wait node (resume via webhook) — pauses execution until callback is received.
- **Configuration:**
  - Resume: webhook, HTTP method POST
- **Important:** Both `Wait1` and `Wait2` share the same `webhookId` in the JSON; in practice n8n manages wait webhooks per execution, but identical IDs can be confusing when editing/exporting.
- **Failure/edge cases:**
  - Callback never received → execution remains waiting.
  - Callback received but payload unexpected; still resumes but next step may fail.

#### Node: Result text to image
- **Type / role:** HTTP Request — fetch task record info.
- **Configuration:**
  - GET (default) `https://api.kie.ai/api/v1/jobs/recordInfo`
  - Query param `taskId = Run text to image1.json.data.taskId`
  - Bearer auth
- **Outputs:** Kie AI record info (typically includes generated image URLs).
- **Current design gap:** The retrieved image URLs are not routed back to Telegram as images anywhere in this workflow.

---

### Block 6 — Kie AI: Image-to-Image Async Flow
**Overview:** Creates a Kie AI image-to-image task, waits for callback, then queries results.  
**Nodes involved:** `Run image to image1`, `Wait2`, `Result image to image`

#### Node: Run image to image1
- **Type / role:** HTTP Request — create img2img task.
- **Configuration:**
  - POST `https://api.kie.ai/api/v1/jobs/createTask`
  - JSON body:
    - `model: "grok-imagine/image-to-image"`
    - `callBackUrl: {{$execution.resumeUrl}}`
    - `input.prompt: {{$json.query}}`
    - `input.image_urls: ["{{$json.image_url}}"]`
- **Failure/edge cases:**
  - `image_url` unreachable (most common failure).
  - Unsupported image format/size per Kie AI constraints.

#### Node: Wait2
- **Type / role:** Wait node (resume via webhook).
- **Same considerations** as Wait1.

#### Node: Result image to image
- **Type / role:** HTTP Request — fetch record info with taskId from `Run image to image1`.
- **Current design gap:** Not used to deliver images to Telegram.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Message | telegramTrigger | Entry point: receive Telegram messages | — | Code | ## STEP 1 - Telegram and switch / Set your Telegram ID here |
| Code | code | Authorization gate by Telegram user ID | Get Message | Switch2 | ## STEP 1 - Telegram and switch / Set your Telegram ID here |
| Switch2 | switch | Route by modality (text/voice/photo) | Code | Get Text; Get voice message; Get image file | ## STEP 1 - Telegram and switch / Set your Telegram ID here |
| Get Text | set | Normalize text input to `chatInput` + `sessionId` | Switch2 (Text) | Grok Imagine Agent | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Get voice message | telegram | Download voice file binary | Switch2 (Audio) | Transcribe recording | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Transcribe recording | openAi (LangChain) | Whisper transcription | Get voice message | Get input text from voice | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Get input text from voice | set | Normalize transcription to `chatInput` + `sessionId` | Transcribe recording | Grok Imagine Agent | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Get image file | telegram | Download photo binary from Telegram | Switch2 (Immagine) | Upload image | ## STEP2 - Upload image to server / Set up your FTP space (eg. with [BunnyCDN](https://bunny.net?ref=0pfu5rh4tp)) |
| Upload image | ftp | Upload image to FTP hosting | Get image file | Set Image Url | ## STEP2 - Upload image to server / Set up your FTP space (eg. with [BunnyCDN](https://bunny.net?ref=0pfu5rh4tp)) |
| Set Image Url | set | Build public `image_url` + caption prompt + session | Upload image | Grok Imagine Agent | ## STEP2 - Upload image to server / Set up your FTP space (eg. with [BunnyCDN](https://bunny.net?ref=0pfu5rh4tp)) |
| Grok 4.1 Fast | lmChatOpenRouter | LLM provider for agent (OpenRouter Grok) | — | Grok Imagine Agent (AI language model port) | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Simple Memory1 | memoryBufferWindow | Session memory for agent | — | Grok Imagine Agent (AI memory port) | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Run text to image | toolWorkflow | Agent tool: triggers backend execution (text→image) | — (called by agent) | Grok Imagine Agent (AI tool port) | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Run image to image | toolWorkflow | Agent tool: triggers backend execution (image→image) | — (called by agent) | Grok Imagine Agent (AI tool port) | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Grok Imagine Agent | agent (LangChain) | Orchestrate intent, call tools, produce response | Get Text / Get input text from voice / Set Image Url | Send a text message | ## STEP 3 Orchestrator Agents / Analyzes user input and determines whether to: - Generate a new image from text description (text-to-image) - Transform an existing image using a prompt (image-to-image) |
| Send a text message | telegram | Send agent output to Telegram user | Grok Imagine Agent | — |  |
| Run Kei AI | executeWorkflowTrigger | Tool execution entry point (called by Tool Workflow nodes) | — | Switch | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Switch | switch | Route tool call by `tool_name` | Run Kei AI | Run text to image1; Run image to image1 | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Run text to image1 | httpRequest | Kie AI createTask (text-to-image) | Switch | Wait1 | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Wait1 | wait | Pause until Kie callback then resume | Run text to image1 | Result text to image | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Result text to image | httpRequest | Kie AI recordInfo (poll results) | Wait1 | — | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Run image to image1 | httpRequest | Kie AI createTask (image-to-image) | Switch | Wait2 | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Wait2 | wait | Pause until Kie callback then resume | Run image to image1 | Result image to image | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Result image to image | httpRequest | Kie AI recordInfo (poll results) | Wait2 | — | ## STEP  4 -  Kie AI API / Get your [Kie API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) for FREE and set Bearer Token |
| Sticky Note | stickyNote | Comment / step label | — | — |  |
| Sticky Note1 | stickyNote | Comment / step label | — | — |  |
| Sticky Note2 | stickyNote | Comment / step label | — | — |  |
| Sticky Note3 | stickyNote | Comment / project description | — | — |  |
| Sticky Note8 | stickyNote | Comment / step label | — | — |  |
| Sticky Note9 | stickyNote | Comment / channel promo | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Bot + Credentials**
   1. Create a bot with **BotFather** and obtain the token.
   2. In n8n, create **Telegram API credentials** with that token.
   3. Note your personal Telegram numeric user ID (for allow-listing).

2) **Add Trigger: Telegram Trigger**
   1. Add node: **Telegram Trigger**
   2. Updates: `message`
   3. Select your Telegram credentials.

3) **Add Authorization Gate**
   1. Add node: **Code**
   2. Paste logic that compares `message.from.id` to your allowed ID.
      - Replace `XXX` with your Telegram user ID.
   3. Connect: `Telegram Trigger → Code`.

4) **Add Modality Router**
   1. Add node: **Switch**
   2. Create 3 rules (Exists checks):
      - Text: `{{$json.message.text}}` exists
      - Audio: `{{$json.message.voice.file_id}}` exists
      - Image: `{{$json.message.photo[0]}}` exists
   3. Connect: `Code → Switch`.

5) **Text Path**
   1. Add node: **Set** named “Get Text”
      - `chatInput = {{$json.message.text}}`
      - `sessionId = {{$json.message.from.id}}`
   2. Connect: `Switch(Text) → Get Text`.

6) **Voice Path**
   1. Add node: **Telegram** (resource: *File*) named “Get voice message”
      - `fileId = {{$('Telegram Trigger').item.json.message.voice.file_id}}`
   2. Add node: **OpenAI** (LangChain) named “Transcribe recording”
      - Resource: Audio
      - Operation: Transcribe
      - Language: `it` (or remove to auto-detect)
      - Configure **OpenAI credentials**
   3. Add node: **Set** named “Get input text from voice”
      - `chatInput = {{$json.text}}`
      - `sessionId = {{$('Telegram Trigger').item.json.message.from.id}}`
   4. Connect: `Switch(Audio) → Get voice message → Transcribe recording → Get input text from voice`.

7) **Image Path (Upload + Public URL)**
   1. Add node: **Telegram** (resource: *File*) named “Get image file”
      - `fileId = {{$json.message.photo[2].file_id}}`  
        (Recommended improvement: select the last array element rather than `[2]`.)
   2. Add node: **FTP** named “Upload image”
      - Operation: Upload
      - Path: `/YOUR_FOLDER/{{$binary.data.fileName}}`
      - Configure **FTP credentials** (e.g., BunnyCDN storage/FTP)
   3. Add node: **Set** named “Set Image Url”
      - `image_url = https://YOUR_PUBLIC_DOMAIN/{{$binary.data.fileName}}`
      - `chatInput = {{$('Telegram Trigger').item.json.message.caption || ""}}`
      - `sessionId = {{$('Telegram Trigger').item.json.message.from.id}}`
   4. Connect: `Switch(Image) → Get image file → Upload image → Set Image Url`.

8) **Create the AI Orchestrator (Agent + LLM + Memory)**
   1. Add node: **OpenRouter Chat Model** named “Grok 4.1 Fast”
      - Model: `x-ai/grok-4.1-fast`
      - Configure **OpenRouter** credentials.
   2. Add node: **Memory Buffer Window** named “Simple Memory1”
      - Session key: `{{$json.sessionId}}`
      - Session type: Custom key
   3. Add node: **AI Agent** named “Grok Imagine Agent”
      - Prompt text: combine `chatInput` and optional `image_url`
      - System message: describe tool usage and response handling (as in your workflow)
   4. Connect inputs to Agent:
      - `Grok 4.1 Fast` → Agent (AI language model port)
      - `Simple Memory1` → Agent (AI memory port)
      - From each modality Set node (`Get Text`, `Get input text from voice`, `Set Image Url`) → Agent (main input)

9) **Create Tooling (Tool Workflow nodes)**
   1. Add **Tool Workflow** node named “Run text to image”
      - Define inputs including at least: `tool_name`, `query`
      - Set `tool_name` default to `"Run text to image"`
      - Provide `query` from `$fromAI('query', ...)`
      - Select target workflow: the workflow that contains the **Execute Workflow Trigger** for Kie AI execution (can be the same workflow, but a separate dedicated workflow is safer).
   2. Add **Tool Workflow** node named “Run image to image”
      - Inputs: `tool_name`, `query`, `image_url`
      - `tool_name = "Run image to image"`
      - `image_url` via `$fromAI('image_url', ...)`
   3. Connect both tool nodes to Agent (AI tool port).

   **Important:** Align tool names in the agent system message with the actual tool node names (case-sensitive), otherwise tool calling can fail.

10) **Backend Execution Entry (Execute Workflow Trigger)**
   1. Add node: **Execute Workflow Trigger** named “Run Kei AI”
      - Define inputs: `tool_name`, `query`, `duration`, `video_url`, `image_url`
   2. Add node: **Switch** named “Switch”
      - Route on `{{$json.tool_name}}` equals `"Run text to image"` or `"Run image to image"`.

11) **Kie AI HTTP Calls + Async Waiting**
   1. Add **HTTP Request** “Run text to image1”
      - POST `https://api.kie.ai/api/v1/jobs/createTask`
      - Auth: Bearer (create a **HTTP Bearer Auth** credential with your Kie API key)
      - Body includes `callBackUrl: {{$execution.resumeUrl}}` and prompt.
   2. Add **Wait** node “Wait1”
      - Resume: Webhook, method POST
   3. Add **HTTP Request** “Result text to image”
      - GET `https://api.kie.ai/api/v1/jobs/recordInfo`
      - Query: `taskId = {{$('Run text to image1').item.json.data.taskId}}`

   4. Add **HTTP Request** “Run image to image1”
      - Same createTask endpoint, model `grok-imagine/image-to-image`, include `image_urls`.
   5. Add **Wait** node “Wait2” (resume via webhook)
   6. Add **HTTP Request** “Result image to image”
      - Query uses taskId from “Run image to image1”.

   7. Wire:
      - `Run Kei AI → Switch`
      - `Switch(text) → Run text to image1 → Wait1 → Result text to image`
      - `Switch(img) → Run image to image1 → Wait2 → Result image to image`

12) **Telegram Response**
   1. Add node: **Telegram** named “Send a text message”
      - Chat ID: `{{$('Telegram Trigger').item.json.message.from.id}}`
      - Text: `{{$json.output}}`
   2. Connect: `Grok Imagine Agent → Send a text message`.

   **Note:** To actually send generated images, add Telegram “Send Photo” nodes fed by URLs from the Kie AI “Result …” nodes, or have the agent consume those outputs and return URLs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Set up your FTP space (eg. with BunnyCDN)” | https://bunny.net?ref=0pfu5rh4tp |
| “Get your Kie API Key for FREE and set Bearer Token” | https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5 |
| “MY NEW YOUTUBE CHANNEL… Subscribe…” | https://youtube.com/@n3witalia |
| YouTube cover image used in sticky note | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |
| Architectural note: current workflow retrieves Kie AI results but doesn’t send images back | Consider adding Telegram “Send Photo” nodes after `Result text to image` / `Result image to image`, or refactor tools to return image URLs to the agent. |

_Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques._