Build a Telegram AI assistant with MemMachine, OpenAI, and voice support

https://n8nworkflows.xyz/workflows/build-a-telegram-ai-assistant-with-memmachine--openai--and-voice-support-12568


# Build a Telegram AI assistant with MemMachine, OpenAI, and voice support

## 1. Workflow Overview

This workflow implements a Telegram-based AI assistant that supports **text and voice messages**, uses **MemMachine** for **persistent cross-session memory**, and can call **Google tools (Gmail, Sheets, Calendar)** through an **MCP (Model Context Protocol) architecture**. The assistant stores each user message, retrieves relevant prior memories, generates a context-aware response with OpenAI, stores the assistant response back to memory, and replies in Telegram.

### 1.1 Input Reception (Telegram)
Receives Telegram updates and routes messages into **voice** vs **text** paths.

### 1.2 Message Normalization (Voice/Text → unified schema)
For voice: downloads audio → transcribes via OpenAI Whisper → extracts consistent fields (text/chat/user metadata).  
For text: extracts the same consistent fields directly.

### 1.3 Persistent Memory (MemMachine)
Stores the incoming user query, searches memory for relevant history filtered by the user identity, then formats/sorts memories into a conversation history string.

### 1.4 AI Reasoning + Tool Access (OpenAI Agent + MCP tools)
Runs an n8n LangChain **AI Agent** with a system prompt embedding conversation history. The agent can access tools via MCP (Gmail/Sheets/Calendar).

### 1.5 Response Persistence + Delivery
Extracts the agent output, stores the assistant response in MemMachine, then sends the reply back to Telegram.

### 1.6 MCP Server Endpoint (separate entrypoint)
A dedicated **MCP Server Trigger** exposes Gmail/Sheets/Calendar “tool endpoints” that the MCP client (used by the AI agent) can call.

---

## 2. Block-by-Block Analysis

### Block A — Input Reception & Routing (Telegram)
**Overview:** Listens for Telegram messages and branches into voice vs text handling based on the payload structure.

**Nodes involved:**
- `1. Telegram Trigger`
- `2. Message Type`

#### Node: 1. Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point; receives Telegram updates.
- **Configuration (interpreted):**
  - Update type: `message`
  - Uses a Telegram credential (bot token) configured in n8n.
- **Input/Output:**
  - **Output →** `2. Message Type`
- **Version notes:** typeVersion `1.2` (Telegram nodes in n8n; UI fields may vary slightly by n8n version).
- **Failure/edge cases:**
  - Telegram credential missing/invalid → auth failures.
  - Bot not set with webhook / conflicting webhook set elsewhere → no events.
  - Non-message updates ignored (only `message` subscribed).

#### Node: 2. Message Type
- **Type / role:** `switch` — routes flow.
- **Configuration choices:**
  - Rule 1 (“Voice”): checks existence of `{{$json.message.voice.file_id}}`
  - Rule 2 (“Text”): checks existence of `{{$json.message.text}}`
  - Each rule **renames output** to “Voice” and “Text”.
- **Input/Output:**
  - **Input ←** `1. Telegram Trigger`
  - **Voice output →** `3a. Download Voice`
  - **Text output →** `3d. Extract Text Data`
- **Edge cases:**
  - Photos/documents/stickers: neither condition matches → workflow ends (no default route configured).
  - Telegram “caption” messages (media with caption) won’t match `message.text` unless adapted.

**Sticky note context:** “Voice Processing”, “Text Processing”.

---

### Block B — Voice Processing (Download → Whisper → Normalize)
**Overview:** For voice messages, downloads the audio file, transcribes it to text, then normalizes fields to match the text path output.

**Nodes involved:**
- `3a. Download Voice`
- `3b. Transcribe Voice`
- `3c. Extract Voice Data`

#### Node: 3a. Download Voice
- **Type / role:** `telegram` (resource: file) — downloads Telegram voice file.
- **Configuration choices:**
  - `resource: file`
  - `fileId: {{$json.message.voice.file_id}}`
- **Input/Output:**
  - **Input ←** `2. Message Type` (Voice branch)
  - **Output →** `3b. Transcribe Voice`
- **Failure/edge cases:**
  - File expired/unavailable or Telegram API errors → download fails.
  - Large files/timeouts depending on n8n and Telegram limits.

#### Node: 3b. Transcribe Voice
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (Audio Transcription) — Whisper transcription.
- **Configuration choices:**
  - `resource: audio`
  - `operation: transcribe`
  - Requires OpenAI credentials in n8n.
- **Input/Output:**
  - **Input ←** `3a. Download Voice` (audio binary)
  - **Output →** `3c. Extract Voice Data`
- **Version notes:** typeVersion `1.6` (LangChain OpenAI node).
- **Failure/edge cases:**
  - Missing OpenAI credential / billing / model availability issues.
  - Audio format issues if Telegram returns unexpected binary metadata.
  - Rate limits/timeouts.

#### Node: 3c. Extract Voice Data
- **Type / role:** `set` — normalizes the transcribed result into the workflow’s standard schema.
- **Configuration choices (fields produced):**
  - `text = {{$json.text}}` (transcription result from previous node)
  - `chat_id = {{ $('1. Telegram Trigger').item.json.message.chat.id }}`
  - `user_id = {{ $('1. Telegram Trigger').item.json.message.from.id }}`
  - `username = {{ from.username || from.first_name }}` (via Telegram Trigger data)
  - `customer_email = {{ from.id }}@telegram.user` (synthetic stable identifier)
- **Input/Output:**
  - **Input ←** `3b. Transcribe Voice`
  - **Output →** `4. Store User Query`
- **Edge cases:**
  - If transcription returns empty/undefined `text`, downstream memory search/agent may behave poorly.
  - Reliance on cross-node reference to `1. Telegram Trigger` means the trigger item must be present and aligned (it is in this linear branch).

**Sticky note context:** “Voice path: Download → Transcribe (Whisper) → Extract data”.

---

### Block C — Text Processing (Normalize)
**Overview:** Extracts text and user/chat metadata directly from Telegram, producing the same standard schema as the voice path.

**Nodes involved:**
- `3d. Extract Text Data`

#### Node: 3d. Extract Text Data
- **Type / role:** `set` — normalizes Telegram text message to standard schema.
- **Configuration choices (fields produced):**
  - `text = {{$json.message.text}}`
  - `chat_id = {{$json.message.chat.id}}`
  - `user_id = {{$json.message.from.id}}`
  - `username = {{$json.message.from.username || $json.message.from.first_name}}`
  - `customer_email = {{$json.message.from.id}}@telegram.user`
- **Input/Output:**
  - **Input ←** `2. Message Type` (Text branch)
  - **Output →** `4. Store User Query`
- **Edge cases:**
  - Telegram commands (`/start`) are still text; may want special handling.
  - Empty text (rare) or unsupported message types won’t reach here.

**Sticky note context:** “Text path: Extract message data”.

---

### Block D — MemMachine Persistent Memory (Store → Search → Format)
**Overview:** Stores the user message in MemMachine, retrieves up to 30 relevant memories filtered to the same user, then formats them chronologically into a conversation history string for the AI agent.

**Nodes involved:**
- `4. Store User Query`
- `5. Search Memory`
- `6. Format Memory`

#### Node: 4. Store User Query
- **Type / role:** `httpRequest` — POST a “user” message to MemMachine `/api/v2/memories`.
- **Configuration choices:**
  - URL: `http://host.docker.internal:8080/api/v2/memories`
  - Method: `POST`
  - JSON body includes:
    - `org_id`, `project_id` (placeholders `your-org-id`, `your-project-id`)
    - `messages[0]`:
      - `content: {{$json.text}}`
      - `producer: {{$json.customer_email}}`
      - `produced_for: assistant`
      - `role: user`
      - `metadata`: customer_email, channel=telegram, username, user_id, timestamp=`{{$now.toISO()}}`
  - Header: `Content-Type: application/json`
- **Input/Output:**
  - **Input ←** normalized item from `3c` or `3d`
  - **Output →** `5. Search Memory`
- **Version notes:** typeVersion `4.1` (HTTP Request node).
- **Failure/edge cases:**
  - MemMachine not running / wrong host mapping:
    - `host.docker.internal` works in Docker Desktop (Mac/Windows) and some Linux setups; may fail on Linux without extra config.
  - Wrong `org_id/project_id` → 4xx from MemMachine.
  - MemMachine schema changes between versions could break `messages` format.

#### Node: 5. Search Memory
- **Type / role:** `httpRequest` — searches MemMachine for relevant memories.
- **Configuration choices:**
  - URL: `http://host.docker.internal:8080/api/v2/memories/search`
  - Method: `POST`
  - JSON body:
    - `org_id`, `project_id`
    - `query: {{ $input.item.json.text }}`
    - `top_k: 30`
    - `filter: "metadata.customer_email='{{ $input.item.json.customer_email }}'"`
    - `types: ["episodic","semantic"]`
- **Input/Output:**
  - **Input ←** `4. Store User Query`
  - **Output →** `6. Format Memory`
- **Failure/edge cases:**
  - Same connectivity/auth issues as node 4.
  - If filter syntax differs in MemMachine version, may return empty results.
  - If response structure differs, node 6 parsing may fail.

#### Node: 6. Format Memory
- **Type / role:** `code` — normalizes MemMachine search response into `conversation_history`.
- **What the code does (interpreted):**
  - Reads the first incoming JSON payload: `memoryData`.
  - Attempts to locate memory content at `memoryData.content` or uses `memoryData` directly.
  - Extracts episodic memory episodes from:
    - `content.episodic_memory.long_term_memory.episodes[]`
    - `content.episodic_memory.short_term_memory.episodes[]`
  - Merges both arrays and **sorts chronologically** using timestamp fallback order:
    - `timestamp` OR `created_at` OR `metadata.timestamp`
  - Builds a text transcript:
    - Keeps only items with non-empty `content`
    - Takes last 20
    - Formats each line as `[role]: content` where role resolves from `producer_role` or `role`
  - Pulls the “current input fields” from `$('4. Store User Query').first()?.json` (note: this is the HTTP response, not necessarily the original message) and outputs:
    - `text, chat_id, customer_email, username`
    - `conversation_history` (or “No previous history.”)
    - `total_memories`
- **Input/Output:**
  - **Input ←** `5. Search Memory`
  - **Output →** `7. AI Agent`
- **Version notes:** Code node typeVersion `2`.
- **Failure/edge cases / important nuance:**
  - **Potential data mismatch:** It reads user fields from the output of `4. Store User Query`, which is MemMachine’s response, not guaranteed to contain `text/chat_id/...`. If MemMachine does not echo these fields, `text` and IDs may become `undefined`.
    - Robust fix: carry forward the normalized Telegram fields explicitly (e.g., using Merge node or storing originals before HTTP calls).
  - If MemMachine search response doesn’t include the expected nested structure, `long_term_memory`/`short_term_memory` may be missing; the code handles missing arrays safely but may produce empty history.
  - Date parsing: invalid timestamps become `Invalid Date` → sort order can degrade.

**Sticky note context:** “Memory system: Store message → Search history (30 memories) → Sort chronologically”.

---

### Block E — AI Processing + MCP Tooling (Agent + Model + MCP client)
**Overview:** Builds an AI Agent prompt containing conversation history and connects it to an OpenAI chat model and an MCP client, enabling tool usage.

**Nodes involved:**
- `7. AI Agent`
- `OpenAI Chat Model`
- `MCP Client`

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the LLM to the agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - No built-in tools configured here (tools are provided via MCP client).
- **Input/Output connections:**
  - **Output (ai_languageModel) →** `7. AI Agent`
- **Failure/edge cases:**
  - OpenAI credential missing, model not available, rate limiting.

#### Node: MCP Client
- **Type / role:** `mcpClientTool` — exposes MCP tools to the agent.
- **Configuration choices:**
  - Default options (not explicitly set in JSON).
  - Must be configured in n8n UI to connect to the MCP server endpoint (implicitly represented by this same workflow’s MCP Server Trigger).
- **Input/Output:**
  - **Output (ai_tool) →** `7. AI Agent`
- **Failure/edge cases:**
  - If MCP server trigger workflow endpoint is not reachable/published, tool calls fail.
  - If authentication is required by MCP setup, misconfiguration causes tool invocation errors.

#### Node: 7. AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates reasoning, calls tools, returns final response.
- **Configuration choices:**
  - User text: `={{ $json.text }}`
  - System message dynamically injects:
    - `username`, `customer_email`
    - `conversation_history`
    - tool descriptions and behavioral instructions (concise, confirm important actions)
    - current date: `{{ $now }}`
    - memories count: `{{ $json.total_memories }}`
  - Prompt type: “define” (custom system prompt).
- **Inputs/Outputs:**
  - **Input (main) ←** `6. Format Memory`
  - **Input (ai_languageModel) ←** `OpenAI Chat Model`
  - **Input (ai_tool) ←** `MCP Client`
  - **Output (main) →** `8. Extract Response`
- **Failure/edge cases:**
  - If `conversation_history` is huge, prompt may exceed context limits (mitigated by last 20 lines).
  - If `text` is undefined (see node 6 nuance), agent may respond incorrectly.
  - Tool calls may produce partial failures; agent output may contain error text unless handled.

**Sticky note context:** “AI + Tools: Process with context, access Gmail/Sheets/Calendar via MCP”.

---

### Block F — Response Flow (Extract → Store → Send)
**Overview:** Takes the agent output, stores it as assistant memory, then sends it back to Telegram chat.

**Nodes involved:**
- `8. Extract Response`
- `9. Store AI Response`
- `10. Send Response`

#### Node: 8. Extract Response
- **Type / role:** `set` — maps the agent output to `ai_response` and restores identifiers for downstream.
- **Configuration choices:**
  - `ai_response = {{$json.output}}`
  - `customer_email/username/chat_id` pulled from `6. Format Memory`
- **Input/Output:**
  - **Input ←** `7. AI Agent`
  - **Output →** `9. Store AI Response`
- **Edge cases:**
  - If agent output field is not `output` (varies by agent node versions/config), `ai_response` could be empty.
  - If node 6 produced undefined IDs, Telegram send will fail.

#### Node: 9. Store AI Response
- **Type / role:** `httpRequest` — stores assistant response in MemMachine.
- **Configuration choices:**
  - URL: `http://host.docker.internal:8080/api/v2/memories`
  - Method: POST
  - JSON body:
    - `org_id`, `project_id`
    - message content: `{{$json.ai_response}}`
    - `producer: assistant`
    - `produced_for: {{$json.customer_email}}`
    - `role: assistant`
    - metadata: customer_email, channel=telegram, username, timestamp
- **Input/Output:**
  - **Input ←** `8. Extract Response`
  - **Output →** `10. Send Response`
- **Failure/edge cases:**
  - MemMachine connectivity/ID issues.
  - If `ai_response` contains Markdown that MemMachine rejects (unlikely) or size limits.

#### Node: 10. Send Response
- **Type / role:** `telegram` — sends a message back to the originating chat.
- **Configuration choices:**
  - `chatId = {{$json.chat_id}}`
  - `text = {{$json.ai_response}}`
  - `parse_mode: Markdown`
  - `appendAttribution: false`
- **Input/Output:**
  - **Input ←** `9. Store AI Response`
  - No downstream nodes.
- **Failure/edge cases:**
  - Invalid Markdown can cause Telegram send errors (common with unescaped `_`, `*`, `[`, `]`).
  - Chat ID missing/invalid → send fails.
  - Telegram rate limits.

**Sticky note context:** “Response: Extract → Store in memory → Send to Telegram”.

---

### Block G — MCP Server (Tool Endpoint Entry Point)
**Overview:** Provides an MCP server endpoint inside n8n that exposes Gmail/Sheets/Calendar tools for the agent to call indirectly via the MCP client tool.

**Nodes involved:**
- `MCP Server Trigger`
- `Gmail Tool`
- `Google Sheets Tool`
- `Google Calendar Tool`

#### Node: MCP Server Trigger
- **Type / role:** `mcpTrigger` — entry point for MCP tool calls.
- **Configuration choices:**
  - Path is set to a UUID-like string: `298e8ee6-95f2-408b-97df-af61f48a20ec`
- **Connections:**
  - Receives `ai_tool` connections from the tool nodes (they register with this trigger).
- **Failure/edge cases:**
  - If n8n instance URL/public access is not configured properly, MCP client cannot reach it.
  - If MCP requires authentication in your environment and it’s not set, tools won’t execute.

#### Node: Gmail Tool
- **Type / role:** `gmailTool` — AI-callable Gmail operations (here focused on sending).
- **Configuration choices:**
  - Uses `$fromAI()` expressions for:
    - To, Subject, Message
  - Requires Google OAuth2 credential with Gmail scopes.
- **Connections:**
  - **ai_tool →** `MCP Server Trigger`
- **Failure/edge cases:**
  - Missing Gmail permissions/scopes; token expired.
  - AI may provide invalid email addresses or empty subject/body—consider validation.

#### Node: Google Sheets Tool
- **Type / role:** `googleSheetsTool` — AI-callable Sheets operations.
- **Configuration choices:**
  - Document and Sheet selected via `$fromAI('Document')` and `$fromAI('Sheet')`
  - Requires Google OAuth2 credential with Sheets scopes.
- **Connections:**
  - **ai_tool →** `MCP Server Trigger`
- **Failure/edge cases:**
  - AI may supply IDs/names that do not exist or lack access.
  - Spreadsheet permissions issues.

#### Node: Google Calendar Tool
- **Type / role:** `googleCalendarTool` — AI-callable Calendar operations.
- **Configuration choices:**
  - Calendar chosen via `$fromAI('Calendar')` (with an email-like regex in config)
  - Requires Google OAuth2 credential with Calendar scopes.
- **Connections:**
  - **ai_tool →** `MCP Server Trigger`
- **Failure/edge cases:**
  - Calendar ID mismatch; timezone/locale ambiguity.
  - AI can attempt destructive actions—system prompt says confirm before important actions, but you may want hard guards.

**Sticky note context:** “MCP Server: Exposes Gmail, Sheets, Calendar. MCP Client: Connects AI to tools”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📌 START HERE | Sticky Note | Documentation / setup instructions |  |  | ## 🤖 Telegram assistant with memory… (Install MemMachine, update org_id/project_id, add credentials, test) |
| Memory Example | Sticky Note | Example / expected behavior |  |  | ## 🎯 Memory Example (Day 1/3/7…) |
| Quick Setup | Sticky Note | Setup commands and config checklist |  |  | ## 🚀 Quick Setup (git clone MemMachine, docker-compose up -d, update nodes 4/5/9, add creds) |
| Customization | Sticky Note | Suggested extensions |  |  | ## 💡 Customization (increase top_k, add tools, edit system prompt node 7…) |
| MCP Architecture | Sticky Note | Architecture explanation |  |  | **MCP Server:** Exposes Gmail, Sheets, Calendar. **MCP Client:** Connects AI to tools |
| Voice Processing | Sticky Note | Notes for voice branch |  |  | **Voice path:** Download → Transcribe (Whisper) → Extract data |
| Text Processing | Sticky Note | Notes for text branch |  |  | **Text path:** Extract message data |
| MemMachine Memory | Sticky Note | Notes for memory block |  |  | **Memory system:** Store message → Search history (30 memories) → Sort chronologically |
| AI Processing | Sticky Note | Notes for AI/tool block |  |  | **AI + Tools:** Process with context, access Gmail/Sheets/Calendar via MCP |
| Response Flow | Sticky Note | Notes for response block |  |  | **Response:** Extract → Store in memory → Send to Telegram |
| 1. Telegram Trigger | Telegram Trigger | Receive Telegram messages | — | 2. Message Type |  |
| 2. Message Type | Switch | Route Voice vs Text | 1. Telegram Trigger | 3a. Download Voice; 3d. Extract Text Data |  |
| 3a. Download Voice | Telegram | Download voice file | 2. Message Type (Voice) | 3b. Transcribe Voice | **Voice path:** Download → Transcribe (Whisper) → Extract data |
| 3b. Transcribe Voice | OpenAI (LangChain) | Whisper transcription | 3a. Download Voice | 3c. Extract Voice Data | **Voice path:** Download → Transcribe (Whisper) → Extract data |
| 3c. Extract Voice Data | Set | Normalize voice transcript + metadata | 3b. Transcribe Voice | 4. Store User Query | **Voice path:** Download → Transcribe (Whisper) → Extract data |
| 3d. Extract Text Data | Set | Normalize text + metadata | 2. Message Type (Text) | 4. Store User Query | **Text path:** Extract message data |
| 4. Store User Query | HTTP Request | Persist user message in MemMachine | 3c or 3d | 5. Search Memory | **Memory system:** Store message → Search history (30 memories) → Sort chronologically |
| 5. Search Memory | HTTP Request | Retrieve relevant memories | 4. Store User Query | 6. Format Memory | **Memory system:** Store message → Search history (30 memories) → Sort chronologically |
| 6. Format Memory | Code | Sort/format memory into conversation_history | 5. Search Memory | 7. AI Agent | **Memory system:** Store message → Search history (30 memories) → Sort chronologically |
| 7. AI Agent | LangChain Agent | Generate response, call tools via MCP | 6. Format Memory (+ OpenAI model + MCP tool) | 8. Extract Response | **AI + Tools:** Process with context, access Gmail/Sheets/Calendar via MCP |
| OpenAI Chat Model | OpenAI Chat Model | LLM backing the agent | — | 7. AI Agent (ai_languageModel) | **AI + Tools:** Process with context, access Gmail/Sheets/Calendar via MCP |
| MCP Client | MCP Client Tool | Provides MCP tools to agent | — | 7. AI Agent (ai_tool) | **AI + Tools:** Process with context, access Gmail/Sheets/Calendar via MCP |
| 8. Extract Response | Set | Map agent output to ai_response + ids | 7. AI Agent | 9. Store AI Response | **Response:** Extract → Store in memory → Send to Telegram |
| 9. Store AI Response | HTTP Request | Persist assistant response in MemMachine | 8. Extract Response | 10. Send Response | **Response:** Extract → Store in memory → Send to Telegram |
| 10. Send Response | Telegram | Send reply to Telegram chat | 9. Store AI Response | — | **Response:** Extract → Store in memory → Send to Telegram |
| MCP Server Trigger | MCP Trigger | MCP server endpoint entrypoint | — | — (tools register to it) | **MCP Server:** Exposes Gmail, Sheets, Calendar. **MCP Client:** Connects AI to tools |
| Gmail Tool | Gmail Tool | MCP-exposed Gmail actions | MCP Server Trigger (registration via ai_tool link) | MCP Server Trigger (ai_tool) | **MCP Server:** Exposes Gmail, Sheets, Calendar. **MCP Client:** Connects AI to tools |
| Google Sheets Tool | Google Sheets Tool | MCP-exposed Sheets actions | MCP Server Trigger (registration via ai_tool link) | MCP Server Trigger (ai_tool) | **MCP Server:** Exposes Gmail, Sheets, Calendar. **MCP Client:** Connects AI to tools |
| Google Calendar Tool | Google Calendar Tool | MCP-exposed Calendar actions | MCP Server Trigger (registration via ai_tool link) | MCP Server Trigger (ai_tool) | **MCP Server:** Exposes Gmail, Sheets, Calendar. **MCP Client:** Connects AI to tools |

---

## 4. Reproducing the Workflow from Scratch

1) **Prerequisites**
1. Deploy **MemMachine** (as per note):
   - `git clone https://github.com/MemMachine/MemMachine`
   - `docker-compose up -d`
2. Decide how n8n will reach MemMachine:
   - If n8n runs in Docker Desktop: `http://host.docker.internal:8080`
   - Otherwise use the correct hostname/IP (e.g., Docker network service name).

2) **Create the Telegram entry flow**
3. Add node **Telegram Trigger** named `1. Telegram Trigger`
   - Updates: `message`
   - Select Telegram Bot credential (create it with your bot token).
4. Add node **Switch** named `2. Message Type`
   - Add rule “Voice”: condition “exists” on expression `{{$json.message.voice.file_id}}`
   - Add rule “Text”: condition “exists” on expression `{{$json.message.text}}`
   - Ensure outputs are renamed “Voice” and “Text”.
5. Connect `1. Telegram Trigger` → `2. Message Type`.

3) **Build the voice branch**
6. Add **Telegram** node named `3a. Download Voice`
   - Resource: `file`
   - File ID: `{{$json.message.voice.file_id}}`
7. Add **OpenAI (LangChain)** node named `3b. Transcribe Voice`
   - Resource: `audio`
   - Operation: `transcribe`
   - Select OpenAI credential.
8. Add **Set** node named `3c. Extract Voice Data`
   - Add fields:
     - `text` (string) = `{{$json.text}}`
     - `chat_id` (number) = `{{ $('1. Telegram Trigger').item.json.message.chat.id }}`
     - `user_id` (number) = `{{ $('1. Telegram Trigger').item.json.message.from.id }}`
     - `username` (string) = `{{ $('1. Telegram Trigger').item.json.message.from.username || $('1. Telegram Trigger').item.json.message.from.first_name }}`
     - `customer_email` (string) = `{{ $('1. Telegram Trigger').item.json.message.from.id }}@telegram.user`
9. Connect `2. Message Type` (Voice output) → `3a. Download Voice` → `3b. Transcribe Voice` → `3c. Extract Voice Data`.

4) **Build the text branch**
10. Add **Set** node named `3d. Extract Text Data`
   - Fields:
     - `text` = `{{$json.message.text}}`
     - `chat_id` = `{{$json.message.chat.id}}`
     - `user_id` = `{{$json.message.from.id}}`
     - `username` = `{{$json.message.from.username || $json.message.from.first_name}}`
     - `customer_email` = `{{$json.message.from.id}}@telegram.user`
11. Connect `2. Message Type` (Text output) → `3d. Extract Text Data`.

5) **MemMachine store + search**
12. Add **HTTP Request** node named `4. Store User Query`
   - Method: POST
   - URL: `http://host.docker.internal:8080/api/v2/memories`
   - Body: JSON with `org_id`, `project_id` and one message (role user) using the normalized fields.
   - Header: `Content-Type: application/json`
   - Replace placeholders with your real `org_id` and `project_id`.
13. Add **HTTP Request** node named `5. Search Memory`
   - Method: POST
   - URL: `http://host.docker.internal:8080/api/v2/memories/search`
   - JSON body:
     - `org_id`, `project_id`
     - `query` from current item `text`
     - `top_k: 30`
     - `filter` on `metadata.customer_email` equal to the same user identifier
     - `types: episodic, semantic`
14. Connect:
   - `3c. Extract Voice Data` → `4. Store User Query`
   - `3d. Extract Text Data` → `4. Store User Query`
   - `4. Store User Query` → `5. Search Memory`

6) **Format memory for prompting**
15. Add **Code** node named `6. Format Memory`
   - Implement logic to:
     - Read MemMachine response
     - Collect episodes, sort by timestamp, format into `conversation_history`
     - Output `text, chat_id, customer_email, username, total_memories, conversation_history`
16. Connect `5. Search Memory` → `6. Format Memory`.

7) **MCP server (tool exposure)**
17. Add **MCP Server Trigger** node named `MCP Server Trigger`
   - Set a path (any unique string).
18. Add tool nodes:
   - **Gmail Tool** (name `Gmail Tool`) and configure Google credential (Gmail scope).
   - **Google Sheets Tool** (name `Google Sheets Tool`) and configure Google credential (Sheets scope).
   - **Google Calendar Tool** (name `Google Calendar Tool`) and configure Google credential (Calendar scope).
   - Keep `$fromAI()`-style parameters so the AI can supply arguments.
19. Connect each Tool node to the MCP Server Trigger using the **ai_tool** connection type (in the UI: “Connect as AI Tool” / tool registration).

8) **AI agent + model + MCP client**
20. Add **OpenAI Chat Model** node named `OpenAI Chat Model`
   - Model: `gpt-4o-mini`
   - Select OpenAI credential.
21. Add **MCP Client Tool** node named `MCP Client`
   - Configure it to point to your MCP server endpoint (the MCP Server Trigger).
22. Add **AI Agent** node named `7. AI Agent`
   - Input text: `{{$json.text}}`
   - System message: include `username`, `customer_email`, `conversation_history`, and tool behavior rules (confirm important actions).
23. Connect:
   - `6. Format Memory` → `7. AI Agent` (main)
   - `OpenAI Chat Model` → `7. AI Agent` (ai_languageModel)
   - `MCP Client` → `7. AI Agent` (ai_tool)

9) **Persist assistant response + send to Telegram**
24. Add **Set** node `8. Extract Response`
   - `ai_response = {{$json.output}}`
   - Copy `chat_id`, `customer_email`, `username` from `6. Format Memory`
25. Add **HTTP Request** node `9. Store AI Response`
   - POST to `/api/v2/memories`
   - Store assistant message (role assistant) with metadata and correct `org_id/project_id`.
26. Add **Telegram** node `10. Send Response`
   - Chat ID: `{{$json.chat_id}}`
   - Text: `{{$json.ai_response}}`
   - Parse mode: Markdown
27. Connect: `7. AI Agent` → `8. Extract Response` → `9. Store AI Response` → `10. Send Response`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: David Olusola (@DaexAI). Assistant remembers conversations across sessions using MemMachine; supports voice transcription; Gmail/Sheets/Calendar tools; MCP architecture. | From sticky note “📌 START HERE” |
| Install MemMachine: `git clone https://github.com/MemMachine/MemMachine` then `docker-compose up -d` | From sticky note “Quick Setup” / “START HERE” |
| Update `org_id` and `project_id` in nodes **4**, **5**, **9** | From sticky notes “START HERE” and “Quick Setup” |
| Voice behavior example: assistant can recall older requests and act on follow-ups | From sticky note “Memory Example” |
| Customization ideas: increase `top_k` (node 5), add Notion/Slack/Trello tools, multi-channel expansion, edit system prompt (node 7) | From sticky note “Customization” |

