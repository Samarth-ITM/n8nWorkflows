Generate AI videos and carousels with Blotato for Instagram and TikTok

https://n8nworkflows.xyz/workflows/generate-ai-videos-and-carousels-with-blotato-for-instagram-and-tiktok-13526


# Generate AI videos and carousels with Blotato for Instagram and TikTok

## 1. Workflow Overview

**Purpose:**  
This workflow turns a Telegram message (containing a public URL + a target platform request) into a fully automated publishing pipeline using **Blotato**: it extracts content from the URL, generates either an **Instagram carousel** or a **TikTok video**, waits until generation is complete, posts to the selected platform, then sends a Telegram confirmation.

**Primary use cases:**
- Auto-generate short-form content from YouTube/articles/social links and publish it as:
  - **Instagram**: tweet-card style quote carousel
  - **TikTok**: AI-image-based narrated video with captions
- Operate via a single Telegram message trigger, with an AI agent orchestrating the steps.

### 1.1 Input Reception (Telegram)
Receives the user message that includes a URL and “instagram”/“tiktok”.

### 1.2 AI Orchestration (Agent + LLM + Memory)
A LangChain agent (powered by OpenAI) interprets the message, decides the branch (Instagram vs TikTok), and calls Blotato/Telegram “tools” in the correct order, persisting conversational context.

### 1.3 Content Extraction (Blotato Source)
Creates a Blotato “source” from the URL and polls until the extracted content is ready.

### 1.4 Visual Generation (Blotato Video/Carousel Templates)
Generates either a tweet-card carousel (Instagram) or an AI story video (TikTok) using Blotato templates.

### 1.5 Status Polling & Retrieval (Blotato Visual Get)
Polls the visual until `status = done`, then retrieves media URLs.

### 1.6 Publishing + Confirmation
Posts to Instagram/TikTok via Blotato and finally notifies the user in Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Intake
**Overview:** Receives Telegram messages and forwards them to the AI agent as the single entry point.  
**Nodes involved:** `Telegram Trigger`

#### Node: Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — webhook-based trigger for Telegram bot updates.
- **Configuration (interpreted):**
  - Listens to **updates: `message`**.
  - Uses a configured Telegram bot credential.
- **Key variables:**
  - Incoming text: `$json.message.text`
  - Chat id: `$json.message.chat.id`
- **Connections:**
  - **Output (main)** → `Social Media Autopilot`
- **Potential failures / edge cases:**
  - Telegram credential invalid/revoked.
  - Bot not added to the chat (or user blocked the bot).
  - Message missing a URL or platform keyword (handled by agent logic: asks for clarification and stops).

---

### Block 2 — Agent Orchestration (LLM + Memory + Tools)
**Overview:** Central control plane. The agent reads the Telegram message, extracts URL/platform, runs Blotato steps, polls until completion, posts, then notifies Telegram.  
**Nodes involved:** `Social Media Autopilot`, `OpenAI ChatGPT`, `Simple Memory`

#### Node: Social Media Autopilot
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain agent that can call connected “AI tools” (other nodes) and use memory + a chat model.
- **Configuration (interpreted):**
  - **Input text:** `input user from telegram : {{ $json.message.text }}`
  - **Prompt type:** `define` (custom system message)
  - **Max iterations:** `100` (allows repeated polling calls)
  - **System message:** Enforces strict sequence:
    1) Extract URL + detect platform  
    2) Create source  
    3) Poll source until `status = completed`  
    4) Create visual (Instagram carousel OR TikTok video)  
    5) Poll visual until `status = done`  
    6) Extract media URL(s)  
    7) Post to platform (mandatory)  
    8) Send Telegram confirmation (must be last)
- **Tools available to the agent (by connections):**
  - `Create source`, `Get source`, `Create visual - tweet card carousel`, `Create visual - AI image video`, `Get visual`, `Post to Instagram`, `Post to TikTok`, `Send notification`
- **Connections:**
  - **Input:** from `Telegram Trigger`
  - **ai_languageModel:** from `OpenAI ChatGPT`
  - **ai_memory:** from `Simple Memory`
  - **ai_tool:** to all Blotato tool nodes + Telegram send node
- **Potential failures / edge cases:**
  - Ambiguous platform: system message instructs agent to ask user and stop.
  - Infinite/long polling: capped by `maxIterations=100`; if Blotato stays pending too long, agent may fail before completion.
  - Tool input mismatches: the tool nodes use `$fromAI(...)` fields; if the agent doesn’t supply required parameters (e.g., Source_ID / Video_ID), calls fail.
  - The system message example includes a “✅” character; the actual send node text is AI-provided—publication confirmation content depends on agent compliance.

#### Node: OpenAI ChatGPT
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model used by the agent.
- **Configuration (interpreted):**
  - Model: **`gpt-4o-mini`**
  - Default options (no special temperature/tools config shown).
- **Credentials:** OpenAI API credential configured in n8n.
- **Connections:**
  - **ai_languageModel** → `Social Media Autopilot`
- **Potential failures / edge cases:**
  - Invalid OpenAI key / quota exceeded / rate limits.
  - Model name unavailable in the account/region.
  - Output variability: agent may need strong guardrails (provided via the system message).

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversation memory for the agent.
- **Configuration (interpreted):**
  - Session key: `{{ $json.message.chat.id }}` (per Telegram chat)
  - Context window length: `35` turns/items
- **Connections:**
  - **ai_memory** → `Social Media Autopilot`
- **Potential failures / edge cases:**
  - If chat.id is missing (non-message update), sessionKey expression fails (but trigger is limited to `message` updates).
  - Memory can accumulate irrelevant context across many runs in the same chat; may affect agent decisions.

---

### Block 3 — Source Creation & Polling (Blotato)
**Overview:** Creates a Blotato “source” from the provided URL (article/video/social link), then repeatedly retrieves it until extraction completes.  
**Nodes involved:** `Create source`, `Get source`

#### Node: Create source
- **Type / role:** `@blotato/n8n-nodes-blotato.blotatoTool` — Blotato API tool node, “source” resource creation.
- **Configuration (interpreted):**
  - Resource: **`source`**
  - `sourceUrl`: provided by the agent via `$fromAI('URL')`
  - `customInstructions`: optional, via `$fromAI('Optional_Instructions')`
- **Credentials:** Blotato API credential.
- **Connections:**
  - Exposed as **ai_tool** to `Social Media Autopilot`
- **Key expectations:**
  - Output should include a `sourceId` and initial `status` (typically pending/processing).
- **Potential failures / edge cases:**
  - URL not publicly accessible / blocked / requires auth.
  - Unsupported source type on Blotato side.
  - API auth errors, timeouts, or rate limits.

#### Node: Get source
- **Type / role:** Blotato tool node, “source” resource retrieval.
- **Configuration (interpreted):**
  - Resource: **`source`**
  - Operation: **`get`**
  - `sourceId`: supplied by the agent via `$fromAI('Source_ID')`
- **Connections:**
  - Exposed as **ai_tool** to `Social Media Autopilot`
- **Key expectations:**
  - Returns `status`; agent waits until `status = completed`.
  - Returns extracted `content` when completed.
- **Potential failures / edge cases:**
  - Wrong/empty sourceId → 404/not found.
  - Source stuck in `pending/processing` beyond iteration budget.

---

### Block 4 — Visual Generation (Blotato Templates)
**Overview:** Generates the final media asset using Blotato templates. Agent chooses which node to call based on platform.  
**Nodes involved:** `Create visual - tweet card carousel`, `Create visual - AI image video`

#### Node: Create visual - tweet card carousel
- **Type / role:** Blotato tool node — creates a **carousel-like quote card** (produced as a “video” resource in this workflow).
- **Configuration (interpreted):**
  - Resource: **`video`** (even though output is carousel-like; template likely yields multiple images/frames)
  - Template: **“Twitter/X style quote cards with minimal style”**
  - Prompt: `$fromAI('Prompt')` (agent-generated, derived from extracted content)
  - Template inputs (pre-filled defaults):
    - handle: `doc.firass`
    - authorName: `Dr. FIRAS`
    - profileImage: `https://www.dr-firas.com/logo.jpg` (must be publicly accessible)
    - verified: `false`
    - (optional) quotes, theme, aspectRatio
- **Connections:** Exposed as **ai_tool** to `Social Media Autopilot`
- **Key expectations:**
  - Returns a `videoId` (visual ID) and a generation `status`.
- **Potential failures / edge cases:**
  - profile image URL not reachable (template may fail).
  - Quotes formatting: template schema expects JSON-like string (e.g., `["item 1","item 2"]`); malformed strings can break generation.
  - Template ID/version mismatch if Blotato changes template endpoints.

#### Node: Create visual - AI image video
- **Type / role:** Blotato tool node — generates a TikTok-style narrated video from scenes (AI images/videos) with captions.
- **Configuration (interpreted):**
  - Resource: **`video`**
  - Template: “Create scenes with images, videos, or AI-generated images…”
  - Prompt: `$fromAI('Prompt')`
  - Template inputs defaults:
    - voiceName: `Brian (American, deep)`
    - aiImageModel: `replicate/recraft-ai/recraft-v3`
    - animateAiImages: `true`
    - trimToVoiceover: `true`
    - (optional) scenes (JSON string), captionPosition, transition, aspectRatio, highlightColor
- **Connections:** Exposed as **ai_tool** to `Social Media Autopilot`
- **Potential failures / edge cases:**
  - Scene JSON malformed (if agent supplies `scenes`).
  - Underlying AI image model availability/quotas (Replicate/OpenAI/fal.ai as applicable via Blotato).
  - Longer generation times → increased risk of hitting agent iteration limit.

---

### Block 5 — Visual Polling & Media Retrieval
**Overview:** Polls the generated visual until it is fully ready (`status = done`), then the agent extracts `mediaUrl` (video) or `imageUrls` (carousel).  
**Nodes involved:** `Get visual`

#### Node: Get visual
- **Type / role:** Blotato tool node — retrieves a video/visual generation status and outputs media URLs when ready.
- **Configuration (interpreted):**
  - Resource: **`video`**
  - Operation: **`get`**
  - `videoId`: `$fromAI('Video_ID')`
- **Connections:** Exposed as **ai_tool** to `Social Media Autopilot`
- **Key expectations:**
  - Returns `status` including states like: `generating-script`, `processing`, `queued`, `pending`, and final `done`.
  - Returns media fields when done:
    - For video: `mediaUrl`
    - For carousel: `imageUrls`
- **Potential failures / edge cases:**
  - Agent must keep polling; premature posting is explicitly forbidden by system message, but still possible if agent misbehaves.
  - `mediaUrl/imageUrls` may be null briefly even if status changes—agent system message instructs validation.

---

### Block 6 — Publishing (Instagram/TikTok)
**Overview:** Posts the media to the chosen platform using Blotato-connected accounts.  
**Nodes involved:** `Post to Instagram`, `Post to TikTok`

#### Node: Post to Instagram
- **Type / role:** Blotato tool node — publishes content to Instagram via a Blotato-connected account.
- **Configuration (interpreted):**
  - Account: `doc.firass` (accountId `11892`)
  - `postContentText`: `$fromAI('Text')`
  - `postContentMediaUrls`: `$fromAI('Media_URLs')` (string; agent should format as required by Blotato—often comma-separated or JSON depending on node expectations)
- **Connections:** Exposed as **ai_tool** to `Social Media Autopilot`
- **Potential failures / edge cases:**
  - Instagram account not connected/expired inside Blotato.
  - Wrong media URL format (single vs multiple URLs for carousel).
  - Instagram constraints: aspect ratio, file type, size, caption length, carousel limits.

#### Node: Post to TikTok
- **Type / role:** Blotato tool node — publishes content to TikTok via a Blotato-connected account.
- **Configuration (interpreted):**
  - Platform fixed: `tiktok`
  - Account: `eliteshicos` (accountId `30526`)
  - `postContentText`: `$fromAI('Text')`
  - `postContentMediaUrls`: `$fromAI('Media_URLs')` (typically a single video URL)
- **Connections:** Exposed as **ai_tool** to `Social Media Autopilot`
- **Potential failures / edge cases:**
  - TikTok auth/session expired in Blotato.
  - Upload/publish restrictions, region limitations, or media encoding constraints.

---

### Block 7 — Telegram Confirmation
**Overview:** Sends a Telegram message back to the user only after successful posting.  
**Nodes involved:** `Send notification`

#### Node: Send notification
- **Type / role:** `n8n-nodes-base.telegramTool` — sends a message via Telegram bot.
- **Configuration (interpreted):**
  - chatId: `{{ $json.message.chat.id }}`
  - text: `$fromAI('Text')` (agent should write “Published successfully on Instagram/TikTok” etc.)
- **Connections:** Exposed as **ai_tool** to `Social Media Autopilot`
- **Potential failures / edge cases:**
  - Bot blocked by user / cannot message user (Telegram error).
  - Agent sends confirmation before posting (system message forbids it, but this is policy-based, not technically enforced).

---

### Block 8 — Documentation / Notes (Sticky Note)
**Overview:** A workspace note containing branding and setup resources.  
**Nodes involved:** `Sticky Note6`

#### Node: Sticky Note6
- **Type / role:** `n8n-nodes-base.stickyNote` — documentation overlay in the canvas (non-executable).
- **Content includes:** branding image + Notion link + setup checklist + description of workflow behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Entry point: receive Telegram message | — | Social Media Autopilot | # 🚀  Generate AI videos and carousels with Blotato and publish to Instagram & TikTok  \n## (By Dr. Firas)\n\n![SORA2 logo](https://www.dr-firas.com/blotato-miniature.png)\n\n# 📘 Documentation  \nAccess detailed setup instructions, API config, platform connection guides, and workflow customization tips: 📎 [Open the full documentation on Notion](https://automatisation.notion.site/Turn-AI-Videos-Carousels-Into-Income-with-n8n-Fully-Automated-x-Blotato-30c3d6550fd9804b999ede955fdf409d?source=copy_link)\n\n## 🔐 Setup\n\nTo use this workflow, you will need:\n\n- An active **n8n** instance\n- A **[Blotato](https://blotato.com/?ref=firas)** account with API access\n- Instagram and/or TikTok accounts connected in **[Blotato](https://blotato.com/?ref=firas)**\n- A **Telegram Bot** for triggering the workflow and receiving notifications\n\nSetup steps:\n1. Import the workflow JSON into n8n.\n2. Add your **[Blotato](https://blotato.com/?ref=firas)** API credentials.\n3. Configure the Telegram Trigger with your bot token.\n4. Select your Instagram and TikTok accounts in the **[Blotato](https://blotato.com/?ref=firas)** post nodes.\n5. Activate the workflow.\n\n---\n## What this workflow does\n\nThis workflow provides a complete **end-to-end automation pipeline**:\n\n1. Receives a message from **Telegram** containing a public URL and a publishing instruction.\n2. Creates a content source from the URL using **Blotato**.\n3. Retrieves and validates the extracted text content.\n4. Generates either:\n   - An **AI tweet-card carousel** for Instagram, or\n   - An **AI-generated video** for TikTok.\n5. Continuously checks the visual generation status until it is fully completed.\n6. Publishes the final media automatically to **Instagram or TikTok**.\n7. Sends a confirmation message back to Telegram once the post is successfully published. |
| Social Media Autopilot | @n8n/n8n-nodes-langchain.agent | Orchestrates all steps; chooses branch; calls tools | Telegram Trigger; OpenAI ChatGPT; Simple Memory | (tool calls) Create/Get source, Create/Get visual, Post nodes, Send notification | (same note as above) |
| OpenAI ChatGPT | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM powering the agent | — | Social Media Autopilot | (same note as above) |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Per-chat short-term memory | — | Social Media Autopilot | (same note as above) |
| Create source | @blotato/n8n-nodes-blotato.blotatoTool | Create Blotato source from URL | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Get source | @blotato/n8n-nodes-blotato.blotatoTool | Poll/retrieve source until completed | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Create visual - tweet card carousel | @blotato/n8n-nodes-blotato.blotatoTool | Generate Instagram quote-card carousel | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Create visual - AI image video | @blotato/n8n-nodes-blotato.blotatoTool | Generate TikTok AI narrated video | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Get visual | @blotato/n8n-nodes-blotato.blotatoTool | Poll visual until status done; retrieve URLs | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Post to Instagram | @blotato/n8n-nodes-blotato.blotatoTool | Publish media + caption to Instagram | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Post to TikTok | @blotato/n8n-nodes-blotato.blotatoTool | Publish media + caption to TikTok | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Send notification | n8n-nodes-base.telegramTool | Send final confirmation to Telegram | Social Media Autopilot (as ai_tool) | Social Media Autopilot (tool result) | (same note as above) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation note (non-executable) | — | — | # 🚀  Generate AI videos and carousels with Blotato and publish to Instagram & TikTok (content as shown) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: **Message**
   - Credentials: create/select **Telegram Bot API** credential (token from BotFather).
   - This node is the workflow entry point.

2) **Create the OpenAI chat model node**
   - Add node: **OpenAI Chat Model** (LangChain) / `lmChatOpenAi`
   - Model: **gpt-4o-mini**
   - Credentials: create/select **OpenAI API** credential.

3) **Create Memory node**
   - Add node: **Simple Memory** (`memoryBufferWindow`)
   - Session id type: **Custom key**
   - Session key expression: `{{ $json.message.chat.id }}`
   - Context window length: **35**

4) **Create the Agent node**
   - Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Text field: `input user from telegram : {{ $json.message.text }}`
   - Set **Prompt Type** to “Define”
   - Paste the provided system message (the “Social Media Autopilot (End-to-End Execution)” instructions).
   - Options: set **Max iterations = 100**

5) **Connect core nodes**
   - Connect **Telegram Trigger (main)** → **AI Agent (main)**
   - Connect **OpenAI Chat Model (ai_languageModel)** → **AI Agent (ai_languageModel)**
   - Connect **Simple Memory (ai_memory)** → **AI Agent (ai_memory)**

6) **Add Blotato tool nodes (credentials required)**
   - Create a credential: **Blotato API** (API key from Blotato).
   - Add node: **Blotato Tool** named **Create source**
     - Resource: **source**
     - sourceUrl: set to AI-filled input using `$fromAI('URL')`
     - customInstructions: `$fromAI('Optional_Instructions')`
   - Add node: **Blotato Tool** named **Get source**
     - Resource: **source**
     - Operation: **get**
     - sourceId: `$fromAI('Source_ID')`

7) **Add visual generation nodes**
   - Add node: **Create visual - tweet card carousel**
     - Resource: **video**
     - Template: select **Twitter/X style quote cards with minimal style**
     - Prompt: `$fromAI('Prompt')`
     - Template inputs: set defaults:
       - handle `doc.firass`
       - authorName `Dr. FIRAS`
       - profileImage `https://www.dr-firas.com/logo.jpg`
       - verified `false`
   - Add node: **Create visual - AI image video**
     - Resource: **video**
     - Template: select the “AI story video” template (scenes + voiceover + captions)
     - Prompt: `$fromAI('Prompt')`
     - Template inputs defaults:
       - voiceName `Brian (American, deep)`
       - aiImageModel `replicate/recraft-ai/recraft-v3`
       - animateAiImages `true`
       - trimToVoiceover `true`

8) **Add visual polling node**
   - Add node: **Get visual**
     - Resource: **video**
     - Operation: **get**
     - videoId: `$fromAI('Video_ID')`

9) **Add publishing nodes**
   - Add node: **Post to Instagram**
     - Select Blotato-connected Instagram account (via accountId list)
     - postContentText: `$fromAI('Text')`
     - postContentMediaUrls: `$fromAI('Media_URLs')`
   - Add node: **Post to TikTok**
     - Platform: `tiktok`
     - Select Blotato-connected TikTok account
     - postContentText: `$fromAI('Text')`
     - postContentMediaUrls: `$fromAI('Media_URLs')`

10) **Add Telegram send node (confirmation)**
   - Add node: **Telegram** (Send Message / telegramTool) named **Send notification**
   - chatId: `{{ $json.message.chat.id }}`
   - text: `$fromAI('Text')`
   - Credentials: same Telegram bot credential as the trigger.

11) **Expose all tools to the Agent**
   - Connect each of these nodes to the agent using **AI Tool** connections:
     - `Create source` → Agent (ai_tool)
     - `Get source` → Agent (ai_tool)
     - `Create visual - tweet card carousel` → Agent (ai_tool)
     - `Create visual - AI image video` → Agent (ai_tool)
     - `Get visual` → Agent (ai_tool)
     - `Post to Instagram` → Agent (ai_tool)
     - `Post to TikTok` → Agent (ai_tool)
     - `Send notification` → Agent (ai_tool)

12) **(Optional) Add the sticky note**
   - Add a Sticky Note with the provided content and links for operational context.

13) **Activate and test**
   - Activate workflow.
   - Send a Telegram message like:  
     - “Post this on instagram: https://example.com/article …”  
     - or “Make a tiktok from https://youtube.com/...”

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Branding image used in the canvas note | https://www.dr-firas.com/blotato-miniature.png |
| Full documentation link (Notion) | https://automatisation.notion.site/Turn-AI-Videos-Carousels-Into-Income-with-n8n-Fully-Automated-x-Blotato-30c3d6550fd9804b999ede955fdf409d?source=copy_link |
| Blotato signup/reference link | https://blotato.com/?ref=firas |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.