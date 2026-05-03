OpenClaw Clone 🦞: Expandable Personal Telegram AI Agent Template

https://n8nworkflows.xyz/workflows/openclaw-clone-----expandable-personal-telegram-ai-agent-template-14008


# OpenClaw Clone 🦞: Expandable Personal Telegram AI Agent Template

# OpenClaw Clone 🦞: Expandable Personal Telegram AI Agent Template

## 1. Workflow Overview

This workflow is a Telegram-centric AI orchestration system built in n8n. It receives user inputs from Telegram, optional scheduled/internal/webhook inputs, normalizes them into a common format, sends them to a central AI agent, lets that agent autonomously choose among many connected tools, and returns the result either as text or audio.

Its main purpose is to act as a modular personal AI assistant that can:
- converse over Telegram,
- remember prior interactions via PostgreSQL chat memory,
- use web, RAG, Google Workspace, calculator, media, and MCP/sub-workflow tools,
- escalate to a human when needed.

### 1.1 Input Reception and Authorization
The workflow has three practical entry patterns:
- Telegram messages via a Telegram Trigger
- Scheduled prompts via a Schedule Trigger
- External/internal webhook-based prompts via a Webhook node

Telegram traffic is access-controlled through a Code node that checks the sender ID.

### 1.2 Telegram Content Routing and Preprocessing
Authorized Telegram messages are classified by content type:
- text,
- voice,
- image.

Each type is transformed into the normalized fields the agent expects:
- `chatInput`
- `sessionId`
- optional `image_url`

### 1.3 Central AI Orchestration
The `OpenClaw Agents` node is the core orchestrator. It uses:
- Google Gemini as the chat model,
- PostgreSQL as conversation memory,
- a large toolset attached as AI tools.

This block decides what tools to call and synthesizes the final answer.

### 1.4 Specialized Tooling and Extensions
The agent can access:
- Perplexity for web search,
- ScrapeGraphAI for scraping,
- Qdrant RAG with OpenAI embeddings and Cohere reranking,
- Calculator,
- Google Workspace via MCP-based tool clusters,
- sub-workflows,
- generic MCP clients,
- Telegram-based human escalation.

### 1.5 Output Formatting and Delivery
The workflow checks whether the original Telegram message was voice. If yes, it converts the AI output to speech and sends audio back. Otherwise, it sends a text reply.

### 1.6 MCP Server Blocks for Google Services
The workflow also embeds several MCP trigger/tool server sections for Gmail, Calendar, Drive, Docs, Sheets, and Slides. These are not part of the main execution path for Telegram input, but they expose grouped tools to MCP clients, including the main orchestrator’s MCP client tools.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Access Control

### Overview
This block receives inbound Telegram messages and filters them by authorized Telegram user ID. Unauthorized users are effectively blocked because only authorized payloads are forwarded.

### Nodes Involved
- `Get Message`
- `Code`

### Node Details

#### Get Message
- **Type and role:** `n8n-nodes-base.telegramTrigger` — entry node for Telegram messages.
- **Configuration choices:** Listens to `message` updates only.
- **Key expressions/variables used:** None internally; downstream nodes use `message.from.id`, `message.text`, `message.voice.file_id`, and `message.photo`.
- **Input/Output connections:** Entry node → outputs to `Code`.
- **Version-specific requirements:** Type version `1.1`; requires Telegram credentials and bot webhook registration handled by n8n.
- **Edge cases/failures:**
  - Telegram credential or webhook issues
  - Bot not started by the user
  - Unsupported message structure
  - Rate limits from Telegram
- **Sub-workflow reference:** None.

#### Code
- **Type and role:** `n8n-nodes-base.code` — authorization gate.
- **Configuration choices:** Checks whether `message.from.id !== XXX`; if so, returns `{ unauthorized: true }`, else returns original items.
- **Key expressions/variables used:**  
  - `$input.first().json.message.from.id`
  - hardcoded placeholder `XXX`
- **Input/Output connections:** Input from `Get Message` → output to `Switch2`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases/failures:**
  - If `XXX` is not replaced, the workflow will fail or reject all users depending on syntax
  - If a Telegram update lacks `message.from.id`, expression access may break
  - Unauthorized output still flows forward, but without expected message fields; downstream routing will likely emit no matching branch
- **Sub-workflow reference:** None.

---

## 2.2 Alternate Entry Points: Scheduled and Webhook Inputs

### Overview
This block allows the same AI orchestrator to be invoked without Telegram. It supports periodic prompts and custom webhook-fed prompts, both normalized into the same structure as Telegram text.

### Nodes Involved
- `Schedule Trigger`
- `Set up istruction`
- `Webhook`
- `Normalization`

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger` — scheduled workflow entry.
- **Configuration choices:** Basic interval trigger; currently generic/default and intended for user customization.
- **Key expressions/variables used:** None.
- **Input/Output connections:** Entry node → `Set up istruction`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failures:**
  - Misconfigured schedule frequency
  - Activated workflow required for production scheduling
- **Sub-workflow reference:** None.

#### Set up istruction
- **Type and role:** `n8n-nodes-base.set` — builds normalized AI input for scheduled prompts.
- **Configuration choices:** Sets:
  - `chatInput = xxxx`
  - `sessionId = xxxx`
  Both are placeholders.
- **Key expressions/variables used:** Static placeholder values.
- **Input/Output connections:** Input from `Schedule Trigger` → output to `OpenClaw Agents`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - If placeholders are not replaced, scheduled prompts are useless
  - `sessionId` should be stable per user/conversation to preserve memory continuity
- **Sub-workflow reference:** None.

#### Webhook
- **Type and role:** `n8n-nodes-base.webhook` — HTTP entry point.
- **Configuration choices:** Exposes path `56c7aeae-eb8a-4d2e-a149-dfe02b586d18`.
- **Key expressions/variables used:** None in node config.
- **Input/Output connections:** Entry node → `Normalization`.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases/failures:**
  - Public endpoint exposure/security concerns
  - Caller payload must include fields compatible with the normalization logic
- **Sub-workflow reference:** None.

#### Normalization
- **Type and role:** `n8n-nodes-base.set` — maps webhook payload into AI input fields.
- **Configuration choices:** Sets:
  - `chatInput = {{$json.message.text}}`
  - `sessionId = {{$json.message.from.id}}`
- **Key expressions/variables used:** `message.text`, `message.from.id`
- **Input/Output connections:** Input from `Webhook` → output to `OpenClaw Agents`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - Webhook callers must provide Telegram-like JSON shape, or expressions will resolve empty/fail
  - Better redesigned if using non-Telegram webhook sources
- **Sub-workflow reference:** None.

---

## 2.3 Telegram Content Routing and Text Handling

### Overview
This block detects the incoming Telegram content type and routes text directly into the orchestrator with normalized memory/session fields.

### Nodes Involved
- `Switch2`
- `Get Text`

### Node Details

#### Switch2
- **Type and role:** `n8n-nodes-base.switch` — routes Telegram messages into text, audio, or image branches.
- **Configuration choices:** Three named outputs:
  - `Text` if `message.text` exists
  - `Audio` if `message.voice.file_id` exists
  - `Immagine` if `message.photo[0]` exists
- **Key expressions/variables used:**
  - `{{$json.message.text}}`
  - `{{$json.message.voice.file_id}}`
  - `{{$json.message.photo[0]}}`
- **Input/Output connections:** Input from `Code` → outputs to:
  - `Get Text`
  - `Get voice message`
  - `Get image file`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases/failures:**
  - Documents, video, stickers, locations, etc. are not handled
  - Unauthorized payload `{ unauthorized: true }` will not match any branch
  - Mixed message types depend on Telegram payload structure; first matching output routing applies by branch setup
- **Sub-workflow reference:** None.

#### Get Text
- **Type and role:** `n8n-nodes-base.set` — normalizes Telegram text input.
- **Configuration choices:** Sets:
  - `chatInput = {{$json.message.text}}`
  - `sessionId = {{$json.message.from.id}}`
- **Key expressions/variables used:** `message.text`, `message.from.id`
- **Input/Output connections:** Input from `Switch2` text branch → output to `OpenClaw Agents`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - Empty text payloads
  - Session ID typed as number here, but string in image branch; mixed typing can affect memory partitioning if backend is strict
- **Sub-workflow reference:** None.

---

## 2.4 Voice Handling and Transcription

### Overview
This block downloads Telegram voice audio, transcribes it with OpenAI, and converts the transcript into normalized text for the AI agent.

### Nodes Involved
- `Get voice message`
- `Transcribe recording`
- `Get input text from voice`

### Node Details

#### Get voice message
- **Type and role:** `n8n-nodes-base.telegram` — downloads the Telegram voice file.
- **Configuration choices:** Resource `file`; `fileId` from original Telegram message.
- **Key expressions/variables used:** `{{ $('Get Message').item.json.message.voice.file_id }}`
- **Input/Output connections:** Input from `Switch2` audio branch → output to `Transcribe recording`.
- **Version-specific requirements:** Type version `1.2`; requires Telegram credentials with file access.
- **Edge cases/failures:**
  - File expired/unavailable
  - Telegram API or auth problems
  - Large files increasing latency
- **Sub-workflow reference:** None.

#### Transcribe recording
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi` — audio transcription.
- **Configuration choices:** Resource `audio`, operation `transcribe`, language set to `it`.
- **Key expressions/variables used:** Binary input from previous node; output expected in `text`.
- **Input/Output connections:** Input from `Get voice message` → output to `Get input text from voice`.
- **Version-specific requirements:** Type version `1.7`; requires OpenAI credentials and model availability for transcription.
- **Edge cases/failures:**
  - Wrong transcription language for non-Italian speech
  - Unsupported or corrupt audio format
  - API limits or cost issues
  - Slow transcription on long audio
- **Sub-workflow reference:** None.

#### Get input text from voice
- **Type and role:** `n8n-nodes-base.set` — normalizes transcript for the main agent.
- **Configuration choices:** Sets:
  - `chatInput = {{$json.text}}`
  - `sessionId = {{ $('Get Message').item.json.message.from.id }}`
- **Key expressions/variables used:** `text`, original sender ID.
- **Input/Output connections:** Input from `Transcribe recording` → output to `OpenClaw Agents`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - Empty transcript
  - Transcription failure producing missing `text`
- **Sub-workflow reference:** None.

---

## 2.5 Image Handling and URL Preparation

### Overview
This block downloads an image from Telegram, uploads it to FTP, constructs a public image URL, and forwards both the URL and optional caption to the AI agent.

### Nodes Involved
- `Get image file`
- `Upload image`
- `Set Image Url`

### Node Details

#### Get image file
- **Type and role:** `n8n-nodes-base.telegram` — downloads one of the photo sizes from Telegram.
- **Configuration choices:** Uses `message.photo[2].file_id`, implying a preference for a larger photo variant.
- **Key expressions/variables used:** `{{$json.message.photo[2].file_id}}`
- **Input/Output connections:** Input from `Switch2` image branch → output to `Upload image`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:**
  - `photo[2]` may not exist if Telegram returns fewer sizes
  - File access/auth issues
  - Caption may be absent
- **Sub-workflow reference:** None.

#### Upload image
- **Type and role:** `n8n-nodes-base.ftp` — uploads image binary to an FTP server.
- **Configuration choices:** Upload operation with path `/XXX/{{ $binary.data.fileName }}`.
- **Key expressions/variables used:** `$binary.data.fileName`
- **Input/Output connections:** Input from `Get image file` → output to `Set Image Url`.
- **Version-specific requirements:** Type version `1`; requires FTP credentials.
- **Edge cases/failures:**
  - Placeholder path `XXX` not replaced
  - Missing binary property name or file name
  - FTP auth, permission, connectivity, and path issues
- **Sub-workflow reference:** None.

#### Set Image Url
- **Type and role:** `n8n-nodes-base.set` — converts uploaded file info into agent-ready fields.
- **Configuration choices:** Sets:
  - `image_url = https://XXX/{{ $binary.data.fileName }}`
  - `chatInput = {{ $('Get Message').item.json.message.caption || "" }}`
  - `sessionId = {{ $('Get Message').item.json.message.from.id }}`
- **Key expressions/variables used:** public URL placeholder, caption fallback, sender ID.
- **Input/Output connections:** Input from `Upload image` → output to `OpenClaw Agents`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - Public domain placeholder `XXX` not replaced
  - FTP upload succeeds but URL is not actually public
  - Empty caption means the AI receives only image URL context
- **Sub-workflow reference:** None.

---

## 2.6 Central AI Orchestrator

### Overview
This is the core intelligence layer. It receives normalized user input, optional image URLs, chat memory, a Gemini language model, and a set of tools. It plans and executes actions and returns a synthesized final output.

### Nodes Involved
- `OpenClaw Agents`
- `Google Gemini Chat Model`
- `Postgres Chat Memory`

### Node Details

#### OpenClaw Agents
- **Type and role:** `@n8n/n8n-nodes-langchain.agent` — main AI agent/orchestrator.
- **Configuration choices:**
  - Prompt type: `define`
  - Text input combines:
    - `{{$json.chatInput ?? ''}}`
    - `Image Url (if exist): {{$json.image_url ?? ''}}`
  - Uses a long custom system message describing tool-selection logic, constraints, and orchestration policy.
- **Key expressions/variables used:**
  - `chatInput`
  - `image_url`
  - AI-linked tool inputs via `$fromAI(...)`
- **Input/Output connections:**
  - Main inputs from `Get Text`, `Get input text from voice`, `Set Image Url`, `Normalization`, `Set up istruction`
  - AI language model input from `Google Gemini Chat Model`
  - AI memory input from `Postgres Chat Memory`
  - AI tool inputs from all connected tools
  - Main output to `From audio to audio?`
- **Version-specific requirements:** Type version `3.1`; depends on LangChain-capable n8n version and compatible AI tool nodes.
- **Edge cases/failures:**
  - Prompt/tool mismatch can cause poor tool use
  - If MCP clients are not configured, agent may try unavailable tools
  - Long chains may hit token/runtime limits
  - Inconsistent tool descriptions can mislead the agent
- **Sub-workflow reference:** Indirectly invokes tool workflows through connected `toolWorkflow` nodes.

#### Google Gemini Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — language model used by the main agent.
- **Configuration choices:** Default options, no explicit model shown in JSON.
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI language model connection → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1`; requires Google Gemini credentials/API access.
- **Edge cases/failures:**
  - Missing/invalid API key
  - Regional/model availability issues
  - Safety or quota limits
- **Sub-workflow reference:** None.

#### Postgres Chat Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryPostgresChat` — persistent conversation memory store.
- **Configuration choices:** Default config in JSON; actual DB/credential setup is external.
- **Key expressions/variables used:** Relies on upstream `sessionId` values to segment conversations.
- **Input/Output connections:** AI memory connection → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1.3`; requires PostgreSQL connectivity and schema support.
- **Edge cases/failures:**
  - Database auth/connectivity failures
  - Session ID type inconsistency across branches
  - Schema initialization issues
- **Sub-workflow reference:** None.

---

## 2.7 Direct Agent Tools: Search, Scraping, RAG, Math, Media, MCP, Escalation

### Overview
These are the non-Google-MCP tool extensions connected directly to the main agent. They enable live web access, structured extraction, retrieval over private knowledge, arithmetic, media generation/editing, generic MCP extensions, Telegram messaging, and human approval/escalation.

### Nodes Involved
- `Websearch Agent`
- `Scraper Agent`
- `RAG Agent`
- `Embeddings OpenAI`
- `Reranker Cohere`
- `Calculator`
- `Image and Video Grok Agent`
- `Others MCP Client`
- `Others SubWF`
- `Escalation`
- `Send a text message in Telegram`

### Node Details

#### Websearch Agent
- **Type and role:** `n8n-nodes-base.perplexityTool` — Perplexity-powered search tool.
- **Configuration choices:** Single AI-defined message content.
- **Key expressions/variables used:** `$fromAI('message0_Text', ..., 'string')`
- **Input/Output connections:** AI tool connection → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1`; requires Perplexity credentials.
- **Edge cases/failures:** API key issues, search quality variability, rate limits.
- **Sub-workflow reference:** None.

#### Scraper Agent
- **Type and role:** `n8n-nodes-scrapegraphai.scrapegraphAiTool` — structured web extraction tool.
- **Configuration choices:** AI supplies URL, extraction prompt, and optional flags for JS rendering, scrolling, output schema, and pagination.
- **Key expressions/variables used:**
  - `$fromAI('User_Prompt', ...)`
  - `$fromAI('Website_URL', ...)`
  - boolean flags from AI
- **Input/Output connections:** AI tool connection → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1`; requires ScrapeGraphAI integration.
- **Edge cases/failures:** blocked sites, dynamic rendering failure, anti-bot protections, malformed extraction prompts.
- **Sub-workflow reference:** None.

#### RAG Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant` — retrieval tool over Qdrant.
- **Configuration choices:**
  - Mode: `retrieve-as-tool`
  - `useReranker = true`
  - tool description: `RAG Agent`
  - collection is currently empty in JSON and must be configured
- **Key expressions/variables used:** Tool invocation via agent; collection selection external.
- **Input/Output connections:**
  - AI tool → `OpenClaw Agents`
  - AI embedding input from `Embeddings OpenAI`
  - AI reranker input from `Reranker Cohere`
- **Version-specific requirements:** Type version `1.3`; requires Qdrant connection.
- **Edge cases/failures:** missing collection, empty index, embedding mismatch, vector DB connectivity issues.
- **Sub-workflow reference:** None.

#### Embeddings OpenAI
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — embedding model for RAG retrieval.
- **Configuration choices:** Default.
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI embedding → `RAG Agent`.
- **Version-specific requirements:** Type version `1.2`; requires OpenAI credentials.
- **Edge cases/failures:** API quota/model issues.
- **Sub-workflow reference:** None.

#### Reranker Cohere
- **Type and role:** `@n8n/n8n-nodes-langchain.rerankerCohere` — reranks retrieved results.
- **Configuration choices:** Default.
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI reranker → `RAG Agent`.
- **Version-specific requirements:** Type version `1`; requires Cohere credentials.
- **Edge cases/failures:** API auth/quota issues.
- **Sub-workflow reference:** None.

#### Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator` — arithmetic/calculation tool.
- **Configuration choices:** Default.
- **Key expressions/variables used:** None; AI sends formula internally.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:** malformed mathematical expressions.
- **Sub-workflow reference:** None.

#### Image and Video Grok Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — sub-workflow tool for image/video generation/editing.
- **Configuration choices:**
  - References workflow `klLje7jNDPINkW4Y`
  - Description: `Create and edit Image and Video`
  - Inputs: `query`, `duration`, `image_url`, `tool_name`, `video_url`
- **Key expressions/variables used:** all via `$fromAI(...)`
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `2.2`; target workflow must exist and expose matching inputs.
- **Edge cases/failures:** missing workflow, mismatched input schema, downstream media API failures.
- **Sub-workflow reference:** `Image Grok Agent with Telegram` (`klLje7jNDPINkW4Y`).

#### Others MCP Client
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — generic placeholder MCP client.
- **Configuration choices:** Default options only; endpoint not configured.
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1.2`; requires configured MCP endpoint to be usable.
- **Edge cases/failures:** node is non-functional until endpoint is provided.
- **Sub-workflow reference:** None.

#### Others SubWF
- **Type and role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — placeholder sub-workflow tool.
- **Configuration choices:**
  - Currently points to same workflow ID as media tool
  - Description in Italian: use only when modifying an existing video, not for creating one
  - Provides `query`, fixed `duration: 0`, and `tool_name: "Run text to image"`
- **Key expressions/variables used:** `$fromAI('query', ...)`
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases/failures:** current configuration appears placeholder/inconsistent with description; likely needs replacement.
- **Sub-workflow reference:** currently also `klLje7jNDPINkW4Y`.

#### Escalation
- **Type and role:** `n8n-nodes-base.telegramHitlTool` — human-in-the-loop approval/escalation tool.
- **Configuration choices:**
  - `chatId` from AI
  - approval type `double`
- **Key expressions/variables used:** `$fromAI('Chat_ID', ..., 'string')`
- **Input/Output connections:** AI tool → `OpenClaw Agents`
- **Version-specific requirements:** Type version `1.2`; requires Telegram setup for HITL interaction.
- **Edge cases/failures:** wrong chat ID, human approval delays, unclear escalation path.
- **Sub-workflow reference:** None.

#### Send a text message in Telegram
- **Type and role:** `n8n-nodes-base.telegramTool` — AI tool for sending Telegram messages.
- **Configuration choices:** AI defines `text` and `chatId`.
- **Key expressions/variables used:**
  - `$fromAI('Text', ..., 'string')`
  - `$fromAI('Chat_ID', ..., 'string')`
- **Input/Output connections:** Connected as AI tool into `Escalation` rather than directly into the main agent, which is unusual.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:** because it is attached to `Escalation` and not `OpenClaw Agents`, it may only be usable through the escalation tool context, not as a normal direct tool.
- **Sub-workflow reference:** None.

---

## 2.8 Google Workspace MCP Client Tools Used by the Main Agent

### Overview
These nodes expose external MCP servers as tools to the main agent. They represent high-level gateways for Gmail, Drive, Calendar, Docs, Sheets, and Slides.

### Nodes Involved
- `Gmail Agent`
- `Google Drive Agent`
- `Google Calendar Agent`
- `Google Docs Agent`
- `Google Sheets Agent`
- `Google Slides Agent`

### Node Details

#### Gmail Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — MCP client for Gmail tools.
- **Configuration choices:** Endpoint URL explicitly set to `https://n8n.skool.n3witalia.com/mcp-test/63975752-0598-4992-9422-165a84c8798c`
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:** endpoint availability, auth, cross-environment mismatch.
- **Sub-workflow reference:** None.

#### Google Drive Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool`
- **Configuration choices:** Currently points to the same endpoint URL as Gmail Agent, which likely needs correction.
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:** likely misconfigured endpoint.
- **Sub-workflow reference:** None.

#### Google Calendar Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool`
- **Configuration choices:** No endpoint URL shown; incomplete configuration.
- **Key expressions/variables used:** None.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:** unusable until configured.
- **Sub-workflow reference:** None.

#### Google Docs Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool`
- **Configuration choices:** Default only; endpoint missing.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Edge cases/failures:** unusable until configured.
- **Sub-workflow reference:** None.

#### Google Sheets Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool`
- **Configuration choices:** Default only; endpoint missing.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Edge cases/failures:** unusable until configured.
- **Sub-workflow reference:** None.

#### Google Slides Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClientTool`
- **Configuration choices:** Default only; endpoint missing.
- **Input/Output connections:** AI tool → `OpenClaw Agents`.
- **Edge cases/failures:** unusable until configured.
- **Sub-workflow reference:** None.

---

## 2.9 Output Routing, TTS, and Telegram Delivery

### Overview
After the agent produces a response, this block decides whether to return text or audio depending on whether the original user input was voice.

### Nodes Involved
- `From audio to audio?`
- `Generate Audio Response`
- `Fix mimeType for Audio`
- `Send an audio file`
- `Send a text message`

### Node Details

#### From audio to audio?
- **Type and role:** `n8n-nodes-base.if` — checks whether original Telegram input was voice.
- **Configuration choices:** Condition tests existence of `$('Get Message').item.json.message.voice.file_id`.
- **Key expressions/variables used:** original Telegram voice file ID existence.
- **Input/Output connections:** Input from `OpenClaw Agents` → true branch to `Generate Audio Response`, false branch to `Send a text message`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases/failures:** if workflow was triggered by schedule/webhook rather than Telegram, referenced message path may not exist.
- **Sub-workflow reference:** None.

#### Generate Audio Response
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi` — text-to-speech generation.
- **Configuration choices:**
  - input `{{$json.output}}`
  - voice `onyx`
  - resource `audio`
- **Key expressions/variables used:** `output`
- **Input/Output connections:** Input from `From audio to audio?` true branch → `Fix mimeType for Audio`
- **Version-specific requirements:** Type version `1.8`; requires OpenAI TTS-capable credentials.
- **Edge cases/failures:** long text, TTS quota, unsupported output assumptions.
- **Sub-workflow reference:** None.

#### Fix mimeType for Audio
- **Type and role:** `n8n-nodes-base.code` — adjusts binary mime type from `audio/mp3` to `audio/mpeg`.
- **Configuration choices:** Iterates over all binary properties and rewrites mime type if needed.
- **Key expressions/variables used:** `item.binary[propName].mimeType`
- **Input/Output connections:** Input from `Generate Audio Response` → output to `Send an audio file`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases/failures:** if binary is absent, code safely skips; if OpenAI output format changes, fix may be unnecessary or insufficient.
- **Sub-workflow reference:** None.

#### Send an audio file
- **Type and role:** `n8n-nodes-base.telegram` — sends binary audio reply to Telegram.
- **Configuration choices:**
  - operation `sendAudio`
  - `binaryData = true`
  - `chatId = {{ $('Get Message').item.json.message.from.id }}`
- **Key expressions/variables used:** original Telegram sender ID.
- **Input/Output connections:** Input from `Fix mimeType for Audio`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:** if triggered outside Telegram context, chat ID expression fails.
- **Sub-workflow reference:** None.

#### Send a text message
- **Type and role:** `n8n-nodes-base.telegram` — sends text reply to Telegram.
- **Configuration choices:**
  - text `{{$json.output}}`
  - chat ID from original Telegram sender
- **Key expressions/variables used:**
  - `$json.output`
  - `$('Get Message').item.json.message.from.id`
- **Input/Output connections:** Input from `From audio to audio?` false branch.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases/failures:** same Telegram-context dependency issue for non-Telegram entry points.
- **Sub-workflow reference:** None.

---

## 2.10 MCP Server Block: Gmail

### Overview
This block exposes Gmail operations through an MCP trigger. It is intended to be called by MCP clients such as the main agent’s `Gmail Agent`.

### Nodes Involved
- `MCP Gmail Trigger`
- `Get a message in Gmail`
- `Send a message in Gmail`
- `Get many messages in Gmail`
- `Reply to a message in Gmail`
- `Create a draft in Gmail`
- `Delete a draft in Gmail`
- `Delete a message in Gmail`
- `Add label to message in Gmail`
- `Remove label from message in Gmail`
- `Send message and wait for response in Gmail`

### Node Details
All Gmail tool nodes are `n8n-nodes-base.gmailTool`, version `2.1`, and connect via `ai_tool` to `MCP Gmail Trigger`.

#### MCP Gmail Trigger
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpTrigger` — MCP server endpoint for Gmail tools.
- **Configuration choices:** Path `63975752-0598-4992-9422-165a84c8798c`
- **Input/Output connections:** Entry MCP trigger receiving tool calls; all Gmail tools connect into it as AI tools.
- **Version-specific requirements:** Type version `2`.
- **Edge cases/failures:** public endpoint exposure, MCP client compatibility.

Common Gmail tool characteristics:
- AI parameters are provided using `$fromAI(...)`
- Require Gmail OAuth credentials
- Susceptible to auth expiry, invalid IDs, missing permissions, message formatting issues

Specific operations:
- **Get a message in Gmail:** get single message by `Message_ID`
- **Send a message in Gmail:** send email with `To`, `Subject`, `Message`
- **Get many messages in Gmail:** search with query and date filters; optional attachment download; optional `returnAll`
- **Reply to a message in Gmail:** reply by `Message_ID`
- **Create a draft in Gmail:** create draft with optional recipient, reply-to, thread
- **Delete a draft in Gmail:** delete draft by `Draft_ID`
- **Delete a message in Gmail:** deletes message but uses field label `Draft_ID`, likely a naming inconsistency
- **Add label to message in Gmail:** add labels by names/IDs; also uses `Draft_ID` field for message ID, likely inconsistent
- **Remove label from message in Gmail:** same inconsistency
- **Send message and wait for response in Gmail:** asynchronous mail flow; risk of long waiting windows/timeouts depending on implementation

Sub-workflow reference: None.

---

## 2.11 MCP Server Block: Google Calendar

### Overview
This block exposes Google Calendar operations through MCP.

### Nodes Involved
- `MCP Calendar Trigger`
- `Create an event in Google Calendar`
- `Get many events in Google Calendar`
- `Get an event in Google Calendar`
- `Delete an event in Google Calendar`
- `Get availability in a calendar in Google Calendar`

### Node Details
All tool nodes are `n8n-nodes-base.googleCalendarTool`, version `1.3`, connected as AI tools to `MCP Calendar Trigger`.

#### MCP Calendar Trigger
- **Type and role:** MCP server endpoint for Calendar operations.
- **Configuration choices:** Path `0f72eb90-2a1c-403e-98e1-901d76c2825c`
- **Edge cases/failures:** endpoint security and MCP client alignment.

Specific tools:
- **Create an event in Google Calendar:** requires `Start`, `End`, and `Calendar`
- **Get many events in Google Calendar:** search between `After` and `Before`; optional `returnAll`
- **Get an event in Google Calendar:** fetch single event by `Event_ID`
- **Delete an event in Google Calendar:** destructive action by `Event_ID`
- **Get availability in a calendar in Google Calendar:** checks free/busy between `Start_Time` and `End_Time`, resource `calendar`

Common failures:
- invalid calendar IDs,
- timezone formatting issues,
- auth scope problems,
- deleting wrong event if event ID ambiguous.

---

## 2.12 MCP Server Block: Google Drive

### Overview
This block exposes Google Drive operations through MCP.

### Nodes Involved
- `MCP Drive Trigger`
- `Download file in Google Drive`
- `Search files and folders in Google Drive`
- `Get many shared drives in Google Drive`
- `Create file from text in Google Drive`
- `Move file in Google Drive`

### Node Details
All tool nodes are `n8n-nodes-base.googleDriveTool`, version `3`, connected to `MCP Drive Trigger`.

#### MCP Drive Trigger
- **Type and role:** MCP server endpoint for Drive actions.
- **Configuration choices:** Path `031809b0-c277-4ff1-b65a-9e72294b1ff1`

Specific tools:
- **Download file in Google Drive:** download file by ID
- **Search files and folders in Google Drive:** query-based file/folder search with AI-defined limit
- **Get many shared drives in Google Drive:** list drives; `limit = 50`; optional `returnAll`
- **Create file from text in Google Drive:** creates text file named with `{{$now.format('yyyyLLdd')}}`, with AI-provided content, drive ID, and folder ID
- **Move file in Google Drive:** move file to destination drive/folder

Common failures:
- missing drive/folder permissions,
- invalid IDs,
- wrong shared drive contexts,
- binary size/download limitations.

---

## 2.13 MCP Server Block: Google Docs

### Overview
This block exposes Google Docs operations through MCP.

### Nodes Involved
- `MCP Docs Trigger`
- `Create a document in Google Docs`
- `Get a document in Google Docs`
- `Update a document in Google Docs`

### Node Details
All tool nodes are `n8n-nodes-base.googleDocsTool`, version `2`, connected to `MCP Docs Trigger`.

#### MCP Docs Trigger
- **Type and role:** MCP server endpoint for Google Docs.
- **Configuration choices:** Path `1f35317d-7b8e-48b9-a419-936ea5b63ae6`

Specific tools:
- **Create a document in Google Docs:** create doc with AI-defined title; folder is `default`
- **Get a document in Google Docs:** retrieve doc using `Doc_ID_or_URL`
- **Update a document in Google Docs:** performs insert action with AI-provided text

Common failures:
- invalid document URL/ID,
- formatting limitations,
- insufficient permissions.

---

## 2.14 MCP Server Block: Google Sheets

### Overview
This block exposes Google Sheets operations through MCP.

### Nodes Involved
- `MCP Sheet Trigger`
- `Get row(s) in sheet in Google Sheets`
- `Append or update row in sheet in Google Sheets`
- `Update row in sheet in Google Sheets`
- `Create sheet in Google Sheets`
- `Delete rows or columns from sheet in Google Sheets`
- `Clear sheet in Google Sheets`

### Node Details
All tool nodes are `n8n-nodes-base.googleSheetsTool`, version `4.7`, connected to `MCP Sheet Trigger`.

#### MCP Sheet Trigger
- **Type and role:** MCP server endpoint for Sheets.
- **Configuration choices:** Uses path `1f35317d-7b8e-48b9-a419-936ea5b63ae6`, which duplicates the Docs trigger path and likely needs correction.

Specific tools:
- **Get row(s) in sheet in Google Sheets:** fetch rows from document and sheet
- **Append or update row in sheet in Google Sheets:** auto-maps input data
- **Update row in sheet in Google Sheets:** update existing row(s)
- **Create sheet in Google Sheets:** create worksheet/tab with title in an existing spreadsheet
- **Delete rows or columns from sheet in Google Sheets:** deletes rows using `startIndex` and `numberToDelete`
- **Clear sheet in Google Sheets:** clears sheet with optional `keepFirstRow`

Common failures:
- duplicate MCP path collision,
- wrong document/sheet IDs,
- mapping mismatches,
- delete/clear being destructive.

---

## 2.15 MCP Server Block: Google Slides

### Overview
This block exposes Google Slides operations through MCP.

### Nodes Involved
- `MCP Slides Trigger`
- `Create a presentation in Google Slides`
- `Get a presentation in Google Slides`
- `Get slides from a presentation in Google Slides`
- `Replace text in a presentation in Google Slides`

### Node Details
All tool nodes are `n8n-nodes-base.googleSlidesTool`, version `2`, connected to `MCP Slides Trigger`.

#### MCP Slides Trigger
- **Type and role:** MCP server endpoint for Slides.
- **Configuration choices:** Also uses path `1f35317d-7b8e-48b9-a419-936ea5b63ae6`, duplicating Docs and Sheets, which should be fixed.

Specific tools:
- **Create a presentation in Google Slides:** create by title
- **Get a presentation in Google Slides:** fetch by `Presentation_ID`
- **Get slides from a presentation in Google Slides:** list slides with optional `returnAll`
- **Replace text in a presentation in Google Slides:** search/replace text pairs with optional `Revision_ID`

Common failures:
- duplicated MCP path,
- invalid presentation ID,
- text replacement mismatches,
- permission issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Message | telegramTrigger | Telegram entry point |  | Code | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Code | code | Authorize Telegram sender ID | Get Message | Switch2 | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Switch2 | switch | Route text/voice/image | Code | Get Text; Get voice message; Get image file | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Get Text | set | Normalize Telegram text input | Switch2 | OpenClaw Agents | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Get voice message | telegram | Download Telegram voice file | Switch2 | Transcribe recording | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Transcribe recording | @n8n/n8n-nodes-langchain.openAi | Transcribe voice to text | Get voice message | Get input text from voice | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Get input text from voice | set | Normalize transcript into agent input | Transcribe recording | OpenClaw Agents | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Get image file | telegram | Download Telegram photo | Switch2 | Upload image | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Upload image | ftp | Upload image to FTP | Get image file | Set Image Url | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Set Image Url | set | Build public image URL and normalize input | Upload image | OpenClaw Agents | ## STEP 2 - Set up Telegram<br>Connect to your Telegram Bot and replace xxx with the required information, specifically the Telegram ID. Set your FTP Space<br>You can send text, audio or image |
| Schedule Trigger | scheduleTrigger | Scheduled entry point |  | Set up istruction | ## STEP 2 - Set up recurrences<br>Please replace xxx with the requested information, in particular the Telegram ID and the prompt to be scheduled periodically, for example every morning "Send me the latest news on AI"<br><br>Also add any external input webhooks you want. |
| Set up istruction | set | Build scheduled prompt input | Schedule Trigger | OpenClaw Agents | ## STEP 2 - Set up recurrences<br>Please replace xxx with the requested information, in particular the Telegram ID and the prompt to be scheduled periodically, for example every morning "Send me the latest news on AI"<br><br>Also add any external input webhooks you want. |
| Webhook | webhook | HTTP entry point |  | Normalization | ## STEP 2 - Set up recurrences<br>Please replace xxx with the requested information, in particular the Telegram ID and the prompt to be scheduled periodically, for example every morning "Send me the latest news on AI"<br><br>Also add any external input webhooks you want. |
| Normalization | set | Normalize webhook payload | Webhook | OpenClaw Agents | ## STEP 2 - Set up recurrences<br>Please replace xxx with the requested information, in particular the Telegram ID and the prompt to be scheduled periodically, for example every morning "Send me the latest news on AI"<br><br>Also add any external input webhooks you want. |
| OpenClaw Agents | @n8n/n8n-nodes-langchain.agent | Main orchestration agent | Get Text; Get input text from voice; Set Image Url; Set up istruction; Normalization | From audio to audio? | ## STEP 3 - Set up OpenClaw agent<br>In the prompt system add ALL tools you want |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for main agent |  | OpenClaw Agents | ## STEP 3 - Set up OpenClaw agent<br>In the prompt system add ALL tools you want |
| Postgres Chat Memory | @n8n/n8n-nodes-langchain.memoryPostgresChat | Persistent conversation memory |  | OpenClaw Agents | ## STEP 3 - Set up OpenClaw agent<br>In the prompt system add ALL tools you want |
| Websearch Agent | perplexityTool | Live web search tool |  | OpenClaw Agents | ## STEP 3 - Set up OpenClaw agent<br>In the prompt system add ALL tools you want |
| Scraper Agent | n8n-nodes-scrapegraphai.scrapegraphAiTool | Web scraping/extraction tool |  | OpenClaw Agents | ## STEP 3 - Set up OpenClaw agent<br>In the prompt system add ALL tools you want |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Arithmetic tool |  | OpenClaw Agents | ## STEP 3 - Set up OpenClaw agent<br>In the prompt system add ALL tools you want |
| Gmail Agent | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client for Gmail |  | OpenClaw Agents | ## STEP 4 - Google MCP<br>Please configure MCP Server below (MCP Trigger) with the current MCP url |
| Google Drive Agent | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client for Drive |  | OpenClaw Agents | ## STEP 4 - Google MCP<br>Please configure MCP Server below (MCP Trigger) with the current MCP url |
| Google Calendar Agent | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client for Calendar |  | OpenClaw Agents | ## STEP 4 - Google MCP<br>Please configure MCP Server below (MCP Trigger) with the current MCP url |
| Google Docs Agent | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client for Docs |  | OpenClaw Agents | ## STEP 4 - Google MCP<br>Please configure MCP Server below (MCP Trigger) with the current MCP url |
| Google Sheets Agent | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client for Sheets |  | OpenClaw Agents | ## STEP 4 - Google MCP<br>Please configure MCP Server below (MCP Trigger) with the current MCP url |
| Google Slides Agent | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client for Slides |  | OpenClaw Agents | ## STEP 4 - Google MCP<br>Please configure MCP Server below (MCP Trigger) with the current MCP url |
| RAG Agent | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Retrieval tool over vector DB |  | OpenClaw Agents | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Embeddings OpenAI | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model for RAG |  | RAG Agent | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Reranker Cohere | @n8n/n8n-nodes-langchain.rerankerCohere | Reranker for RAG results |  | RAG Agent | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Image and Video Grok Agent | @n8n/n8n-nodes-langchain.toolWorkflow | Sub-workflow tool for media creation/editing |  | OpenClaw Agents | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Others MCP Client | @n8n/n8n-nodes-langchain.mcpClientTool | Placeholder external MCP client |  | OpenClaw Agents | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Others SubWF | @n8n/n8n-nodes-langchain.toolWorkflow | Placeholder sub-workflow tool |  | OpenClaw Agents | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Escalation | telegramHitlTool | Human approval/escalation tool |  | OpenClaw Agents | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| Send a text message in Telegram | telegramTool | Telegram tool used inside escalation/tooling context |  | Escalation | ## STEP 5 - Expansion tool<br>Configure [RAG from Zero](https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/), [Image and Video Grok Agent](https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/), ALL sub Workflow and extarnel MCP you want and Escalation Agent |
| From audio to audio? | if | Decide text vs audio reply | OpenClaw Agents | Generate Audio Response; Send a text message | ## STEP 6 - Output<br>If it is a voice message the agent responds with an audio, otherwise with text |
| Generate Audio Response | @n8n/n8n-nodes-langchain.openAi | Convert final text to speech | From audio to audio? | Fix mimeType for Audio | ## STEP 6 - Output<br>If it is a voice message the agent responds with an audio, otherwise with text |
| Fix mimeType for Audio | code | Correct audio MIME type for Telegram | Generate Audio Response | Send an audio file | ## STEP 6 - Output<br>If it is a voice message the agent responds with an audio, otherwise with text |
| Send an audio file | telegram | Send voice/audio reply to Telegram | Fix mimeType for Audio |  | ## STEP 6 - Output<br>If it is a voice message the agent responds with an audio, otherwise with text |
| Send a text message | telegram | Send text reply to Telegram | From audio to audio? |  | ## STEP 6 - Output<br>If it is a voice message the agent responds with an audio, otherwise with text |
| MCP Gmail Trigger | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server endpoint for Gmail tools |  |  |  |
| Get a message in Gmail | gmailTool | Get Gmail message by ID |  | MCP Gmail Trigger |  |
| Send a message in Gmail | gmailTool | Send Gmail email |  | MCP Gmail Trigger |  |
| Get many messages in Gmail | gmailTool | Search/list Gmail messages |  | MCP Gmail Trigger |  |
| Reply to a message in Gmail | gmailTool | Reply to Gmail message |  | MCP Gmail Trigger |  |
| Create a draft in Gmail | gmailTool | Create Gmail draft |  | MCP Gmail Trigger |  |
| Delete a draft in Gmail | gmailTool | Delete Gmail draft |  | MCP Gmail Trigger |  |
| Delete a message in Gmail | gmailTool | Delete Gmail message |  | MCP Gmail Trigger |  |
| Add label to message in Gmail | gmailTool | Add Gmail labels |  | MCP Gmail Trigger |  |
| Remove label from message in Gmail | gmailTool | Remove Gmail labels |  | MCP Gmail Trigger |  |
| Send message and wait for response in Gmail | gmailTool | Send email and wait for reply |  | MCP Gmail Trigger |  |
| MCP Calendar Trigger | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server endpoint for Calendar tools |  |  |  |
| Create an event in Google Calendar | googleCalendarTool | Create calendar event |  | MCP Calendar Trigger |  |
| Get many events in Google Calendar | googleCalendarTool | List calendar events |  | MCP Calendar Trigger |  |
| Get an event in Google Calendar | googleCalendarTool | Get single event |  | MCP Calendar Trigger |  |
| Delete an event in Google Calendar | googleCalendarTool | Delete calendar event |  | MCP Calendar Trigger |  |
| Get availability in a calendar in Google Calendar | googleCalendarTool | Check calendar availability |  | MCP Calendar Trigger |  |
| MCP Drive Trigger | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server endpoint for Drive tools |  |  |  |
| Download file in Google Drive | googleDriveTool | Download Drive file |  | MCP Drive Trigger |  |
| Search files and folders in Google Drive | googleDriveTool | Search Drive files/folders |  | MCP Drive Trigger |  |
| Get many shared drives in Google Drive | googleDriveTool | List shared drives |  | MCP Drive Trigger |  |
| Create file from text in Google Drive | googleDriveTool | Create Drive text file |  | MCP Drive Trigger |  |
| Move file in Google Drive | googleDriveTool | Move Drive file |  | MCP Drive Trigger |  |
| MCP Docs Trigger | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server endpoint for Docs tools |  |  |  |
| Create a document in Google Docs | googleDocsTool | Create Google Doc |  | MCP Docs Trigger |  |
| Get a document in Google Docs | googleDocsTool | Get Google Doc content |  | MCP Docs Trigger |  |
| Update a document in Google Docs | googleDocsTool | Update Google Doc |  | MCP Docs Trigger |  |
| MCP Sheet Trigger | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server endpoint for Sheets tools |  |  |  |
| Get row(s) in sheet in Google Sheets | googleSheetsTool | Read rows from sheet |  | MCP Sheet Trigger |  |
| Append or update row in sheet in Google Sheets | googleSheetsTool | Append/update rows |  | MCP Sheet Trigger |  |
| Update row in sheet in Google Sheets | googleSheetsTool | Update existing row |  | MCP Sheet Trigger |  |
| Create sheet in Google Sheets | googleSheetsTool | Create worksheet/tab |  | MCP Sheet Trigger |  |
| Delete rows or columns from sheet in Google Sheets | googleSheetsTool | Delete rows/columns |  | MCP Sheet Trigger |  |
| Clear sheet in Google Sheets | googleSheetsTool | Clear worksheet data |  | MCP Sheet Trigger |  |
| MCP Slides Trigger | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server endpoint for Slides tools |  |  |  |
| Create a presentation in Google Slides | googleSlidesTool | Create presentation |  | MCP Slides Trigger |  |
| Get a presentation in Google Slides | googleSlidesTool | Get presentation metadata |  | MCP Slides Trigger |  |
| Get slides from a presentation in Google Slides | googleSlidesTool | List slides |  | MCP Slides Trigger |  |
| Replace text in a presentation in Google Slides | googleSlidesTool | Replace text in slides |  | MCP Slides Trigger |  |
| Sticky Note | stickyNote | Documentation note |  |  |  |
| Sticky Note1 | stickyNote | Documentation note |  |  |  |
| Sticky Note2 | stickyNote | Documentation note |  |  |  |
| Sticky Note3 | stickyNote | Documentation note |  |  |  |
| Sticky Note4 | stickyNote | Documentation note |  |  |  |
| Sticky Note5 | stickyNote | Documentation note |  |  |  |
| Sticky Note6 | stickyNote | Documentation note |  |  |  |
| Sticky Note9 | stickyNote | Documentation note |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `OpenClaw n8n Agent`.

2. **Add Telegram inbound trigger**
   - Create `Telegram Trigger` node named `Get Message`.
   - Configure Telegram credentials from BotFather bot token.
   - Set updates to `message`.

3. **Add authorization gate**
   - Create `Code` node named `Code`.
   - Paste logic that checks `message.from.id` against your Telegram user ID.
   - Replace `XXX` with your authorized Telegram numeric ID.
   - Connect `Get Message -> Code`.

4. **Add content router**
   - Create `Switch` node named `Switch2`.
   - Add 3 outputs with renamed keys:
     - `Text`: condition `message.text exists`
     - `Audio`: condition `message.voice.file_id exists`
     - `Immagine`: condition `message.photo[0] exists`
   - Connect `Code -> Switch2`.

5. **Build text branch**
   - Create `Set` node named `Get Text`.
   - Add fields:
     - `chatInput` = `{{$json.message.text}}`
     - `sessionId` = `{{$json.message.from.id}}`
   - Connect `Switch2(Text) -> Get Text`.

6. **Build voice branch download**
   - Create `Telegram` node named `Get voice message`.
   - Resource: `file`.
   - `fileId` = `{{ $('Get Message').item.json.message.voice.file_id }}`
   - Connect `Switch2(Audio) -> Get voice message`.

7. **Build voice transcription**
   - Create `OpenAI` LangChain node named `Transcribe recording`.
   - Resource: `audio`
   - Operation: `transcribe`
   - Set language to `it` or your preferred language.
   - Connect `Get voice message -> Transcribe recording`.

8. **Normalize voice transcript**
   - Create `Set` node named `Get input text from voice`.
   - Fields:
     - `chatInput` = `{{$json.text}}`
     - `sessionId` = `{{ $('Get Message').item.json.message.from.id }}`
   - Connect `Transcribe recording -> Get input text from voice`.

9. **Build image branch download**
   - Create `Telegram` node named `Get image file`.
   - Resource: `file`
   - `fileId` = `{{$json.message.photo[2].file_id}}`
   - Connect `Switch2(Immagine) -> Get image file`.
   - If you want more robustness, use the last photo size dynamically instead of fixed `[2]`.

10. **Upload image to FTP**
    - Create `FTP` node named `Upload image`.
    - Operation: `upload`
    - Path: `/YOUR_PATH/{{ $binary.data.fileName }}`
    - Configure FTP credentials.
    - Connect `Get image file -> Upload image`.

11. **Create public image URL**
    - Create `Set` node named `Set Image Url`.
    - Fields:
      - `image_url` = `https://YOUR_DOMAIN/{{ $binary.data.fileName }}`
      - `chatInput` = `{{ $('Get Message').item.json.message.caption || "" }}`
      - `sessionId` = `{{ $('Get Message').item.json.message.from.id }}`
    - Connect `Upload image -> Set Image Url`.

12. **Add scheduled entry**
    - Create `Schedule Trigger` named `Schedule Trigger`.
    - Configure desired recurrence.
    - Create `Set` node named `Set up istruction`.
    - Fields:
      - `chatInput` = your fixed scheduled prompt, e.g. `Send me the latest AI news`
      - `sessionId` = stable identifier like your Telegram ID
    - Connect `Schedule Trigger -> Set up istruction`.

13. **Add webhook entry**
    - Create `Webhook` node named `Webhook`.
    - Choose a path.
    - Create `Set` node named `Normalization`.
    - If your caller sends Telegram-like JSON, map:
      - `chatInput = {{$json.message.text}}`
      - `sessionId = {{$json.message.from.id}}`
    - Connect `Webhook -> Normalization`.
   - If your caller sends custom JSON, adapt fields accordingly.

14. **Add main AI agent**
    - Create `AI Agent` node named `OpenClaw Agents`.
    - Set prompt mode to define custom system message.
    - User text input should include:
      - `{{$json.chatInput ?? ''}}`
      - `Image Url (if exist): {{$json.image_url ?? ''}}`
    - Paste and customize the orchestrator system prompt.
    - Connect:
      - `Get Text -> OpenClaw Agents`
      - `Get input text from voice -> OpenClaw Agents`
      - `Set Image Url -> OpenClaw Agents`
      - `Set up istruction -> OpenClaw Agents`
      - `Normalization -> OpenClaw Agents`

15. **Add Gemini model**
    - Create `Google Gemini Chat Model` node.
    - Configure Gemini API credentials.
    - Connect it to `OpenClaw Agents` using AI language model connection.

16. **Add PostgreSQL memory**
    - Create `Postgres Chat Memory` node.
    - Configure PostgreSQL credentials.
    - Ensure the memory uses the incoming `sessionId`.
    - Connect it to `OpenClaw Agents` using AI memory connection.

17. **Add Perplexity web search tool**
    - Create `Perplexity Tool` node named `Websearch Agent`.
    - Message content should be provided by AI via generated fields.
    - Configure Perplexity credentials.
    - Connect as AI tool to `OpenClaw Agents`.

18. **Add ScrapeGraphAI tool**
    - Create `ScrapeGraphAI Tool` named `Scraper Agent`.
    - Expose AI-driven inputs:
      - website URL
      - extraction prompt
      - optional flags for JS rendering, pagination, scrolling, output schema
    - Configure ScrapeGraphAI API key.
    - Connect as AI tool to `OpenClaw Agents`.

19. **Add calculator tool**
    - Create `Calculator` tool node.
    - Connect as AI tool to `OpenClaw Agents`.

20. **Add RAG tool**
    - Create `Qdrant Vector Store` node named `RAG Agent`.
    - Set mode to `retrieve-as-tool`.
    - Enable reranker.
    - Choose your Qdrant collection.
    - Create `Embeddings OpenAI` node and connect as AI embedding.
    - Create `Reranker Cohere` node and connect as AI reranker.
    - Configure OpenAI and Cohere credentials.
    - Connect `RAG Agent` as AI tool to `OpenClaw Agents`.

21. **Add media sub-workflow tool**
    - Create `Tool Workflow` node named `Image and Video Grok Agent`.
    - Select the sub-workflow that handles image/video generation and editing.
    - Define inputs:
      - `tool_name`
      - `query`
      - `duration`
      - `video_url`
      - `image_url`
    - Connect as AI tool to `OpenClaw Agents`.

22. **Add placeholder/extra sub-workflow**
    - Create another `Tool Workflow` node named `Others SubWF`.
    - Replace the current placeholder with your own workflow.
    - Expose the input schema needed by that sub-workflow.
    - Connect as AI tool to `OpenClaw Agents`.

23. **Add external MCP placeholder**
    - Create `MCP Client Tool` node named `Others MCP Client`.
    - Configure the external MCP endpoint URL.
    - Connect as AI tool to `OpenClaw Agents`.

24. **Add escalation tool**
    - Create `Telegram HITL Tool` node named `Escalation`.
    - Configure Telegram credentials.
    - Use AI-provided `chatId`.
    - Set approval type to `double` if desired.
    - Connect as AI tool to `OpenClaw Agents`.

25. **Optionally add Telegram send tool for escalation flows**
    - Create `Telegram Tool` named `Send a text message in Telegram`.
    - Expose `text` and `chatId` to AI.
    - In this JSON it is connected to `Escalation`; you may instead prefer connecting it directly to the main agent if you want normal outbound tool usage.

26. **Add Gmail MCP server**
    - Create `MCP Trigger` named `MCP Gmail Trigger`.
    - Add Gmail Tool nodes for:
      - get message
      - send message
      - get many messages
      - reply
      - create draft
      - delete draft
      - delete message
      - add label
      - remove label
      - send and wait
    - Configure Gmail OAuth2 credentials.
    - For each tool, expose required fields via AI-generated inputs.
    - Connect every Gmail tool to `MCP Gmail Trigger` via AI tool connections.

27. **Add Gmail MCP client to main agent**
    - Create `MCP Client Tool` named `Gmail Agent`.
    - Set endpoint URL to the public MCP URL corresponding to `MCP Gmail Trigger`.
    - Connect as AI tool to `OpenClaw Agents`.

28. **Add Google Calendar MCP server**
    - Create `MCP Trigger` named `MCP Calendar Trigger`.
    - Add Calendar tool nodes for create/get/getAll/delete/availability.
    - Configure Google Calendar OAuth2 credentials.
    - Connect all tools into the trigger.

29. **Add Calendar MCP client**
    - Create `MCP Client Tool` named `Google Calendar Agent`.
    - Set endpoint URL to the Calendar MCP trigger public URL.
    - Connect as AI tool to `OpenClaw Agents`.

30. **Add Google Drive MCP server**
    - Create `MCP Trigger` named `MCP Drive Trigger`.
    - Add Drive tool nodes for download/search/list drives/create text file/move file.
    - Configure Google Drive OAuth2 credentials.
    - Connect all tools into the trigger.

31. **Add Drive MCP client**
    - Create `MCP Client Tool` named `Google Drive Agent`.
    - Set endpoint URL to the Drive MCP trigger URL.
    - Connect as AI tool to `OpenClaw Agents`.

32. **Add Google Docs MCP server**
    - Create `MCP Trigger` named `MCP Docs Trigger`.
    - Add Docs tool nodes for create/get/update.
    - Configure Google Docs OAuth2 credentials.
    - Connect all tools into the trigger.

33. **Add Docs MCP client**
    - Create `MCP Client Tool` named `Google Docs Agent`.
    - Set endpoint URL to the Docs MCP trigger URL.
    - Connect as AI tool to `OpenClaw Agents`.

34. **Add Google Sheets MCP server**
    - Create `MCP Trigger` named `MCP Sheet Trigger`.
    - Add Sheets tool nodes for read, append/update, update, create sheet, delete, clear.
    - Configure Google Sheets OAuth2 credentials.
    - Connect all tools into the trigger.
    - Use a unique MCP path; do not reuse the Docs path.

35. **Add Sheets MCP client**
    - Create `MCP Client Tool` named `Google Sheets Agent`.
    - Set endpoint URL to the Sheets MCP trigger URL.
    - Connect as AI tool to `OpenClaw Agents`.

36. **Add Google Slides MCP server**
    - Create `MCP Trigger` named `MCP Slides Trigger`.
    - Add Slides tool nodes for create, get presentation, get slides, replace text.
    - Configure Google Slides OAuth2 credentials.
    - Connect all tools into the trigger.
    - Use a unique MCP path; do not reuse Docs or Sheets path.

37. **Add Slides MCP client**
    - Create `MCP Client Tool` named `Google Slides Agent`.
    - Set endpoint URL to the Slides MCP trigger URL.
    - Connect as AI tool to `OpenClaw Agents`.

38. **Build output routing**
    - Add `If` node named `From audio to audio?`
    - Condition: original Telegram message has `voice.file_id`
    - Connect `OpenClaw Agents -> From audio to audio?`

39. **Add text-to-speech branch**
    - Create `OpenAI` audio node named `Generate Audio Response`.
    - Input = `{{$json.output}}`
    - Voice = `onyx`
    - Resource = `audio`
    - Connect true branch of `From audio to audio? -> Generate Audio Response`.

40. **Fix audio MIME type**
    - Create `Code` node named `Fix mimeType for Audio`.
    - Add JS to rewrite `audio/mp3` to `audio/mpeg`.
    - Connect `Generate Audio Response -> Fix mimeType for Audio`.

41. **Send audio reply**
    - Create `Telegram` node named `Send an audio file`.
    - Operation `sendAudio`
    - Enable binary data
    - `chatId = {{ $('Get Message').item.json.message.from.id }}`
    - Connect `Fix mimeType for Audio -> Send an audio file`.

42. **Send text reply**
    - Create `Telegram` node named `Send a text message`.
    - Text = `{{$json.output}}`
    - `chatId = {{ $('Get Message').item.json.message.from.id }}`
    - Connect false branch of `From audio to audio? -> Send a text message`.

43. **Add sticky notes/documentation**
    - Recreate optional sticky notes for setup guidance, tool expansion, and output behavior.

44. **Replace all placeholders**
    - Telegram authorized user ID
    - FTP folder path
    - public image domain
    - scheduled prompt/session ID
    - MCP endpoint URLs
    - Qdrant collection
    - any placeholder sub-workflow IDs

45. **Credential checklist**
    - Telegram Bot
    - OpenAI
    - Google Gemini
    - PostgreSQL
    - Perplexity
    - ScrapeGraphAI
    - Cohere
    - FTP
    - Google OAuth2 for Gmail/Calendar/Drive/Docs/Sheets/Slides
    - Qdrant connection if applicable

46. **Recommended corrections before production**
    - Give unique MCP paths to Docs, Sheets, and Slides
    - Fix misconfigured MCP client endpoints
    - Standardize `sessionId` type across all branches
    - Redesign non-Telegram output handling for webhook/schedule paths
    - Validate Gmail parameter naming inconsistencies such as message ID vs draft ID labels

### Sub-workflow setup expectations

#### Image and Video Grok Agent
Expected inputs:
- `tool_name` string
- `query` string
- `duration` number
- `video_url` string optional
- `image_url` string optional

Expected output:
- structured result consumable by the parent agent, ideally with media URL and status message.

#### Others SubWF
Expected inputs:
- currently placeholder; define according to your chosen automation.

Expected output:
- concise structured response that the agent can summarize back to the user.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| I've described my basic idea in this video. | https://www.youtube.com/watch?v=DCo_HsLQ1aY |
| By adapting the system prompt, inserting subworkflows or MCP servers and adjusting with webhooks many of the workflows I have developed on this page, it is possible to potentially extend the template infinitely. | https://n8n.io/creators/n3witalia/ |
| Configure RAG from Zero for the RAG block. | https://n8n.io/workflows/7647-build-a-self-updating-rag-system-with-openai-google-gemini-qdrant-and-google-drive/ |
| Configure Image and Video Grok Agent for media generation/editing. | https://n8n.io/workflows/13182-grok-imagine-video-chatbot-generate-and-modify-videos-via-natural-language/ |
| Subscribe to my new YouTube channel. Here I’ll share videos and Shorts with practical tutorials and FREE templates for n8n. | https://youtube.com/@n3witalia |
| Contact for consulting and support. | mailto:info@n3w.it |
| LinkedIn profile for support/contact. | https://www.linkedin.com/in/davideboizza/ |

## Final implementation notes

- The workflow is architecturally strong but still partly template-grade: several MCP clients are incomplete, some paths are duplicated, and a few placeholders remain intentionally unresolved.
- The main agent is only as capable as its tool descriptions and real endpoint setup. The prompt must stay aligned with actual available tools.
- Scheduled and webhook entry points currently reuse Telegram-oriented output assumptions. If you intend to use those heavily, add dedicated response handling instead of relying on Telegram sender context.