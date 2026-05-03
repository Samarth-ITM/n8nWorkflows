Transform meeting transcripts to content with Fathom, GPT-4 & DALL-E

https://n8nworkflows.xyz/workflows/transform-meeting-transcripts-to-content-with-fathom--gpt-4---dall-e-12009


# Transform meeting transcripts to content with Fathom, GPT-4 & DALL-E

## 1. Workflow Overview

**Purpose:** This n8n workflow turns the *latest* recorded Fathom meeting (coaching/consulting session) into a multi-format content pack—**written post + Google Doc (always)**, optionally **video** and/or **image**—using **LangChain Agents** powered by **GPT-4.1-mini**, plus external generation services (OpenAI Images API, and a third-party video API).

**Typical use cases:**
- Repurpose coaching calls into LinkedIn posts, social graphics, and short recap videos.
- Automate content production from recorded sessions with minimal manual effort via a chat interface.

### Logical blocks
**1.1 Chat entry + orchestration (main workflow)**  
Chat Trigger → Orchestrator Agent (with shared GPT model + memory) → calls tools (Fathom + Content Generator).

**1.2 Content generation (agent tool routing, main workflow)**  
Content Post Generator Agent Tool → calls sub-tools: Transcript-to-Content, Video Generator, Image Generator.

**1.3 Video prompt + video generation (mixed: main workflow + subworkflow placeholder + concrete video API flow)**  
Video Generator Agent Tool → “Text to Video” subworkflow tool (must be linked).  
Additionally, the JSON contains a *separate execute-workflow entry block* that directly calls a video API (kie.ai) and posts to Slack.

**1.4 Image prompt + image generation (mixed: main workflow + concrete OpenAI image generation flow)**  
Image Generator Agent Tool → “Text to Image” subworkflow tool (must be linked).  
Additionally, the JSON contains a concrete OpenAI Images API → binary conversion → Google Drive upload chain.

**1.5 Subworkflow execution entry (executeWorkflowTrigger block)**  
A dedicated “Subworkflow Entry Point” receives inputs like `post_title`, `post_content`, and `input.prompt`, then fans out to Google Docs creation + image generation + video task creation.

> Important: This template mixes “tool subworkflow placeholders” (Text to Video/Text to Image/Transcript to Content) **and** an embedded “executeWorkflowTrigger” pipeline that already implements Google Docs + DALL·E + kie.ai video. You’ll likely choose one consistent approach (all via tool subworkflows, or all via embedded subworkflow logic).

---

## 2. Block-by-Block Analysis

### Block 1 — Chat Entry & Top-Level Orchestration

**Overview:** Receives a chat message, keeps short conversational context, and uses an orchestrator agent to decide what assets to produce and to fetch the latest Fathom transcript first.

**Nodes involved:**
- When chat message received
- Main GPT-4 Model
- Simple Memory
- Content Orchestrator
- Get Fathom Transcript (tool)

#### Node: **When chat message received**
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` (Chat entry trigger).
- **Config (interpreted):** Listens for chat messages in n8n’s chat interface; no special options configured.
- **Inputs/Outputs:**  
  - Output → Content Orchestrator (main)
- **Version notes:** typeVersion **1.3**
- **Failure/edge cases:** Chat not enabled in instance; webhook/channel misconfigured; empty user messages.

#### Node: **Simple Memory**
- **Type / role:** `memoryBufferWindow` (short-term conversation memory for the agent).
- **Config:** `contextWindowLength: 2` (keeps last 2 turns).
- **Connections:**  
  - AI memory → Content Orchestrator
- **Version notes:** typeVersion **1.3**
- **Edge cases:** If multiple concurrent chats share state unexpectedly (depends on n8n chat context handling); limited memory may cause “forgotten” constraints.

#### Node: **Main GPT-4 Model**
- **Type / role:** `lmChatOpenAi` (shared language model for agents).
- **Config:** Model = **gpt-4.1-mini**; default options.
- **Credentials:** OpenAI API credential required.
- **Connections:**  
  - AI language model → Content Orchestrator  
  - AI language model → Content Post Generator
- **Version notes:** typeVersion **1.2**
- **Edge cases:** OpenAI auth/quota errors; model name not available in account/region; timeouts under heavy prompts.

#### Node: **Content Orchestrator**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (top-level agent that routes the request).
- **Config highlights:** System message enforces:
  - Determine intent: video/image/both + requirements
  - **Must** call “Get Fathom Transcript” first
  - Then call “Content Post Generator”
  - Ask clarification if request unclear
- **Connections:**
  - Input (main) from Chat Trigger
  - AI model from Main GPT-4 Model
  - AI memory from Simple Memory
  - AI tool: Get Fathom Transcript
  - AI tool: Content Post Generator
- **Version notes:** typeVersion **2.2**
- **Failure/edge cases:** Tool failures (Fathom down/auth); orchestrator may mis-route if user message ambiguous; token limits if transcript/summary is huge (depends what Fathom returns).

#### Node: **Get Fathom Transcript**
- **Type / role:** `n8n-nodes-base.httpRequestTool` (LangChain tool wrapper around HTTP request).
- **Config:**
  - GET `https://api.fathom.ai/external/v1/meetings`
  - Query params include: `include_summary=true`, `limit=1`, plus `calendar_invitees`, `teams` (names included but values not explicitly set in JSON)
  - Intended behavior: retrieve most recent meeting in last 7 days (note: the actual “past 7 days” filtering is described in prompts but **not** implemented via query parameters here; you may need additional parameters supported by Fathom API if required).
- **Connections:** Tool available to Content Orchestrator.
- **Version notes:** typeVersion **4.2**
- **Edge cases/failures:**
  - Missing auth headers/credentials (none shown in JSON) → 401
  - API pagination/empty items → agent must handle no meetings found
  - Response schema changes (meeting fields missing)
  - Rate limits

---

### Block 2 — Content Post Generator (Coordinator Tool Agent)

**Overview:** Receives user instructions + session data (from orchestrator) and always produces written content (via Transcript-to-Content tool), optionally video and image via specialist tool agents.

**Nodes involved:**
- Content Post Generator
- Transcript to Content (tool subworkflow placeholder)
- Video Generator (tool agent)
- Image Generator (tool agent)

#### Node: **Content Post Generator**
- **Type / role:** `agentTool` (a tool-like agent called by the orchestrator).
- **Config:**
  - `text` is sourced from `$fromAI('Prompt__User_Message_')` (n8n AI override mechanism).
  - System message rules:
    - Always run Transcript to Content
    - Run Video Generator and/or Image Generator depending on request
    - Run tasks “in parallel when possible”
    - Return compiled deliverables (Doc link + optional URLs)
- **Connections:**
  - AI model from Main GPT-4 Model
  - Tools available:
    - Transcript to Content (toolWorkflow)
    - Video Generator (agentTool)
    - Image Generator (agentTool)
- **Version notes:** typeVersion **2.2**
- **Edge cases:** If orchestrator doesn’t pass session payload cleanly, agent may lack transcript; tool failures must be surfaced; parallelism is conceptual—n8n agent tools may still run sequentially depending on implementation.

#### Node: **Transcript to Content**
- **Type / role:** `toolWorkflow` (calls a separate n8n workflow).
- **Config:** Workflow ID is **blank** in JSON (`value: ""`) and must be set to an existing subworkflow. Description says it:
  - Takes transcript + summary
  - Produces structured post + creates Google Doc
  - Returns formatted content + doc link
- **Connections:** Tool for Content Post Generator.
- **Version notes:** typeVersion **2.2**
- **Edge cases:** Not linked → tool unusable; subworkflow must expose expected inputs/outputs; Google auth issues in subworkflow.

---

### Block 3 — Video Generation (Prompt + Subworkflow placeholder)

**Overview:** A specialist agent transforms session summary into a detailed 300–500 word video prompt, then calls a Text-to-Video subworkflow to generate the video.

**Nodes involved:**
- Video Generator
- Video Generator GPT Model
- Text to Video (tool subworkflow placeholder)

#### Node: **Video Generator**
- **Type / role:** `agentTool` (specialist tool agent).
- **Config highlights:**
  - System message is strict: **output only the video prompt text**, no formatting.
  - Requires fetching Fathom transcript via tool (it’s mentioned in prompt, but in this workflow the Video Generator node itself does **not** have Get Fathom Transcript connected as a tool; only the Orchestrator has it. In practice, the Content Post Generator should pass the session data down, otherwise this agent may not be able to retrieve it itself.)
- **Connections:**
  - AI model from **Video Generator GPT Model**
  - Tool: Text to Video
  - Tool used by Content Post Generator
- **Version notes:** typeVersion **2.2**
- **Edge cases:** If session data isn’t passed in, the agent prompt expects it can “use Get Fathom Transcript” but it cannot (tool not connected) → hallucination risk or failure. Consider adding Get Fathom Transcript tool here too, or ensure upstream passes data.

#### Node: **Video Generator GPT Model**
- **Type / role:** `lmChatOpenAi` dedicated model feed for Video/Image specialist agents.
- **Config:** gpt-4.1-mini
- **Credentials:** OpenAI API
- **Connections:** AI language model → Video Generator and Image Generator
- **Version notes:** typeVersion **1.2**
- **Edge cases:** Same as other OpenAI nodes.

#### Node: **Text to Video**
- **Type / role:** `toolWorkflow` (subworkflow call placeholder).
- **Config:** workflowId blank; must be linked. Description suggests it calls Luma/Runway etc and returns a video file/URL.
- **Connections:** Tool available to Video Generator.
- **Version notes:** typeVersion **2.2**
- **Edge cases:** Not linked; subworkflow outputs not matching what upstream expects (e.g., missing `videoUrl`).

---

### Block 4 — Image Generation (Prompt + Subworkflow placeholder)

**Overview:** A specialist agent extracts a key insight and produces a 100–150 word image prompt for a square social graphic, then calls a Text-to-Image subworkflow to generate the asset.

**Nodes involved:**
- Image Generator
- Video Generator GPT Model
- Text to Image (tool subworkflow placeholder)

#### Node: **Image Generator**
- **Type / role:** `agentTool` (specialist tool agent).
- **Config highlights:** System prompt:
  - Find most visual “aha moment”
  - Produce a 100–150 word prompt including style, layout, text overlays, 1:1 ratio
  - Call Text to Image and return final image URL/file
- **Connections:**
  - AI model from Video Generator GPT Model
  - Tool: Text to Image
  - Tool used by Content Post Generator
- **Version notes:** typeVersion **2.2**
- **Edge cases:** Same dependency on upstream passing session data; output expectations depend on Text-to-Image subworkflow.

#### Node: **Text to Image**
- **Type / role:** `toolWorkflow` placeholder.
- **Config:** workflowId blank; must be linked. Intended to call DALL·E or Stability.
- **Connections:** Tool available to Image Generator.
- **Version notes:** typeVersion **2.2**
- **Edge cases:** Not linked; image generation provider limits; large prompts; content policy rejections.

---

### Block 5 — Embedded Subworkflow Execution Entry (Google Doc + DALL·E + Video API + Slack)

**Overview:** This is a separate execution path triggered when another workflow calls it (Execute Workflow Trigger). It creates a Google Doc, generates an image via OpenAI Images API and uploads to Google Drive, and starts a video generation task via kie.ai then posts a Slack message when the video is ready.

**Nodes involved:**
- Subworkflow Entry Point
- Create Google Doc
- Insert Content into Doc
- Call DALL-E API
- Convert Image to Binary
- Upload to Storage
- Format Image Link
- Call Video API
- Fetch Generated Video
- Format Video Link
- Video Ready

#### Node: **Subworkflow Entry Point**
- **Type / role:** `executeWorkflowTrigger` (entry for being called as a subworkflow).
- **Config:** Declares workflow inputs:
  - `post_title`, `post_content`, `original_meeting_title`, `original_meeting_date`, `prompt`, `input.prompt`
- **Connections:** Fans out (in parallel) to:
  - Create Google Doc
  - Call DALL-E API
  - Call Video API
- **Version notes:** typeVersion **1.1**
- **Edge cases:** If caller doesn’t supply required fields, downstream expressions fail (e.g., missing `post_title`).

#### Node: **Create Google Doc**
- **Type / role:** `googleDocs` (create a new doc).
- **Config:**
  - Title from `$json.post_title`
  - Folder ID fixed: `15i8eJEE9WN6G6yJaUzy5JdxJob-eo7Bm`
- **Connections:** → Insert Content into Doc
- **Credentials:** Not shown in JSON; requires Google Docs OAuth2 credentials configured in n8n.
- **Version notes:** typeVersion **2**
- **Edge cases:** Permission denied to folder; missing title; Drive API limits.

#### Node: **Insert Content into Doc**
- **Type / role:** `googleDocs` (update doc; insert text).
- **Config:**
  - Operation update/insert action
  - Text expression: `{{ $('When Executed by Another Workflow').item.json.post_content }}`
    - Note: This references a node name **that does not exist** in this workflow. It likely should reference **Subworkflow Entry Point** instead.
  - Document URL uses created doc id: `{{ $json.id }}`
- **Connections:** none shown beyond doc creation.
- **Version notes:** typeVersion **2**
- **Edge cases:** This is a critical expression bug: the node reference will fail at runtime unless renamed or corrected.

#### Node: **Call DALL-E API**
- **Type / role:** `httpRequest` to OpenAI Images API.
- **Config:**
  - POST `https://api.openai.com/v1/images/generations`
  - Body: `model=gpt-image-1`, `prompt={{ $json.output }}`, `size=1024x1024`
  - Headers: `sendHeaders: true` but header parameters are effectively empty in JSON; you must set `Authorization: Bearer <OPENAI_API_KEY>` (or use n8n OpenAI credentials via dedicated node).
- **Connections:** → Convert Image to Binary
- **Version notes:** typeVersion **4.2**
- **Edge cases:** Missing auth header; OpenAI content policy blocks; large prompt; response shape differences.

#### Node: **Convert Image to Binary**
- **Type / role:** `convertToFile` (turn base64 into binary file).
- **Config:**
  - `sourceProperty: data[0].b64_json`
- **Connections:** → Upload to Storage
- **Version notes:** typeVersion **1.1**
- **Edge cases:** If OpenAI response is not `b64_json` (e.g., URL response), conversion fails.

#### Node: **Upload to Storage**
- **Type / role:** `googleDrive` upload file.
- **Config:**
  - Drive: “My Drive”
  - Folder: `root`
  - File name: `"="` (this looks wrong; likely intended to be an expression but currently is just equals sign).
- **Credentials:** Google Drive OAuth2 credential is configured.
- **Connections:** → Format Image Link
- **Version notes:** typeVersion **3**
- **Edge cases:** Bad filename; upload fails if binary property not detected; permissions/quota.

#### Node: **Format Image Link**
- **Type / role:** `set`
- **Config:** Sets `videoURL` to `JSON.parse($json.data.resultJson).resultUrls[0]`
- **Connections:** none further in JSON.
- **Issue:** The node is named “Format Image Link” but sets `videoURL` from a `resultJson` structure typically associated with the video API, not Google Drive upload. This looks like a copy/paste mistake.
- **Version notes:** typeVersion **3.4**
- **Edge cases:** `data.resultJson` missing → JSON.parse fails.

#### Node: **Call Video API**
- **Type / role:** `httpRequest` to kie.ai create task.
- **Config:**
  - POST `https://api.kie.ai/api/v1/jobs/createTask`
  - Body includes:
    - `model=sora-2-pro-text-to-video`
    - `input.prompt={{ $json.output }}`
    - `input.aspect_ratio=landscape`, `input.n_frames=10`, `input.size=high`
- **Connections:** → Fetch Generated Video
- **Version notes:** typeVersion **4.2**
- **Edge cases:** Missing auth headers (none shown); provider downtime; invalid model name; long queue.

#### Node: **Fetch Generated Video**
- **Type / role:** `httpRequest` polling/lookup.
- **Config:** GET `https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json.data.taskId }}`
- **Connections:** → Format Video Link
- **Version notes:** typeVersion **4.2**
- **Edge cases:** No waiting/retry loop present—if the record isn’t ready immediately, you’ll get “processing” and no `resultJson`. Typically you need a Wait + retry.

#### Node: **Format Video Link**
- **Type / role:** `set`
- **Config:** `videoURL = JSON.parse($json.data.resultJson).resultUrls[0]`
- **Connections:** none further in JSON (but Slack node uses `$json.videoURL`; it is not connected here).
- **Version notes:** typeVersion **3.4**
- **Edge cases:** `resultJson` absent until job completes; JSON.parse failure.

#### Node: **Video Ready**
- **Type / role:** `slack` message to a channel.
- **Config:**
  - Posts to channel ID `C09G17879L1`
  - Message includes Markdown link using `{{ $json.videoURL }}`
  - OAuth2 auth
- **Connections:** No incoming connection in JSON → node will never run unless connected.
- **Version notes:** typeVersion **2.3**
- **Edge cases:** Slack auth revoked; missing `videoURL`; channel access denied.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | LangChain Chat Trigger | Entry point from n8n chat | — | Content Orchestrator | ## 1. Chat Interface Entry Point  \nhttps://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chattrigger/ |
| Content Orchestrator | LangChain Agent | Interprets request, fetches transcript, delegates | When chat message received; Main GPT-4 Model (AI); Simple Memory (AI) | (Tool calls) Content Post Generator; Get Fathom Transcript | ## 2. AI Content Orchestrator  \nhttps://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/ |
| Main GPT-4 Model | OpenAI Chat Model (LangChain) | LLM for Orchestrator + Content Generator | — | Content Orchestrator (AI); Content Post Generator (AI) | ## Memory & AI Models  \nThe workflow uses GPT-4.1-mini... |
| Simple Memory | Buffer Window Memory | Keeps last 2 messages | — | Content Orchestrator (AI memory) | ## Memory & AI Models  \nThe workflow uses GPT-4.1-mini... |
| Content Post Generator | Tool Agent (LangChain) | Produces written content + coordinates video/image | Main GPT-4 Model (AI); (called by) Content Orchestrator | (Tool calls) Transcript to Content; Video Generator; Image Generator | ## 3. Content Post Generator  \nhttps://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.agenttool/ |
| Get Fathom Transcript | HTTP Request Tool | Fetch latest Fathom meeting data | — | Content Orchestrator (tool) |  |
| Transcript to Content | Tool Workflow | Calls subworkflow to write post + doc | — | Content Post Generator (tool) | ### ⚠️ Required Subworkflows Setup!  \nBefore using this template, you MUST create three separate subworkflows: ... |
| Video Generator | Tool Agent (LangChain) | Creates video prompt and triggers video subworkflow | Video Generator GPT Model (AI) | Content Post Generator (tool); Text to Video (tool) | ## 4. Video Generation Pipeline  \nhttps://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolworkflow/ |
| Video Generator GPT Model | OpenAI Chat Model (LangChain) | LLM for video/image specialist agents | — | Video Generator (AI); Image Generator (AI) | ## Memory & AI Models  \nThe workflow uses GPT-4.1-mini... |
| Text to Video | Tool Workflow | Calls external subworkflow to generate video | — | Video Generator (tool) | ### ⚠️ Required Subworkflows Setup!  \nBefore using this template, you MUST create three separate subworkflows: ... |
| Image Generator | Tool Agent (LangChain) | Creates image prompt and triggers image subworkflow | Video Generator GPT Model (AI) | Content Post Generator (tool); Text to Image (tool) | ## 5. Image Generation Pipeline  \nhttps://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolworkflow/ |
| Text to Image | Tool Workflow | Calls external subworkflow to generate image | — | Image Generator (tool) | ### ⚠️ Required Subworkflows Setup!  \nBefore using this template, you MUST create three separate subworkflows: ... |
| Subworkflow Entry Point | Execute Workflow Trigger | Entry for being called by another workflow | — | Create Google Doc; Call DALL-E API; Call Video API |  |
| Create Google Doc | Google Docs | Creates doc in specified folder | Subworkflow Entry Point | Insert Content into Doc |  |
| Insert Content into Doc | Google Docs | Inserts post content into created doc | Create Google Doc | — |  |
| Call DALL-E API | HTTP Request | Generates image via OpenAI Images API | Subworkflow Entry Point | Convert Image to Binary |  |
| Convert Image to Binary | Convert to File | Converts base64 image to binary | Call DALL-E API | Upload to Storage |  |
| Upload to Storage | Google Drive | Uploads generated image file | Convert Image to Binary | Format Image Link |  |
| Format Image Link | Set | (Misconfigured) sets `videoURL` from `resultJson` | Upload to Storage | — |  |
| Call Video API | HTTP Request | Creates video generation job (kie.ai) | Subworkflow Entry Point | Fetch Generated Video |  |
| Fetch Generated Video | HTTP Request | Fetches job record info by taskId | Call Video API | Format Video Link |  |
| Format Video Link | Set | Extracts `videoURL` from `resultJson` | Fetch Generated Video | — |  |
| Video Ready | Slack | Posts final video link to Slack channel | (none connected) | — |  |
| Main Overview | Sticky Note | Documentation / overview | — | — | ## 🎥 AI Content Generator: Transcript to Video & Image  \n(contains Discord + Forum links) |
| Subworkflow Setup Warning | Sticky Note | Warns to create 3 subworkflows | — | — | ### ⚠️ Required Subworkflows Setup! ... |
| Step 1 - Chat Trigger | Sticky Note | Explains chat entry | — | — | (same content as note) |
| Step 2 - Orchestrator | Sticky Note | Explains orchestrator | — | — | (same content as note) |
| Step 3 - Content Generator | Sticky Note | Explains content generator | — | — | (same content as note) |
| Step 4 - Video Pipeline | Sticky Note | Explains video pipeline | — | — | (same content as note) |
| Step 5 - Image Pipeline | Sticky Note | Explains image pipeline | — | — | (same content as note) |
| Memory Note | Sticky Note | Notes model + memory | — | — | (same content as note) |

---

## 4. Reproducing the Workflow from Scratch

### A) Main workflow (chat-driven, agent-based)

1. **Create “When chat message received” (Chat Trigger)**
   - Node: *LangChain Chat Trigger*
   - Leave options default.
   - This is your main entry point.

2. **Create “Main GPT-4 Model” (OpenAI Chat Model)**
   - Node: *LangChain → OpenAI Chat Model*
   - Model: `gpt-4.1-mini`
   - Configure **OpenAI API credentials** in n8n and select them here.

3. **Create “Simple Memory”**
   - Node: *LangChain → Memory Buffer Window*
   - Set **Context Window Length = 2**

4. **Create “Get Fathom Transcript” tool**
   - Node: *HTTP Request Tool*
   - URL: `https://api.fathom.ai/external/v1/meetings`
   - Enable **Send Query Parameters**
   - Add:
     - `include_summary = true`
     - `limit = 1`
     - (Add any Fathom-supported parameters for invitees/teams as required by their API docs)
   - Add authentication (typically **Bearer token** header) according to your Fathom External API requirements.

5. **Create “Content Orchestrator”**
   - Node: *LangChain → AI Agent*
   - Paste the provided orchestrator system message (from the JSON).
   - Connect:
     - Chat Trigger → Content Orchestrator (main)
     - Main GPT-4 Model → Content Orchestrator (AI languageModel)
     - Simple Memory → Content Orchestrator (AI memory)
     - Get Fathom Transcript → Content Orchestrator (AI tool)

6. **Create “Content Post Generator” (Tool Agent)**
   - Node: *LangChain → Tool Agent*
   - Set its system message (from JSON).
   - Ensure its **Text/Input** uses the `$fromAI('Prompt__User_Message_')` pattern (or replace with a simpler mapping if you prefer standard fields).
   - Connect:
     - Main GPT-4 Model → Content Post Generator (AI languageModel)
     - Content Post Generator → Content Orchestrator (AI tool connection)

7. **Create “Transcript to Content” (Tool Workflow placeholder)**
   - Node: *LangChain → Tool Workflow*
   - Select the subworkflow (once created) in **Workflow ID**
   - Connect it as an **AI tool** to Content Post Generator.

8. **Create “Video Generator GPT Model”**
   - Node: *LangChain → OpenAI Chat Model*
   - Model: `gpt-4.1-mini`
   - Same OpenAI credentials.

9. **Create “Video Generator” (Tool Agent)**
   - Node: *LangChain → Tool Agent*
   - Paste the long “Coaching Session to Video Prompt” system message.
   - Connect:
     - Video Generator GPT Model → Video Generator (AI languageModel)
     - Video Generator → Content Post Generator (AI tool)

10. **Create “Text to Video” (Tool Workflow)**
   - Node: *LangChain → Tool Workflow*
   - Link to your “Text to Video” subworkflow via **Workflow ID**
   - Connect: Text to Video → Video Generator (AI tool)

11. **Create “Image Generator” (Tool Agent)**
   - Node: *LangChain → Tool Agent*
   - Paste the image specialist system message.
   - Connect:
     - Video Generator GPT Model → Image Generator (AI languageModel)
     - Image Generator → Content Post Generator (AI tool)

12. **Create “Text to Image” (Tool Workflow)**
   - Node: *LangChain → Tool Workflow*
   - Link to your “Text to Image” subworkflow via **Workflow ID**
   - Connect: Text to Image → Image Generator (AI tool)

> Recommended fix: Either (a) ensure the Orchestrator passes the Fathom session payload into Content Post Generator and down into Video/Image agents, **or** (b) also connect “Get Fathom Transcript” as a tool to Video Generator and Image Generator so they can fetch it themselves.

---

### B) Required subworkflows (must be created separately)

You must create **three separate workflows** and select them inside the three Tool Workflow nodes:

1. **Subworkflow: Transcript to Content**
   - **Inputs expected:** transcript text + summary + optional metadata (title/date/client).
   - **Outputs expected:** `post_title`, `post_content`, and `googleDocUrl` (or doc ID).
   - Likely nodes: LLM → compose post → Google Docs create/update.

2. **Subworkflow: Text to Video**
   - **Inputs expected:** `prompt` (the video prompt text).
   - **Outputs expected:** `videoUrl` or equivalent.
   - Likely nodes: HTTP request to Luma/Runway/etc + polling + return URL.

3. **Subworkflow: Text to Image**
   - **Inputs expected:** `prompt` (image prompt).
   - **Outputs expected:** `imageUrl` or file reference.
   - Likely nodes: OpenAI Images (or Stability) → store → return.

---

### C) Optional/embedded “executeWorkflowTrigger” pipeline (as shown in JSON)

If you want to keep the embedded entry workflow logic:

1. Create **Subworkflow Entry Point** (Execute Workflow Trigger) and define inputs:
   - `post_title`, `post_content`, `original_meeting_title`, `original_meeting_date`, `prompt`, `input.prompt`

2. Add Google Docs nodes:
   - **Create Google Doc**: title = `{{$json.post_title}}`, choose folderId
   - **Insert Content into Doc**: fix expression to reference the trigger node correctly, e.g.:
     - `{{$('Subworkflow Entry Point').item.json.post_content}}`

3. Add OpenAI image generation via HTTP:
   - **Call DALL-E API**: add header `Authorization: Bearer <key>` and `Content-Type: application/json`
   - Ensure you’re using the response format that includes `b64_json` if you keep Convert-to-Binary as-is.

4. Add Drive upload:
   - Fix file name (replace `"="` with a real name expression like `{{$json.post_title}}.png`).

5. Add kie.ai video creation + polling:
   - Add required auth headers for kie.ai.
   - Add a **Wait/Loop** node pattern to poll until `resultJson` exists.

6. Connect **Format Video Link → Video Ready** (currently missing).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Join the Discord | https://discord.com/invite/XPKeKXeB7d |
| Ask in the Forum | https://community.n8n.io/ |
| Chat Trigger node documentation | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chattrigger/ |
| AI Agent node documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/ |
| Tool Agent node documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.agenttool/ |
| Tool Workflow node documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolworkflow/ |

**Disclaimer (as provided):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.