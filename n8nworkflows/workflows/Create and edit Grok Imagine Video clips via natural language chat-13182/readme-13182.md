Create and edit Grok Imagine Video clips via natural language chat

https://n8nworkflows.xyz/workflows/create-and-edit-grok-imagine-video-clips-via-natural-language-chat-13182


# Create and edit Grok Imagine Video clips via natural language chat

## Disclaimer (provided)
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Create and edit Grok Imagine Video clips via natural language chat  
Workflow name: **Grok Imagine Video Agent**

---

## 1. Workflow Overview

This workflow provides a **chat-based AI agent** that can:
- **Generate a new video from text**
- **Generate a new video from an input image**
- **Edit an existing video**

It uses:
- **OpenRouter (Grok 4.1 Fast)** as the reasoning model (LangChain Agent node)
- **Fal.run “queue” endpoints** for Grok Imagine Video generation/editing (async jobs with request IDs)
- **Optional image upload via FTP (BunnyCDN)** to host user-provided images and convert them into a public URL the model can use

### 1.1 Chat Intake + Optional Image Upload
Receives user messages (and optional image file). If an image is uploaded, it is pushed to FTP storage and a public `image_url` is produced.

### 1.2 AI Orchestration (Intent → Tool choice)
A LangChain agent (powered by Grok 4.1 Fast) interprets the user’s request, ensures required parameters exist (duration, URLs), then calls one of three “tool” workflow nodes.

### 1.3 Tool Dispatch (Sub-workflow trigger → Switch)
Tool calls are normalized into a standard payload and routed through a Switch node to one of the 3 Fal.run operations.

### 1.4 Fal.run Async Job Handling (Poll until COMPLETED)
Each Fal.run request returns a `request_id`. The workflow waits, polls status, loops until `COMPLETED`, then fetches the final result object containing the produced video URL and returns it to the chat.

---

## 2. Block-by-Block Analysis

## Block 1 — Chat trigger + file presence detection + image hosting
**Overview:** Accepts a chat message and optionally an uploaded image. If binary data exists, uploads it to FTP storage and builds a public URL for downstream use by the AI agent.

**Nodes involved:**
- When chat message received
- Binary?
- Upload image
- Set Image Url

### Node: **When chat message received**
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for chat interactions.
- **Configuration (interpreted):**
  - File uploads allowed
  - Allowed MIME types: `image/png,image/jpeg`
- **Key fields:**
  - Outputs `chatInput` (text)
  - Outputs binary data when a file is uploaded
- **Connections:**
  - Output → **Binary?**
- **Potential failures / edge cases:**
  - Chat UI not connected/configured in your n8n environment
  - Unsupported MIME types rejected
  - Large file uploads may be blocked by hosting limits/proxy limits

### Node: **Binary?**
- **Type / role:** `n8n-nodes-base.if` — branches on whether an upload exists.
- **Condition:** `$binary` is **not empty**
- **Connections:**
  - **True** → Upload image
  - **False** → Grok Imagine Video Agent
- **Edge cases:**
  - If the binary property name differs from `data0`, downstream nodes (FTP/Set) can fail
  - Multiple uploaded files aren’t handled explicitly; only `data0` is referenced later

### Node: **Upload image**
- **Type / role:** `n8n-nodes-base.ftp` — uploads the user’s image to an FTP server (BunnyCDN storage).
- **Configuration (interpreted):**
  - Operation: Upload
  - Remote path: `/n3wstorage/test/{{ $binary.data0.fileName }}`
  - Binary property: `data0`
  - Credential: **FTP BunnyCDN**
- **Connections:**
  - Output → Set Image Url
- **Failure modes:**
  - FTP credential/auth failure
  - Missing `binary.data0.fileName`
  - Path permissions / missing directory
  - Upload timeout for large files

### Node: **Set Image Url**
- **Type / role:** `n8n-nodes-base.set` — creates a public URL for the uploaded file.
- **Configuration:**
  - Sets `image_url = https://URL/{{ $binary.data0.fileName }}`
- **Connections:**
  - Output → Grok Imagine Video Agent
- **Important note:** `https://URL/` is a placeholder. You must replace it with your actual public CDN/base URL.
- **Edge cases:**
  - If your CDN requires subfolders, signed URLs, or different paths, the generated `image_url` will be invalid
  - If fileName contains spaces/special chars, URL encoding may be required

**Sticky note(s) applying to this block:**
- “## STEP 1 - Upload image to server”

---

## Block 2 — AI Orchestrator (LLM + memory + agent + tools)
**Overview:** Uses Grok 4.1 Fast through OpenRouter plus conversation memory to decide whether to create/edit video, gather missing parameters, and invoke the correct tool.

**Nodes involved:**
- Grok 4.1 Fast
- Simple Memory
- Grok Imagine Video Agent
- Run text to video (tool)
- Run image to video (tool)
- Run video to video (tool)

### Node: **Grok 4.1 Fast**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider node.
- **Configuration:**
  - Model: `x-ai/grok-4.1-fast`
  - Credential: **OpenRouter**
- **Connections:**
  - Provides `ai_languageModel` input to **Grok Imagine Video Agent**
- **Failure modes:**
  - Invalid/expired OpenRouter API key
  - Model name changes or account lacks access
  - Rate limits / timeouts

### Node: **Simple Memory**
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — maintains short chat history.
- **Configuration:** defaults (buffer window behavior managed by node)
- **Connections:**
  - Provides `ai_memory` input to **Grok Imagine Video Agent**
- **Edge cases:**
  - Long conversations may lose older context if window is limited (defaults depend on node version)
  - Sensitive data persistence considerations (depends on n8n execution storage)

### Node: **Grok Imagine Video Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool selection and parameter collection.
- **Configuration (interpreted):**
  - Prompt text fed to the agent:
    - `{{$json.chatInput}}`
    - `Image Url (if exist): {{$json.image_url ?? ''}}`
  - System message defines strict rules:
    - Ask which operation: text→video / image→video / edit video
    - Only call tools when required parameters are present
    - Duration must be **1–15 seconds**
    - If final output contains a URL, return it and stop
- **Connections:**
  - Receives LLM from **Grok 4.1 Fast**
  - Receives memory from **Simple Memory**
  - Can call tools:
    - Run text to video
    - Run image to video
    - Run video to video
- **Failure modes / edge cases:**
  - Ambiguous user requests → agent will ask clarifying questions
  - If user provides duration outside 1–15, agent should correct, but there is **no hard validation node** enforcing it (Fal.run may reject)
  - If `image_url` is invalid/unreachable, image-to-video will fail downstream
  - Tool call payload mismatches can break routing (depends on correct `tool_name` strings)

### Node: **Run text to video** (tool)
- **Type / role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — exposes a workflow-tool to the agent.
- **Configuration:**
  - Calls workflow: **“Video Grok Agent”** (ID `2WVikao2gvK7Z0nV`) via tool-workflow mechanism
  - Sends inputs:
    - `tool_name = "Run text to video"`
    - `query` from AI
    - `duration` from AI (number)
- **Connections:**
  - Tool output goes back to **Grok Imagine Video Agent**
- **Edge cases:**
  - The referenced workflow must exist and be accessible
  - Tool schema includes extra fields (video_url/image_url) but not required

### Node: **Run image to video** (tool)
- **Type / role:** tool workflow for image-to-video creation.
- **Configuration:**
  - `tool_name = "Run image to video"`
  - `query`, `duration`, `image_url` from AI
- **Edge cases:**
  - If user uploaded image, `image_url` comes from Set node; otherwise the agent expects user-provided URL

### Node: **Run video to video** (tool)
- **Type / role:** tool workflow for editing existing videos.
- **Configuration:**
  - `tool_name = "Run video to video"`
  - `query` and `video_url` from AI
  - `duration` hard-coded to `0` (not used by edit endpoint here)
- **Edge cases:**
  - If Fal.run endpoint later requires duration, this hard-coded `0` may become incompatible

**Sticky note(s) applying to this block:**
- “## STEP 2- Orchestrator Agents”

---

## Block 3 — Tool entrypoint + routing switch (inside the callable workflow path)
**Overview:** Receives tool invocation inputs, then routes execution to the appropriate Fal.run endpoint based on `tool_name`.

**Nodes involved:**
- Run Text-to-Video1
- Switch1

### Node: **Run Text-to-Video1**
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger` — trigger node for executions started via “Execute Workflow” / tool workflow mechanism.
- **Configuration:**
  - Declares expected inputs: `tool_name`, `query`, `duration (number)`, `video_url`, `image_url`
- **Connections:**
  - Output → Switch1
- **Edge cases:**
  - If a tool invocation omits required fields, downstream expressions may resolve to empty strings/numbers and fail at Fal.run

### Node: **Switch1**
- **Type / role:** `n8n-nodes-base.switch` — routes by `tool_name`.
- **Rules (exact string matches):**
  - `"Run text to video"` → Text to Video
  - `"Run video to video"` → Edit Video
  - `"Run image to video"` → Image to Video
- **Connections:**
  - Output 0 → Text to Video
  - Output 1 → Edit Video
  - Output 2 → Image to Video
- **Edge cases:**
  - Any mismatch in `tool_name` (case, spaces) results in no route (and the run stops with no output)
  - Consider adding a default route for error reporting (not present)

---

## Block 4 — Text-to-Video (Fal.run async)
**Overview:** Submits a text-to-video request to Fal.run, then polls status until complete and fetches the final response containing the video URL.

**Nodes involved:**
- Text to Video
- Wait 10 sec.
- Get status
- Completed?
- Get final text to video url

### Node: **Text to Video**
- **Type / role:** `n8n-nodes-base.httpRequest` — submits async job.
- **Endpoint:** `POST https://queue.fal.run/xai/grok-imagine-video/text-to-video`
- **Auth:** HTTP Header Auth credential **Fal.run API**
- **Headers:** `Content-Type: application/json`
- **Body fields:**
  - `prompt`: `{{$json.query}}`
  - `duration`: `{{$json.duration}}`
  - `aspect_ratio`: `16:9`
  - `resolution`: `720p`
- **Connections:**
  - Output → Wait 10 sec.
- **Failure modes:**
  - 401/403 due to missing/invalid Fal.run key
  - 400 if duration out of range or prompt invalid
  - Queue service transient errors/timeouts

### Node: **Wait 10 sec.**
- **Type / role:** `n8n-nodes-base.wait` — delay between polls.
- **Configuration:** 10 seconds
- **Connections:** → Get status

### Node: **Get status**
- **Type / role:** `n8n-nodes-base.httpRequest` — polls job status.
- **Endpoint:** `GET https://queue.fal.run/xai/grok-imagine-video/requests/{{ $('Text to Video').item.json.request_id }}/status`
- **Auth:** Fal.run API header auth
- **Note:** It uses a cross-node reference to **Text to Video** output `request_id`.
- **Connections:** → Completed?
- **Failure modes:**
  - If `request_id` missing (Fal.run error), URL expression breaks
  - If Fal.run returns non-JSON or schema changes, `Completed?` may fail

### Node: **Completed?**
- **Type / role:** `n8n-nodes-base.if` — checks completion.
- **Condition:** `$json.status == "COMPLETED"`
- **Connections:**
  - True → Get final text to video url
  - False → Wait 10 sec. (loop)
- **Edge cases:**
  - Other statuses like `FAILED`/`CANCELED` are not handled; it will loop forever unless Fal.run eventually returns COMPLETED

### Node: **Get final text to video url**
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches final result payload.
- **Endpoint:** `GET https://queue.fal.run/xai/grok-imagine-video/requests/{{ $json.request_id }}`
- **Auth:** Fal.run API header auth
- **Output:** typically includes the generated `video_url` (exact field depends on Fal.run response)
- **Connections:** none (ends execution path and returns output to tool caller/agent)

**Sticky note(s) applying to this block:**
- “## STEP 3 - Grok Imagine Video (Text to video)”

---

## Block 5 — Video Edit (video-to-video) (Fal.run async)
**Overview:** Submits an edit request with `video_url` + prompt, then polls until complete and fetches the final edited video URL.

**Nodes involved:**
- Edit Video
- Wait 10 sec.1
- Get status1
- Completed?1
- Get final edit video url

### Node: **Edit Video**
- **Type / role:** HTTP request to start edit job.
- **Endpoint:** `POST https://queue.fal.run/xai/grok-imagine-video/edit-video`
- **Body fields:**
  - `prompt`: `{{$json.query}}`
  - `video_url`: `{{$json.video_url}}`
  - `resolution`: `auto`
- **Connections:** → Wait 10 sec.1
- **Failure modes:**
  - Invalid/unreachable `video_url`
  - Fal.run validation errors
  - Auth issues

### Node: **Wait 10 sec.1**
- **Type / role:** Wait node
- **Configuration:** 10 seconds
- **Connections:** → Get status1

### Node: **Get status1**
- **Type / role:** poll status
- **Endpoint:** `GET .../requests/{{ $('Edit Video').item.json.request_id }}/status`
- **Connections:** → Completed?1

### Node: **Completed?1**
- **Type / role:** if status is COMPLETED
- **Connections:**
  - True → Get final edit video url
  - False → Wait 10 sec.1 (loop)
- **Edge cases:** same infinite loop risk on `FAILED` states

### Node: **Get final edit video url**
- **Type / role:** fetch final output
- **Endpoint:** `GET .../requests/{{ $json.request_id }}`
- **Connections:** none

**Sticky note(s) applying to this block:**
- “## STEP 4 - Grok Imagine Video (Edit video)”

---

## Block 6 — Image-to-Video (Fal.run async)
**Overview:** Submits an image-to-video request with `image_url` + prompt, then polls until complete and fetches final video URL.

**Nodes involved:**
- Image to Video
- Wait 10 sec.2
- Get status2
- Completed?2
- Get final image to video url

### Node: **Image to Video**
- **Type / role:** HTTP request to create video from image.
- **Endpoint:** `POST https://queue.fal.run/xai/grok-imagine-video/image-to-video`
- **Body fields:**
  - `prompt`: `{{$json.query}}`
  - `duration`: `{{$json.duration}}`
  - `aspect_ratio`: `16:9`
  - `resolution`: `720p`
  - `image_url`: `{{$json.image_url}}`
- **Connections:** → Wait 10 sec.2
- **Failure modes:**
  - Invalid `image_url` / not publicly accessible
  - Duration invalid
  - Auth/rate limit issues

### Node: **Wait 10 sec.2**
- **Type / role:** Wait node for polling interval
- **Configuration:** **30 seconds** (despite the name)
- **Connections:** → Get status2

### Node: **Get status2**
- **Type / role:** poll status
- **Endpoint:** `GET .../requests/{{ $('Image to Video').item.json.request_id }}/status`
- **Connections:** → Completed?2

### Node: **Completed?2**
- **Type / role:** completion check
- **Connections:**
  - True → Get final image to video url
  - False → Wait 10 sec.2 (loop)

### Node: **Get final image to video url**
- **Type / role:** fetch final output
- **Endpoint:** `GET .../requests/{{ $json.request_id }}`
- **Connections:** none

**Sticky note(s) applying to this block:**
- “## STEP 5 - Grok Imagine Video (Image to Video)”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat entrypoint + optional file upload intake | — | Binary? | (from note) Grok Imagine Video Chatbot: Generate & Modify Videos via Natural Language… |
| Binary? | `n8n-nodes-base.if` | Branch if an uploaded image exists | When chat message received | Upload image (true), Grok Imagine Video Agent (false) | ## STEP 1 - Upload image to server |
| Upload image | `n8n-nodes-base.ftp` | Upload user image to FTP/CDN storage | Binary? (true) | Set Image Url | ## STEP 1 - Upload image to server |
| Set Image Url | `n8n-nodes-base.set` | Build public image URL from uploaded file name | Upload image | Grok Imagine Video Agent | ## STEP 1 - Upload image to server |
| Grok 4.1 Fast | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | LLM provider (OpenRouter Grok) | — | Grok Imagine Video Agent (ai_languageModel) | ## STEP 2- Orchestrator Agents |
| Simple Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Conversation memory | — | Grok Imagine Video Agent (ai_memory) | ## STEP 2- Orchestrator Agents |
| Grok Imagine Video Agent | `@n8n/n8n-nodes-langchain.agent` | Intent detection, parameter gathering, tool calling | Binary? (false) / Set Image Url / LLM / Memory | (tool calls) | ## STEP 2- Orchestrator Agents |
| Run text to video | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool wrapper: create video from text | Grok Imagine Video Agent | Grok Imagine Video Agent | ## STEP 2- Orchestrator Agents |
| Run image to video | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool wrapper: create video from image | Grok Imagine Video Agent | Grok Imagine Video Agent | ## STEP 2- Orchestrator Agents |
| Run video to video | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool wrapper: edit existing video | Grok Imagine Video Agent | Grok Imagine Video Agent | ## STEP 2- Orchestrator Agents |
| Run Text-to-Video1 | `n8n-nodes-base.executeWorkflowTrigger` | Entrypoint for tool executions (receives inputs) | — | Switch1 |  |
| Switch1 | `n8n-nodes-base.switch` | Route by tool_name to correct Fal.run operation | Run Text-to-Video1 | Text to Video / Edit Video / Image to Video |  |
| Text to Video | `n8n-nodes-base.httpRequest` | Start Fal.run text-to-video job | Switch1 | Wait 10 sec. | ## STEP 3 - Grok Imagine Video (Text to video) |
| Wait 10 sec. | `n8n-nodes-base.wait` | Polling delay (text-to-video) | Text to Video / Completed? (false) | Get status | ## STEP 3 - Grok Imagine Video (Text to video) |
| Get status | `n8n-nodes-base.httpRequest` | Poll status for text-to-video job | Wait 10 sec. | Completed? | ## STEP 3 - Grok Imagine Video (Text to video) |
| Completed? | `n8n-nodes-base.if` | Check COMPLETED for text-to-video | Get status | Get final text to video url (true) / Wait 10 sec. (false) | ## STEP 3 - Grok Imagine Video (Text to video) |
| Get final text to video url | `n8n-nodes-base.httpRequest` | Fetch final text-to-video result | Completed? (true) | — | ## STEP 3 - Grok Imagine Video (Text to video) |
| Edit Video | `n8n-nodes-base.httpRequest` | Start Fal.run edit-video job | Switch1 | Wait 10 sec.1 | ## STEP 4 - Grok Imagine Video (Edit video) |
| Wait 10 sec.1 | `n8n-nodes-base.wait` | Polling delay (edit video) | Edit Video / Completed?1 (false) | Get status1 | ## STEP 4 - Grok Imagine Video (Edit video) |
| Get status1 | `n8n-nodes-base.httpRequest` | Poll status for edit-video job | Wait 10 sec.1 | Completed?1 | ## STEP 4 - Grok Imagine Video (Edit video) |
| Completed?1 | `n8n-nodes-base.if` | Check COMPLETED for edit-video | Get status1 | Get final edit video url (true) / Wait 10 sec.1 (false) | ## STEP 4 - Grok Imagine Video (Edit video) |
| Get final edit video url | `n8n-nodes-base.httpRequest` | Fetch final edit-video result | Completed?1 (true) | — | ## STEP 4 - Grok Imagine Video (Edit video) |
| Image to Video | `n8n-nodes-base.httpRequest` | Start Fal.run image-to-video job | Switch1 | Wait 10 sec.2 | ## STEP 5 - Grok Imagine Video (Image to Video) |
| Wait 10 sec.2 | `n8n-nodes-base.wait` | Polling delay (image-to-video) set to 30s | Image to Video / Completed?2 (false) | Get status2 | ## STEP 5 - Grok Imagine Video (Image to Video) |
| Get status2 | `n8n-nodes-base.httpRequest` | Poll status for image-to-video job | Wait 10 sec.2 | Completed?2 | ## STEP 5 - Grok Imagine Video (Image to Video) |
| Completed?2 | `n8n-nodes-base.if` | Check COMPLETED for image-to-video | Get status2 | Get final image to video url (true) / Wait 10 sec.2 (false) | ## STEP 5 - Grok Imagine Video (Image to Video) |
| Get final image to video url | `n8n-nodes-base.httpRequest` | Fetch final image-to-video result | Completed?2 (true) | — | ## STEP 5 - Grok Imagine Video (Image to Video) |
| Sticky Note | `n8n-nodes-base.stickyNote` | Comment block | — | — | ## STEP 1 - Upload image to server |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Comment block | — | — | ## STEP 2- Orchestrator Agents |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Comment block | — | — | ## STEP 3 - Grok Imagine Video (Text to video) |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Comment block | — | — | ## STEP 4 - Grok Imagine Video (Edit video) |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Comment block | — | — | ## STEP 5 - Grok Imagine Video (Image to Video) |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Comment block | — | — | Grok Imagine Video Chatbot: Generate & Modify Videos via Natural Language… |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Comment block | — | — | ## MY NEW YOUTUBE CHANNEL — https://youtube.com/@n3witalia |

---

## 4. Reproducing the Workflow from Scratch

### Credentials you must create first
1. **OpenRouter API credential** (n8n: OpenRouter)
   - Add your OpenRouter API key
   - Ensure access to model `x-ai/grok-4.1-fast`
2. **Fal.run API credential** (n8n: HTTP Header Auth)
   - Header typically: `Authorization: Key <FAL_KEY>` (or per your Fal.run account format)
   - Name it e.g. **Fal.run API**
3. **FTP credential** (n8n: FTP)
   - Host, user, password for your BunnyCDN storage (or any FTP host)
   - Name it e.g. **FTP BunnyCDN**

### Build steps (node-by-node)
1. Create **When chat message received** (Chat Trigger)
   - Enable file uploads
   - Allowed MIME types: `image/png,image/jpeg`
2. Add **Binary?** (IF)
   - Condition: `{{$binary}}` → “not empty”
   - Connect: Chat Trigger → Binary?
3. True branch image handling:
   1. Add **Upload image** (FTP)
      - Operation: Upload
      - Binary property: `data0`
      - Path: `/n3wstorage/test/{{ $binary.data0.fileName }}`
      - Select FTP credential
   2. Add **Set Image Url** (Set)
      - Add field `image_url` (string)
      - Value: `https://URL/{{ $binary.data0.fileName }}`
      - Replace `https://URL/` with your real public base URL
   3. Connect: Binary? (true) → Upload image → Set Image Url
4. Add AI components:
   1. **Grok 4.1 Fast** (OpenRouter Chat Model)
      - Model: `x-ai/grok-4.1-fast`
      - Credential: OpenRouter
   2. **Simple Memory** (Memory Buffer Window)
      - Defaults
   3. **Grok Imagine Video Agent** (LangChain Agent)
      - Prompt/input text:
        - `{{$json.chatInput}}`
        - plus `Image Url (if exist): {{$json.image_url ?? ''}}`
      - System message: copy the ruleset (operation selection, required params, duration 1–15, stop on final URL)
      - Attach:
        - `ai_languageModel` from Grok 4.1 Fast
        - `ai_memory` from Simple Memory
5. Connect chat flow to agent:
   - Binary? (false) → Grok Imagine Video Agent
   - Set Image Url → Grok Imagine Video Agent
6. Create 3 **Tool Workflow** nodes (LangChain toolWorkflow):
   - **Run text to video**
     - Tool name output field: `Run text to video`
     - Inputs: `query` (string), `duration` (number)
     - Set `tool_name` constant to “Run text to video”
     - Point to your callable workflow (see step 7–12)
   - **Run image to video**
     - Inputs: `query`, `duration`, `image_url`
     - `tool_name` constant “Run image to video”
   - **Run video to video**
     - Inputs: `query`, `video_url`
     - `tool_name` constant “Run video to video”
     - `duration` can be 0 (as in JSON)
7. Connect each tool node to the agent as **ai_tool**.

### Create the callable (tool-executed) workflow logic
This is the workflow that the tool nodes invoke (in your JSON it’s referenced as “Video Grok Agent”). Build it as follows:

8. Add **Execute Workflow Trigger**
   - Define inputs: `tool_name`, `query`, `duration (number)`, `video_url`, `image_url`
9. Add **Switch** node
   - Switch on: `{{$json.tool_name}}`
   - Rules (equals):
     - “Run text to video” → Output 1
     - “Run video to video” → Output 2
     - “Run image to video” → Output 3
10. Build **Text-to-video path**
   1. HTTP Request “Text to Video”
      - POST `https://queue.fal.run/xai/grok-imagine-video/text-to-video`
      - JSON body: prompt, duration, aspect_ratio 16:9, resolution 720p
      - Fal.run header auth credential
   2. Wait “Wait 10 sec.” (10 seconds)
   3. HTTP Request “Get status”
      - GET `.../requests/{{ $('Text to Video').item.json.request_id }}/status`
   4. IF “Completed?”
      - If `{{$json.status}} == COMPLETED`
      - True → HTTP Request “Get final text to video url” (GET `.../requests/{{ $json.request_id }}`)
      - False → back to Wait
11. Build **Edit-video path**
   1. HTTP Request “Edit Video”
      - POST `https://queue.fal.run/xai/grok-imagine-video/edit-video`
      - JSON body: prompt + video_url + resolution auto
   2. Wait 10 seconds
   3. Poll status using request_id from “Edit Video”
   4. IF completed → fetch final with `/requests/{{request_id}}` else loop
12. Build **Image-to-video path**
   1. HTTP Request “Image to Video”
      - POST `https://queue.fal.run/xai/grok-imagine-video/image-to-video`
      - JSON body includes `image_url`
   2. Wait **30 seconds**
   3. Poll status using request_id from “Image to Video”
   4. IF completed → fetch final else loop
13. In your **main chat workflow**, set each Tool Workflow node to execute this callable workflow, and map inputs accordingly.

**Recommended hardening (optional but important):**
- Add handling for `FAILED` / `CANCELED` statuses to stop looping and return an error message.
- Add numeric validation node (or agent-side + IF guard) enforcing duration 1–15 before calling Fal.run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Grok Imagine Video Chatbot: Generate & Modify Videos via Natural Language…” (full description in sticky note) | Workflow overview and setup guidance embedded in canvas |
| “## MY NEW YOUTUBE CHANNEL 👉 Subscribe…” | https://youtube.com/@n3witalia |
| Image preview link in note | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |

