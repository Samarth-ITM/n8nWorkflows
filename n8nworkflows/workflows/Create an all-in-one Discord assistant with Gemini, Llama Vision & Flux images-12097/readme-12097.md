Create an all-in-one Discord assistant with Gemini, Llama Vision & Flux images

https://n8nworkflows.xyz/workflows/create-an-all-in-one-discord-assistant-with-gemini--llama-vision---flux-images-12097


# Create an all-in-one Discord assistant with Gemini, Llama Vision & Flux images

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow exposes a single POST webhook that your Discord bot calls. It behaves as an all‑in‑one assistant named **O’Carla** with three capabilities:

1) **Normal chat** (Gemini 2.5) with per‑user memory  
2) **Image generation** when the user prompt begins with `gambar:` (Gemini prompt “beautification” → Pollinations Flux → TinyURL → Discord formatted reply)  
3) **Vision & file analysis** when Discord sends attachments (Llama 3.2 Vision via OpenRouter for images; Gemini for extracted text files)

**Main logical blocks (node-to-node):**
- **1.1 Input Reception & Routing (Discord Bot → Webhook):** `Webhook` → `Has Atc` → branches
- **1.2 Image Generation Path (keyword gambar:):** `If` → `Code` → `Fields - Set Values` → `AI Agent - Create Image From Prompt` → `Code - Clean Json` → `Code - Get Prompt` → `Code - Set Filename` → `Create Image` → `Error Handler` → `Handle Respons` → `Change Link To Be Sort` → `Format Discord Response` → `Respond to Webhook2`
- **1.3 Vision & File Analysis Path (attachments):** `Code1` → `If1` → (image) `HTTP Request` → `Discord AI Response Agent2` → `correctNaming3` → `Respond to Webhook4` OR (file) `HTTP Request1` → `correctNaming2` → `Discord AI Response Agent1` → `correctNaming1` → `Respond to Webhook3`
- **1.4 General Chat Path (no attachments + no gambar:):** `If` (false) → `Discord AI Response Agent` → `correctNaming` → `Respond to Webhook`
- **1.5 Memory & Model Hub:** 3 Memory Buffer nodes attach to 3 agent nodes; Gemini/OpenRouter chat models attach as LLM providers.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Smart Routing

**Overview:**  
Receives Discord bot payloads via webhook, then routes to attachment analysis or message intent detection (`gambar:` vs normal chat).

**Nodes involved:** `Webhook`, `Has Atc`, `If`, `Sticky Note6`, `Sticky Note7`

#### Node: Webhook
- **Type / Role:** `n8n-nodes-base.webhook` — entry point for Discord bot POST events.
- **Config:**  
  - Method: **POST**  
  - Path: `b0631bec-9ccc-4eb8-b143-d73609b213c7`  
  - Response mode: **Respond with a “Respond to Webhook” node** (`responseNode`)
- **Key inputs expected (from Discord bot):**  
  - `body.userId` (Discord user id)  
  - `body.question` (user prompt)  
  - `body.attachments` (array; optional)
- **Outputs:** to `Has Atc`
- **Failure modes:** incorrect payload structure (missing `body`), webhook not reachable from bot, wrong method/path.

#### Node: Has Atc
- **Type / Role:** `n8n-nodes-base.if` — detects whether the request contains attachments.
- **Condition:** `{{$json.body.attachments.length}} > 0`
- **True output:** to `Code1` (vision/file analysis path)  
- **False output:** to `If` (image keyword check / chat path)
- **Edge cases:** `attachments` missing or not an array → expression error. (Mitigation: ensure bot always sends `attachments: []`.)

#### Node: If
- **Type / Role:** `n8n-nodes-base.if` — detects image generation intent.
- **Condition:** `{{$json.body.question}} contains "gambar:"`
- **True output:** to `Code` (image generation path)  
- **False output:** to `Discord AI Response Agent` (chat path)
- **Edge cases:** leading whitespace, different casing (`Gambar:`) won’t match due to case sensitivity; also note the leftValue includes a newline in the expression which can be harmless but is messy.

**Sticky notes in this block:**
- **Sticky Note6:** Discord bot integration and routing explanation (covers intake/routing area).
- **Sticky Note7:** Overall product description and requirements (Gemini/OpenRouter keys + Discord bot).

---

### 2.2 General Chat Path (Text-only)

**Overview:**  
Handles standard conversation when there are no attachments and the user did not request an image. Uses Gemini 2.5 Flash plus a per-user memory buffer.

**Nodes involved:** `Discord AI Response Agent`, `Google Gemini Chat Model2`, `Simple Memory`, `correctNaming`, `Respond to Webhook`, `Sticky Note8`, `Sticky Note9`

#### Node: Discord AI Response Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — main conversational agent.
- **Input text:**  
  ```
  ==User Mention: <@{{ $json.body.userId }}>

  Question/Prompt: {{ $json.body.question }}
  ```
- **System message highlights (interpreted):**
  - Assistant name: **O’Carla**, created by **Asla Fikri**
  - Respond in the same language as the user
  - Must not reference unrelated external content (e.g., “Presting Podcast”) unless asked
  - Must not say it was trained/created by Google
  - If asked creator: point to GitHub/LinkedIn links
  - Response formatting requirement: begin with `<@{{userId}}>` and **no line break**
  - Can generate images when prompt starts with `gambar:`
- **Connections:**
  - Main output → `correctNaming`
  - AI language model input (`ai_languageModel`) ← `Google Gemini Chat Model2`
  - AI memory input (`ai_memory`) ← `Simple Memory`
- **Potential issues:**
  - The “no line break” rule conflicts with other parts of the workflow where multi-line responses are created (especially image path). Also the template uses `{{userId}}` in system message but the prompt uses `$json.body.userId`; ensure consistency.

#### Node: Google Gemini Chat Model2
- **Type / Role:** `lmChatGoogleGemini` — LLM provider for the agent (fast chat).
- **Model:** `models/gemini-2.5-flash-preview-05-20`
- **Options:** `temperature 0.5`, `topK 40`, `topP 1`, `maxOutputTokens 65536`, safety thresholds all BLOCK_NONE.
- **Credential:** `googlePalmApi`
- **Failure modes:** invalid API key, model name not available, token limit errors, safety settings policy changes.

#### Node: Simple Memory
- **Type / Role:** `memoryBufferWindow` — per-user conversation memory (last N turns).
- **Session key:** `{{$('Webhook').item.json.body.userId}}`
- **Window length:** 50 interactions
- **Connections:** memory → `Discord AI Response Agent`
- **Edge cases:** if `userId` missing, different users may share memory or memory key becomes empty.

#### Node: correctNaming
- **Type / Role:** `code` — maps agent output to `{ answer: ... }` for Discord bot.
- **Logic:** takes `item.json.output` and returns `{json:{answer:<output>}}`
- **Output:** to `Respond to Webhook`
- **Failure modes:** if agent output field is not `output` (depends on node version/settings), returns undefined answer.

#### Node: Respond to Webhook
- **Type / Role:** `respondToWebhook` — returns response to the original webhook call.
- **Config:** respond with `allIncomingItems`
- **Failure modes:** if multiple response paths execute (should not happen here), n8n may complain about responding twice; ensure only one terminal respond node runs per request.

**Sticky notes in this block:**
- **Sticky Note8:** “Intelligence Hub” (memory + model switching concept).
- **Sticky Note9:** “Discord Output” formatting concept.

---

### 2.3 Professional Image Path (gambar:)

**Overview:**  
When the user prompt starts with `gambar:`, the workflow generates polished image prompts using Gemini, extracts prompts from the agent output, calls Pollinations Flux to generate images, shortens the resulting URL, then formats a Discord-friendly response.

**Nodes involved:** `Code`, `Fields - Set Values`, `AI Agent - Create Image From Prompt`, `Google Gemini Chat Model1`, `Code - Clean Json`, `Code - Get Prompt`, `Code - Set Filename`, `Create Image`, `Error Handler`, `Handle Respons`, `Change Link To Be Sort`, `Format Discord Response`, `Respond to Webhook2`, `Sticky Note`

#### Node: Code
- **Type / Role:** `code` — extracts and cleans the user prompt.
- **Logic:** removes `gambar:` prefix, returns `{prompt, userId}`
- **Output:** to `Fields - Set Values`
- **Edge cases:** if `question` is empty, prompt becomes empty string; downstream prompt generation may degrade.

#### Node: Fields - Set Values
- **Type / Role:** `set` — establishes image generation parameters.
- **Sets:**  
  - `model = "flux"`  
  - `width = "1080"` *(string)*  
  - `height = "1920"` *(string)*
- **Output:** to `AI Agent - Create Image From Prompt`
- **Edge cases:** width/height stored as strings; later used as numbers in query. Pollinations may accept them, but strict systems might not.

#### Node: AI Agent - Create Image From Prompt
- **Type / Role:** `langchain.agent` — creates professional image prompts (prompt engineering).
- **Input text:** `{{$('Webhook').item.json.body.question}}` (note: uses original question, including `gambar:`)
- **System message:** long guideline requiring JSON-like structure:  
  ```
  {
    prompt_image {prompt : "" , ...}
  }
  ```
- **Has output parser:** enabled (but the workflow still “cleans” output later, implying parser may not be producing clean JSON)
- **Connections:**
  - LLM provider ← `Google Gemini Chat Model1`
  - Output → `Code - Clean Json`
- **Failure modes:** model returns text that does not match expected JSON; output parser errors; overly long output.

#### Node: Google Gemini Chat Model1
- **Type / Role:** Gemini LLM for the image prompt agent.
- **Model:** `models/gemini-2.5-flash-preview-05-20`
- **Options:** same style as other Gemini nodes.
- **Credential:** `googlePalmApi`

#### Node: Code - Clean Json
- **Type / Role:** `code` — attempts to extract prompt lines from agent output.
- **Logic (important):**
  - Reads `response = $input.first().json.output`
  - Splits by newline, extracts lines containing `"prompt":`
  - Collects them into `image_prompt: [...]`
- **Output:** to `Code - Get Prompt`
- **Major edge cases / risks:**
  - If the agent output is valid JSON but formatted differently, this extraction fails.
  - It stores the substring after `"prompt":` without removing quotes/commas; may produce malformed prompts.
  - If no prompts found → empty `image_prompt` array; downstream nodes will produce no items (unless alwaysOutputData saves you).

#### Node: Code - Get Prompt
- **Type / Role:** `code` — converts `image_prompt[]` into multiple HTTP request bodies (one per prompt).
- **Logic:** creates items shaped like:
  ```js
  {
    body: {
      prompt,
      image_size: { width, height },
      num_inference_steps: 12,
      guidance_scale: 3.5,
      num_images: 1,
      enable_safety_checker: true
    }
  }
  ```
- **Dependencies:** pulls `width/height` from `Fields - Set Values`
- **Output:** to `Code - Set Filename`
- **Edge cases:** if `image_prompt` contains trailing quotes/commas, Pollinations prompt may become ugly; also the workflow later uses query params anyway, so some fields here are not used by Pollinations node as configured.

#### Node: Code - Set Filename
- **Type / Role:** `code` — adds `fileName` like `images_001.png`, `images_002.png` to each item.
- **Bug:** the code references `items` without defining it (`const items = $input.all()` is missing). This will throw a ReferenceError at runtime.
- **Output:** intended to go to `Create Image`
- **Fix:** add `const items = $input.all();` at top.
- **Failure modes:** as-is, the image path will fail before calling Pollinations.

#### Node: Create Image
- **Type / Role:** `httpRequest` — calls Pollinations image generation endpoint.
- **URL:** `https://image.pollinations.ai/prompt/{{ prompt }}`
  - In the node it is written with a space after `/prompt/` (`"/prompt/ {{ ... }}"`), which can produce an invalid URL. Should be concatenated without spaces.
- **Query (JSON mode):** width/height/model/seed/nologo.  
- **Headers:** `Content-Type: application/json`, `Accept: application/json`
- **Retry:** enabled; waits 5000ms between tries.
- **Output:** to `Error Handler`
- **Failure modes:** Pollinations rate limits, invalid URL encoding, non-JSON response, timeouts.

#### Node: Error Handler
- **Type / Role:** `code` — filters successful image generations.
- **Logic:** collects items where `item.json.url` exists; if none, returns an error item with an `answer`.
- **Issue:** error `answer` uses a mustache template inside a JS string: `<@{{ $json.body.userId }}>` which will not be evaluated in Code node output. Should use the actual userId like `<@${$('Webhook').first().json.body.userId}>`.
- **Output:** to `Handle Respons`

#### Node: Handle Respons
- **Type / Role:** `code` — normalizes image URL and filename; provides fallback direct Pollinations URL if missing.
- **Logic:**
  - If `item.json.url` exists → output `{imageUrl: url, fileName, success:true}`
  - Else build `directUrl = https://image.pollinations.ai/prompt/<encoded prompt>?width=...&height=...&model=flux...`
- **Issue:** uses `$('Code - Get Prompt').item.json.body.prompt` (single item), but in a loop it should use each item’s prompt; otherwise all fallbacks point to the same prompt.
- **Output:** to `Change Link To Be Sort`

#### Node: Change Link To Be Sort
- **Type / Role:** `httpRequest` — shortens the image URL via TinyURL.
- **Endpoint:** `https://tinyurl.com/api-create.php`
- **Query parameter:** `url={{ $json.imageUrl }}`
- **Output:** to `Format Discord Response`
- **Failure modes:** TinyURL rate limit, returns plain text not under `data` depending on n8n HTTP node settings.

#### Node: Format Discord Response
- **Type / Role:** `code` — creates the final Discord message.
- **Logic:**
  - Reads `userId` and original question from `Webhook`
  - Reads `tinyUrl = $input.first().json.data`
  - Validates it starts with tinyurl / pollinations / http
  - Builds a multi-line response with prompt + result link
- **Outputs:** `{answer, imageUrl, success:true}` to `Respond to Webhook2`
- **Potential mismatch:** earlier system rule says “no line break”; here it intentionally uses `\n\n`.

#### Node: Respond to Webhook2
- **Type / Role:** `respondToWebhook` — returns the formatted image response.

**Sticky note in this block:**
- **Sticky Note:** “Professional Image Path” description.

---

### 2.4 Vision & File Analysis Path (Attachments)

**Overview:**  
If a Discord message includes attachments, the workflow checks if the attachment is an image. If it’s an image, it downloads the file as binary and sends it to a vision-capable Llama model via OpenRouter. If it’s not an image (text-based file), it downloads as text and asks Gemini to analyze the file contents.

**Nodes involved:** `Code1`, `If1`, `HTTP Request`, `Discord AI Response Agent2`, `OpenRouter Chat Model`, `Simple Memory2`, `correctNaming3`, `Respond to Webhook4`, `HTTP Request1`, `correctNaming2`, `Discord AI Response Agent1`, `Google Gemini Chat Model`, `Simple Memory1`, `correctNaming1`, `Respond to Webhook3`, `Sticky Note10`, `Sticky Note9`

#### Node: Code1
- **Type / Role:** `code` — extracts first attachment and sets analysis prompt.
- **Logic:**
  - `attachment = body.attachments[0]`
  - `isImage = attachment.contentType.startsWith("image/")`
  - returns `{isImage, url, mime, filename, prompt: body.question || default}`
- **Bugs / inconsistencies:**
  - Uses `attachment.contentType` but later sets `mime: attachment.content_type` (different property name). One of these may be wrong depending on Discord bot payload.
- **Output:** to `If1`

#### Node: If1
- **Type / Role:** `if` — routes image vs non-image.
- **Condition:** `{{$json.isImage}} is true`
- **True output:** `HTTP Request` (download as file/binary)  
- **False output:** `HTTP Request1` (download as text)
- **Edge cases:** `isImage` missing/undefined → false branch.

#### Node: HTTP Request (image download)
- **Type / Role:** `httpRequest` — downloads attachment as **file**.
- **URL:** `{{$json.imageUrl}}`
- **Bug:** upstream `Code1` outputs `url`, not `imageUrl`. This request will fail unless the incoming payload already has `imageUrl`.
- **Response format:** `file` (binary)
- **Output:** to `Discord AI Response Agent2`
- **Failure modes:** 403/404 from CDN, large file timeouts, wrong URL field.

#### Node: Discord AI Response Agent2 (Vision)
- **Type / Role:** `langchain.agent` — vision/file reasoning agent using OpenRouter’s Llama Vision model.
- **Input text:** includes `Question/Prompt: {{ $json.prompt }}` and a malformed mention template: `<@s{{ userId }}>` (extra `s`).
- **Connections:**
  - LLM provider (`ai_languageModel`) ← `OpenRouter Chat Model`
  - Memory (`ai_memory`) ← `Simple Memory2`
  - Output → `correctNaming3`
- **Edge cases:** For actual vision, the agent must receive the binary image in the format expected by n8n LangChain agent. This workflow downloads a file but does not explicitly attach it to the agent as an image input (depends on agent node capabilities/version). Might not work without additional configuration.

#### Node: OpenRouter Chat Model
- **Type / Role:** `lmChatOpenRouter` — vision-capable LLM via OpenRouter.
- **Model:** `meta-llama/llama-3.2-11b-vision-instruct:free`
- **Credential:** `openRouterApi`
- **Failure modes:** free-tier throttling, model availability changes, OpenRouter auth errors.

#### Node: Simple Memory2
- **Type / Role:** memory buffer for Vision agent.
- **Session key:** `userId` from webhook
- **Window:** 50

#### Node: correctNaming3 → Respond to Webhook4
- **correctNaming3 Type / Role:** maps `output` → `{answer}`  
- **Respond to Webhook4:** returns response for attachment-image analysis.

---

#### Node: HTTP Request1 (file download as text)
- **Type / Role:** `httpRequest` — downloads non-image attachments as **text**.
- **URL:** `{{$json.url}}`
- **Response format:** `text`
- **Output:** `correctNaming2`
- **Failure modes:** binary formats (PDF, DOCX) returned as gibberish; large files; encoding issues.

#### Node: correctNaming2
- **Type / Role:** `code` — wraps file text into a prompt.
- **Output prompt:**  
  `Berikut isi file <filename>:\n\n<downloaded text>`
- **Output:** to `Discord AI Response Agent1`

#### Node: Discord AI Response Agent1 (Text analysis)
- **Type / Role:** `langchain.agent` — analyzes extracted file text using Gemini 2.5 Pro.
- **LLM provider:** `Google Gemini Chat Model`
- **Memory:** `Simple Memory1`
- **Mention template bug:** `<@s{{ userId }}>` (extra `s`), and system message expects `<@{{userId}}>` but `userId` is not injected as a variable automatically.
- **Output:** to `correctNaming1` → `Respond to Webhook3`

#### Node: Google Gemini Chat Model
- **Type / Role:** Gemini LLM for more capable reasoning.
- **Model:** `models/gemini-2.5-pro`
- **Options:** similar to others (temp 0.5, very high max tokens)
- **Failure modes:** token explosion when file is large; consider truncation/summarization pre-step.

#### Node: Simple Memory1
- **Type / Role:** memory buffer for file-text agent.
- **Session key:** per webhook userId, window 50.

---

### 2.5 Output Normalization & Webhook Responses (Cross-cutting)

**Overview:**  
Across all branches, Code nodes normalize the agent outputs into a consistent schema `{answer: ...}`, then Respond-to-Webhook nodes return that to the Discord bot.

**Nodes involved:** `correctNaming`, `correctNaming1`, `correctNaming3`, `Respond to Webhook`, `Respond to Webhook2`, `Respond to Webhook3`, `Respond to Webhook4` (+ `af928d3b...` etc.)

**Key risk:** There are multiple Respond-to-Webhook nodes; only one must run per execution. Routing must be mutually exclusive (it is intended to be via IF nodes).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives Discord bot POST payload | — | Has Atc | ## **1. Discord Bot Integration** This Webhook listens for events from your **Discord Bot script**. - It captures the `userId` to maintain separate memories. - It detects `attachments` to trigger the Vision path. - It routes messages starting with `gambar:` to the Image Generation path. |
| Has Atc | n8n-nodes-base.if | Route: attachments vs no attachments | Webhook | Code1, If | ## **1. Discord Bot Integration** This Webhook listens for events from your **Discord Bot script**. - It captures the `userId` to maintain separate memories. - It detects `attachments` to trigger the Vision path. - It routes messages starting with `gambar:` to the Image Generation path. |
| If | n8n-nodes-base.if | Route: gambar: image path vs chat | Has Atc (false) | Code, Discord AI Response Agent | ## **1. Discord Bot Integration** This Webhook listens for events from your **Discord Bot script**. - It captures the `userId` to maintain separate memories. - It detects `attachments` to trigger the Vision path. - It routes messages starting with `gambar:` to the Image Generation path. |
| Code | n8n-nodes-base.code | Strip `gambar:` and extract prompt/userId | If (true) | Fields - Set Values | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Fields - Set Values | n8n-nodes-base.set | Set Flux params (model/width/height) | Code | AI Agent - Create Image From Prompt | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| AI Agent - Create Image From Prompt | @n8n/n8n-nodes-langchain.agent | Generate polished image prompt JSON-ish | Fields - Set Values | Code - Clean Json | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for image prompt agent | — | AI Agent - Create Image From Prompt (ai_languageModel) | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Code - Clean Json | n8n-nodes-base.code | Extract `"prompt":` lines into image_prompt[] | AI Agent - Create Image From Prompt | Code - Get Prompt | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Code - Get Prompt | n8n-nodes-base.code | Build Pollinations request items | Code - Clean Json | Code - Set Filename | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Code - Set Filename | n8n-nodes-base.code | Add filenames per image item | Code - Get Prompt | Create Image | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Create Image | n8n-nodes-base.httpRequest | Call Pollinations Flux to generate image | Code - Set Filename | Error Handler | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Error Handler | n8n-nodes-base.code | Filter success or return image error | Create Image | Handle Respons | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Handle Respons | n8n-nodes-base.code | Normalize imageUrl + fallback direct URL | Error Handler | Change Link To Be Sort | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Change Link To Be Sort | n8n-nodes-base.httpRequest | TinyURL shortening | Handle Respons | Format Discord Response | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Format Discord Response | n8n-nodes-base.code | Build Discord message with mention + link | Change Link To Be Sort | Respond to Webhook2 | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Respond to Webhook2 | n8n-nodes-base.respondToWebhook | Return image response | Format Discord Response | — | ## **2. Professional Image Path** Uses an AI Agent to "beautify" prompts before generating high-res images with Flux. Results are automatically shortened for a clean Discord UI. |
| Discord AI Response Agent | @n8n/n8n-nodes-langchain.agent | Normal chat agent | If (false) | correctNaming | ## **3. Intelligence Hub** - **Memory Buffer:** Saves the last 50 messages to ensure the AI knows the context of the current conversation. - **Model Switching:** Automatically uses **Gemini 2.5** for text or **OpenRouter** models for specialized visual tasks. |
| Google Gemini Chat Model2 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for normal chat | — | Discord AI Response Agent (ai_languageModel) | ## **3. Intelligence Hub** - **Memory Buffer:** Saves the last 50 messages to ensure the AI knows the context of the current conversation. - **Model Switching:** Automatically uses **Gemini 2.5** for text or **OpenRouter** models for specialized visual tasks. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for normal chat | — | Discord AI Response Agent (ai_memory) | ## **3. Intelligence Hub** - **Memory Buffer:** Saves the last 50 messages to ensure the AI knows the context of the current conversation. - **Model Switching:** Automatically uses **Gemini 2.5** for text or **OpenRouter** models for specialized visual tasks. |
| correctNaming | n8n-nodes-base.code | Map agent output → answer | Discord AI Response Agent | Respond to Webhook | ## **4. Discord Output** Formats and sends the final AI response back to the user with correct mentions and clear formatting. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return chat response | correctNaming | — | ## **4. Discord Output** Formats and sends the final AI response back to the user with correct mentions and clear formatting. |
| Code1 | n8n-nodes-base.code | Extract first attachment + isImage + prompt | Has Atc (true) | If1 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| If1 | n8n-nodes-base.if | Route: image vs non-image attachment | Code1 | HTTP Request, HTTP Request1 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| HTTP Request | n8n-nodes-base.httpRequest | Download image attachment as file | If1 (true) | Discord AI Response Agent2 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Vision LLM provider | — | Discord AI Response Agent2 (ai_languageModel) | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| Simple Memory2 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for vision agent | — | Discord AI Response Agent2 (ai_memory) | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| Discord AI Response Agent2 | @n8n/n8n-nodes-langchain.agent | Vision analysis agent | HTTP Request | correctNaming3 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| correctNaming3 | n8n-nodes-base.code | Map vision output → answer | Discord AI Response Agent2 | Respond to Webhook4 | ## **4. Discord Output** Formats and sends the final AI response back to the user with correct mentions and clear formatting. |
| Respond to Webhook4 | n8n-nodes-base.respondToWebhook | Return vision response | correctNaming3 | — | ## **4. Discord Output** Formats and sends the final AI response back to the user with correct mentions and clear formatting. |
| HTTP Request1 | n8n-nodes-base.httpRequest | Download non-image attachment as text | If1 (false) | correctNaming2 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| correctNaming2 | n8n-nodes-base.code | Build “file content” prompt | HTTP Request1 | Discord AI Response Agent1 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for file-text analysis | — | Discord AI Response Agent1 (ai_languageModel) | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| Simple Memory1 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for file-text agent | — | Discord AI Response Agent1 (ai_memory) | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| Discord AI Response Agent1 | @n8n/n8n-nodes-langchain.agent | File text analysis agent | correctNaming2 | correctNaming1 | ## **👁️ Vision & File Analysis** Triggers when an image or file is uploaded. - **Image Support:** Uses **Llama 3.2 Vision** to describe content. - **File Support:** Downloads and reads text-based files to provide summaries or answers based on the document. |
| correctNaming1 | n8n-nodes-base.code | Map analysis output → answer | Discord AI Response Agent1 | Respond to Webhook3 | ## **4. Discord Output** Formats and sends the final AI response back to the user with correct mentions and clear formatting. |
| Respond to Webhook3 | n8n-nodes-base.respondToWebhook | Return file-text analysis response | correctNaming1 | — | ## **4. Discord Output** Formats and sends the final AI response back to the user with correct mentions and clear formatting. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note | — | — | ### **Try O'Carla!** O'Carla is an all-in-one Discord AI assistant... Contact via [LinkedIn](https://www.linkedin.com/in/aslamul-fikri-alfirdausi) or [GitHub](https://github.com/masrigaa). |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note | — | — | ## **1. Discord Bot Integration** ... |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note | — | — | ## **2. Professional Image Path** ... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation note | — | — | ## **3. Intelligence Hub** ... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation note | — | — | ## **4. Discord Output** ... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation note | — | — | ## **👁️ Vision & File Analysis** ... |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook (Trigger)**
   1. Add node **Webhook**.
   2. Set **HTTP Method: POST**.
   3. Set **Path** to a unique string (or keep generated).
   4. Set **Response mode: Using “Respond to Webhook” node**.

2) **Add routing: attachments vs no attachments**
   1. Add **IF** node named **Has Atc**.
   2. Condition: Number → `{{$json.body.attachments.length}}` **greater than** `0`.
   3. Connect `Webhook → Has Atc`.

3) **Attachment branch: extract attachment metadata**
   1. Add **Code** node named `Code1` on **true** output of `Has Atc`.
   2. Use code to pick `body.attachments[0]`, set:
      - `isImage` = content type startsWith `"image/"`
      - `url`, `filename`, `prompt` (from `body.question` or a default)
   3. Ensure your Discord bot payload uses consistent fields (`contentType` vs `content_type`).

4) **Attachment branch routing: image vs file**
   1. Add **IF** node `If1`.
   2. Condition: Boolean → `{{$json.isImage}}` is true.
   3. Connect `Code1 → If1`.

5) **Image attachment (Vision) sub-branch**
   1. Add **HTTP Request** node (name `HTTP Request`).
      - URL: `{{$json.url}}` (important: use `url`, not `imageUrl`)
      - Response: **File**
   2. Add **OpenRouter Chat Model** node:
      - Credential: **OpenRouter API**
      - Model: `meta-llama/llama-3.2-11b-vision-instruct:free`
   3. Add **Simple Memory** node (name `Simple Memory2`):
      - Session Key: `{{$('Webhook').item.json.body.userId}}`
      - Context window: 50
   4. Add **AI Agent** node (name `Discord AI Response Agent2`):
      - Set prompt text to include the user question and mention.
      - Attach **OpenRouter Chat Model** to `ai_languageModel`.
      - Attach **Simple Memory2** to `ai_memory`.
      - Ensure the agent is configured to accept image/binary input if required by your n8n version.
   5. Add **Code** node `correctNaming3` mapping agent output to `{answer: ...}`.
   6. Add **Respond to Webhook** node `Respond to Webhook4`.
   7. Connect: `If1(true) → HTTP Request → Discord AI Response Agent2 → correctNaming3 → Respond to Webhook4`.

6) **Non-image attachment (text file) sub-branch**
   1. Add **HTTP Request** node `HTTP Request1`:
      - URL: `{{$json.url}}`
      - Response: **Text**
   2. Add **Code** node `correctNaming2` to build: `prompt = "Berikut isi file ...\n\n<text>"`
   3. Add **Gemini Chat Model** node `Google Gemini Chat Model`:
      - Credential: Google Gemini / PaLM credential
      - Model: `models/gemini-2.5-pro`
   4. Add **Memory** node `Simple Memory1`:
      - Session key: webhook userId
      - Window: 50
   5. Add **AI Agent** node `Discord AI Response Agent1`:
      - Input: `{{$json.prompt}}`
      - LLM: `Google Gemini Chat Model`
      - Memory: `Simple Memory1`
   6. Add **Code** node `correctNaming1` to map output → `{answer: ...}`
   7. Add **Respond to Webhook** node `Respond to Webhook3`
   8. Connect: `If1(false) → HTTP Request1 → correctNaming2 → Discord AI Response Agent1 → correctNaming1 → Respond to Webhook3`

7) **No-attachment branch: detect image keyword**
   1. From `Has Atc(false)`, add **IF** node `If`.
   2. Condition: String contains `gambar:` on `{{$json.body.question}}`.
   3. Connect `Has Atc(false) → If`.

8) **No-attachment + normal chat**
   1. Add **Gemini Chat Model** node `Google Gemini Chat Model2`:
      - Model: `models/gemini-2.5-flash-preview-05-20`
   2. Add **Memory** node `Simple Memory` with session key = webhook userId, window 50.
   3. Add **AI Agent** node `Discord AI Response Agent` using the provided system message constraints.
      - Connect `Google Gemini Chat Model2` to `ai_languageModel`
      - Connect `Simple Memory` to `ai_memory`
   4. Add **Code** node `correctNaming` mapping output → `{answer: ...}`
   5. Add **Respond to Webhook** node `Respond to Webhook`
   6. Connect: `If(false) → Discord AI Response Agent → correctNaming → Respond to Webhook`

9) **No-attachment + gambar: image generation**
   1. Add **Code** node `Code` to strip `gambar:` and emit `{prompt, userId}`.
   2. Add **Set** node `Fields - Set Values` to set `model=flux`, `width=1080`, `height=1920`.
   3. Add **Gemini Chat Model** node `Google Gemini Chat Model1` (Flash).
   4. Add **AI Agent** `AI Agent - Create Image From Prompt` with the long “image prompt guidelines” system message.
      - Attach `Google Gemini Chat Model1` as LLM.
   5. Add **Code** `Code - Clean Json` to extract prompt(s).
   6. Add **Code** `Code - Get Prompt` to create one item per prompt.
   7. Add **Code** `Code - Set Filename` and **fix it** by starting with `const items = $input.all();`.
   8. Add **HTTP Request** `Create Image`:
      - URL should be `https://image.pollinations.ai/prompt/{{$json.body.prompt}}` (or use the prompt field you produced)
      - Ensure **no extra spaces** in the URL
      - Add query parameters: width/height/model/seed/nologo
   9. Add **Code** `Error Handler` and fix mention formatting inside JS (use actual userId).
   10. Add **Code** `Handle Respons`.
   11. Add **HTTP Request** `Change Link To Be Sort`:
       - `https://tinyurl.com/api-create.php?url={{$json.imageUrl}}`
   12. Add **Code** `Format Discord Response` (ensure it matches your “no line break” policy if needed).
   13. Add **Respond to Webhook** node `Respond to Webhook2`.
   14. Connect: `If(true) → Code → Fields - Set Values → AI Agent - Create Image From Prompt → Code - Clean Json → Code - Get Prompt → Code - Set Filename → Create Image → Error Handler → Handle Respons → Change Link To Be Sort → Format Discord Response → Respond to Webhook2`

10) **Credentials**
   - Create/attach **Google Gemini (PaLM) credential** for all Gemini model nodes.
   - Create/attach **OpenRouter API credential** for OpenRouter model node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| O’Carla assistant branding, capabilities, and requirements: Gemini API key, OpenRouter API key, Discord bot production webhook URL. | Included in workflow sticky note: “Try O’Carla!” |
| Creator attribution rules: if asked who created the assistant, point to GitHub/LinkedIn. | GitHub: https://github.com/masrigaa ; LinkedIn: https://www.linkedin.com/in/aslamul-fikri-alfirdausi |

