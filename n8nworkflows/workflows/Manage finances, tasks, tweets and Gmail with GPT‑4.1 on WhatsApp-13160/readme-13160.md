Manage finances, tasks, tweets and Gmail with GPT‑4.1 on WhatsApp

https://n8nworkflows.xyz/workflows/manage-finances--tasks--tweets-and-gmail-with-gpt-4-1-on-whatsapp-13160


# Manage finances, tasks, tweets and Gmail with GPT‑4.1 on WhatsApp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Manage finances, tasks, tweets and Gmail with GPT‑4.1 on WhatsApp

This workflow turns WhatsApp (via Green-API) messages into actions handled by a central GPT-4.1-mini “router” agent. It supports:
- Expense tracking in **Google Sheets**
- Task management in **Google Tasks**
- Posting to **X (Twitter)**
- Searching/reading **Gmail** (read-only)
- **Web search** (SerpAPI) for up-to-date info
- **Calculator** for pure math
- Voice message handling (download → **Whisper transcription**) and optional voice replies (**OpenAI TTS**)

### Logical Blocks
**1.1 Input Reception & Classification (WhatsApp / Chat trigger)**
- Accepts inbound webhook from Green-API (WhatsApp) and classifies message type (audio vs text).

**1.2 Voice Handling (Download → Transcribe → Normalize)**
- For audio messages: download audio file, transcribe with OpenAI, mark that the reply should be audio.

**1.3 Main Routing Agent (Intent Router)**
- Single “main-agent” decides whether to call one sub-agent/tool (expenses/tasks/tweets/gmail/web/calc) or respond directly.

**1.4 Sub-Agents & Tools (Action Executors)**
- Expense agent → Google Sheets (append/read)
- Tasks agent → Google Tasks API (HTTP tools)
- Tweet agent → X post
- Gmail agent → Gmail search/list
- Web search tool → SerpAPI
- Calculator tool → arithmetic

**1.5 Response Delivery (Text vs Voice)**
- If original input was audio → generate TTS audio and send file to WhatsApp
- Otherwise send text to WhatsApp

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Classification
**Overview:** Receives messages from WhatsApp via Green-API webhook and routes by message type (audio/text; image/doc rules exist but are not fully wired).

**Nodes involved:**
- **Webhook**
- **Switch**
- **When chat message received** (manual testing entry)

#### Node: Webhook
- **Type / Role:** Webhook Trigger (n8n) to receive Green-API inbound events.
- **Key config:** HTTP **POST**, path set to a unique UUID-like string.
- **Input/Output:** Entry node → outputs to **Switch**.
- **Edge cases:**
  - Green-API signature/format mismatches can break `$json.body...` expressions downstream.
  - Public webhook requires correct URL configuration in Green-API.
- **Sticky note:**  
  - “## Classify input”  
  - Global note (How it works / setup) applies broadly (see section 3).

#### Node: Switch
- **Type / Role:** Switch node to classify WhatsApp message type.
- **Key config:** Rules check `{{$json.body.messageData.typeMessage}}` equals:
  - `audioMessage` → output “audio”
  - `textMessage` → output “text”
  - `imageMessage` → output “image” (defined but not connected)
  - `documentMessage` → output “doc” (defined but not connected)
- **Connections:**
  - audio → **download-audio**
  - text → **audio to 0**
- **Edge cases:**
  - If `typeMessage` has unexpected value, nothing routes.
  - Image/doc paths exist but have no outgoing connections in this workflow.
- **Sticky note:** “## Classify input”

#### Node: When chat message received
- **Type / Role:** LangChain Chat Trigger for manual execution/testing in n8n UI (not WhatsApp).
- **Key config:** No special options.
- **Connections:** Goes directly to **main-agent**.
- **Edge cases:**
  - Chat trigger payload differs from WhatsApp webhook payload; this workflow expects `$json.text` in main-agent for both paths (WhatsApp path normalizes via Set nodes).
- **Sticky note:** “## Chat Trigger for manual execution”

---

### 2.2 Voice Handling (Download → Transcribe → Normalize)
**Overview:** For audio WhatsApp messages, downloads the audio from Green-API and transcribes it using OpenAI, then normalizes to `{ text, audio }`.

**Nodes involved:**
- **download-audio**
- **Transcribe a recording**
- **set audio to one**
- **audio to 0** (text normalization; in parallel block)

#### Node: download-audio
- **Type / Role:** HTTP Request to fetch the audio binary.
- **Key config:** URL: `{{$json.body.messageData.fileMessageData.downloadUrl}}`
- **Connections:** → **Transcribe a recording**
- **Edge cases:**
  - Download URL may expire.
  - Response may not be binary unless HTTP Request is configured to handle binary (depends on Green-API behavior; if it returns a file stream, ensure n8n stores it as binary).
  - If messageData path differs, expression fails.

#### Node: Transcribe a recording
- **Type / Role:** OpenAI node (LangChain OpenAI resource) to transcribe audio (Whisper).
- **Key config:** `resource: audio`, `operation: transcribe`.
- **Connections:** → **set audio to one**
- **Credentials:** OpenAI API credential “rizwan”.
- **Edge cases:**
  - Audio must be provided as binary in the expected input field; misconfigured binary property causes empty transcription.
  - OpenAI quota/403, model availability, or timeouts on long audio.

#### Node: set audio to one
- **Type / Role:** Set node to normalize output after transcription.
- **Key config:** Creates:
  - `text = {{$json.text}}` (expects transcription node output in `text`)
  - `audio = "1"` (string flag meaning “reply with audio”)
- **Connections:** → **main-agent**
- **Edge cases:** If transcription output field is not `text`, `text` becomes empty.

#### Node: audio to 0
- **Type / Role:** Set node to normalize plain text messages.
- **Key config:** Creates:
  - `text = {{$json.body.messageData.textMessageData.textMessage}}`
  - `audio = "0"` (string flag meaning “reply with text”)
- **Connections:** → **main-agent**
- **Edge cases:** Expression depends on Green-API payload shape; any schema change breaks.

**Sticky note:** “## Classify input” (covers classification/normalization concept)

---

### 2.3 Main Routing Agent (Intent Router)
**Overview:** A single GPT agent (“n8n Wizard”) decides which tool/sub-agent to call based on intent, or answers directly. It also includes strict safety/routing rules and uses short-term memory.

**Nodes involved:**
- **main-agent**
- **OpenAI Chat Model1**
- **Simple Memory**
- **quick-web-search**
- **Calculator**

#### Node: main-agent
- **Type / Role:** LangChain Agent (router/orchestrator).
- **Key config:**
  - Input text: `{{$json.text}}`
  - System message: strict routing rules:
    - Use only: expense-tracker, Task-manager-agent, Tweet-agent, get-gmail, Quick-google-search, Calculator
    - Never invent data; ask one clarifying question if ambiguous
    - Quick-google-search must be a full `https://` URL
  - Dynamically inserts current date: `{{$now.toFormat('yyyy-MM-dd')}}`
- **Connections:**
  - Main output → **If** (to decide audio vs text response)
  - AI language model input from **OpenAI Chat Model1**
  - AI memory input from **Simple Memory**
  - Tools available via AI tool connections: expense-tracker, Task-manager-agent, Tweet-agent, get-gmail, quick-web-search, Calculator
- **Edge cases:**
  - If `text` is missing (bad normalization), agent may produce irrelevant output or ask clarifying question.
  - If multiple intents in one message, system message says step-by-step tool calls; ensure your agent node is allowed to do multiple tool calls in one run (depends on agent settings and n8n version).

#### Node: OpenAI Chat Model1
- **Type / Role:** LLM provider for main-agent.
- **Key config:** Model `gpt-4.1-mini`.
- **Credentials:** OpenAI “moosa-n8n”.
- **Edge cases:** Model naming/availability; API quota.

#### Node: Simple Memory
- **Type / Role:** Buffer window memory for short context retention.
- **Key config:** Default memory buffer window (no explicit params shown).
- **Connections:** Memory → main-agent.
- **Edge cases:** Memory may cause unintended context carryover; reduce window or clear if privacy concerns.

#### Node: quick-web-search
- **Type / Role:** SerpAPI tool node used by main-agent for real-time/external info.
- **Key config:** Default SerpAPI options.
- **Credentials:** “SerpAPI ali”
- **Connections:** Tool → main-agent
- **Edge cases:** SerpAPI quota, blocked queries, network timeouts.

#### Node: Calculator
- **Type / Role:** Calculator tool node used by main-agent for pure math.
- **Key config:** Default.
- **Connections:** Tool → main-agent
- **Edge cases:** None major; ensure inputs are numeric strings.

**Sticky note:** “## main Agent” (applies to main-agent)

---

### 2.4 Sub-Agent: Expense Tracking (Google Sheets)
**Overview:** A specialized finance agent that only reads/appends rows in a Google Sheet and returns summaries.

**Nodes involved:**
- **expense-tracker**
- **OpenAI Chat Model4**
- **Append row in sheet in Google Sheets**
- **Get row(s) in sheet in Google Sheets**

#### Node: expense-tracker
- **Type / Role:** LangChain Agent Tool (sub-agent invoked by main-agent).
- **Key config:**
  - Receives text instruction via `$fromAI('Prompt__User_Message_')`
  - System message enforces:
    - Only expense tracking
    - Append on spending reports (extract type/amount/category/description)
    - Read for summaries using filters
    - No delete/update, only read/append
    - Amounts in PKR, concise output, tabular for multi-rows
- **Connections:**
  - Uses **OpenAI Chat Model4**
  - Tool access to Google Sheets nodes (append/get)
  - Tool output returns to main-agent
- **Edge cases:**
  - If user doesn’t specify amount/category/type clearly, sub-agent should ask one question (per its rules).
  - Currency assumption is PKR; if user provides other currency, ambiguity arises.

#### Node: OpenAI Chat Model4
- **Type / Role:** LLM for expense-tracker.
- **Key config:** `gpt-4.1-mini`
- **Credentials:** OpenAI “rizwan”
- **Edge cases:** quota/model availability.

#### Node: Append row in sheet in Google Sheets
- **Type / Role:** Google Sheets Tool node to append a new expense row.
- **Key config:**
  - Operation: Append to sheet “EXPENSES” in document “Expense Tracker”
  - Mapped columns:
    - `date = {{$now.toFormat('d-M-yyyy HH:mm')}}`
    - `type, amount, category, description` from `$fromAI(...)`
- **Credentials:** Google Sheets OAuth2 “moosa-abc1”
- **Connections:** Tool → expense-tracker
- **Edge cases:**
  - Spreadsheet permissions / OAuth scope issues.
  - Column names must match sheet headers exactly.
  - If amount is non-numeric text, you’ll store it as string (conversion disabled).

#### Node: Get row(s) in sheet in Google Sheets
- **Type / Role:** Google Sheets Tool node to retrieve expenses with filters.
- **Key config:**
  - Filters (combined with **OR**):
    - `date` equals `$fromAI('values0_Value')`
    - `category` equals `$fromAI('values1_Value')`
- **Credentials:** Google Sheets OAuth2 “moosa-abc1”
- **Connections:** Tool → expense-tracker
- **Edge cases:**
  - Using OR can return broad results; for “date range” queries, this config may be insufficient unless the agent encodes ranges into supported filters (it currently uses equals).
  - Date stored as a formatted string; querying by date equality can be brittle.

**Sticky note:** “## Sub Agent for expense Tracking”

---

### 2.5 Sub-Agent: Google Tasks (via HTTP Tools)
**Overview:** A strict execution-only agent that performs exactly one Google Tasks API operation per instruction using provided URLs/JSON.

**Nodes involved:**
- **Task-manager-agent**
- **OpenAI Chat Model**
- **Create a task in Google Tasks**
- **create list**
- **update task**
- **get list**
- **get tasks**

#### Node: Task-manager-agent
- **Type / Role:** LangChain Agent Tool (sub-agent).
- **Key config:** System message rules:
  - Does not interpret intent; trusts IDs and values
  - Never verifies IDs by listing
  - Calls only one tool per request
  - Chooses tool based on presence of keys: `listId`, `createTask`, `createList`, `updateTask`, `deleteTask`
- **Connections:** Uses **OpenAI Chat Model**; has tool access to the HTTP request tool nodes.
- **Edge cases:**
  - If main-agent gives an instruction without the expected keys, the tasks agent may do nothing or respond incorrectly.
  - “delete task” logic is mentioned, but there is **no delete-task node** in this workflow (gap).

#### Node: OpenAI Chat Model
- **Type / Role:** LLM for Task-manager-agent.
- **Key config:** `gpt-4.1-mini`
- **Credentials:** OpenAI “moosa-n8n”
- **Edge cases:** quota.

#### Node: Create a task in Google Tasks
- **Type / Role:** HTTP Request Tool (POST) to create a task.
- **Key config:**
  - URL from `$fromAI('URL')` (expects full Google Tasks endpoint)
  - JSON body from `$fromAI('JSON')`
  - Auth: predefined credential type `googleTasksOAuth2Api`
  - Sets `Content-Type: application/json`
- **Credentials:** Google Tasks OAuth2 “moosa-abc1”
- **Connections:** Tool → Task-manager-agent
- **Edge cases:** If URL is malformed or missing list ID, Google returns 400/404.

#### Node: create list
- **Type / Role:** HTTP Request Tool (POST) to create a task list.
- **Key config:** Fixed URL `https://tasks.googleapis.com/tasks/v1/users/@me/lists`, JSON body from `$fromAI('JSON')`
- **Credentials:** Google Tasks OAuth2 “moosa-abc1”
- **Connections:** Tool → Task-manager-agent
- **Edge cases:** Missing required fields (e.g., title) yields 400.

#### Node: update task
- **Type / Role:** HTTP Request Tool (PATCH) to update a task.
- **Key config:** URL from `$fromAI('URL')`, JSON from `$fromAI('JSON')`
- **Credentials:** Google Tasks OAuth2 “moosa-abc1”
- **Connections:** Tool → Task-manager-agent
- **Edge cases:** PATCH requires correct task ID URL; partial update semantics.

#### Node: get list
- **Type / Role:** HTTP Request Tool (GET) listing all task lists.
- **Key config:** Fixed URL `https://tasks.googleapis.com/tasks/v1/users/@me/lists`
- **Credentials:** Google Tasks OAuth2 “moosa-abc1”
- **Connections:** Tool → Task-manager-agent
- **Edge cases:** Large list pagination not handled explicitly.

#### Node: get tasks
- **Type / Role:** HTTP Request Tool (GET) for tasks.
- **Key config:** URL from `$fromAI('URL')`; body from `$fromAI('JSON')` (unusual for GET, but enabled).
- **Credentials:** Google Tasks OAuth2 “moosa-abc1”
- **Connections:** Tool → Task-manager-agent
- **Edge cases:** Many APIs ignore GET bodies; ensure query parameters are in URL instead.

**Sticky note:** “## Sub Agent for Google Tasks”

---

### 2.6 Sub-Agent: Tweet Posting (X)
**Overview:** A tweet agent that posts exactly once using X credentials, or suggests alternative if >280 chars.

**Nodes involved:**
- **Tweet-agent**
- **OpenAI Chat Model3**
- **Create Tweet in X**

#### Node: Tweet-agent
- **Type / Role:** LangChain Agent Tool.
- **Key config:** System message:
  - Use post tweet tool to post
  - If >280 characters, suggest alternative
  - Post only once
- **Connections:** Uses **OpenAI Chat Model3** and tool **Create Tweet in X**
- **Edge cases:** If the agent is asked to draft only (not post), prompt says “If you get instructions to post… then post”; main-agent must be clear.

#### Node: OpenAI Chat Model3
- **Type / Role:** LLM for Tweet-agent.
- **Key config:** `gpt-4.1-mini`
- **Credentials:** OpenAI “rizwan”

#### Node: Create Tweet in X
- **Type / Role:** Twitter/X tool node that posts a tweet.
- **Key config:** Text from `$fromAI('Text')`
- **Credentials:** Twitter OAuth2 “X account 2”
- **Edge cases:** Rate limits, duplicate tweet restrictions, auth scopes, policy restrictions.

**Sticky note:** “## Sub Agent for Posting Tweet”

---

### 2.7 Sub-Agent: Gmail Retrieval (Read-only)
**Overview:** A Gmail agent that searches emails using Gmail “getAll” and returns concise structured summaries.

**Nodes involved:**
- **get-gmail**
- **OpenAI Chat Model2**
- **Get many messages in Gmail**

#### Node: get-gmail
- **Type / Role:** LangChain Agent Tool.
- **Key config:** System message:
  - Use attached Gmail tool when tasked
  - Keep answers clean/short/structured; bullet/numbered lists for multiple emails
- **Connections:** Uses **OpenAI Chat Model2** and tool **Get many messages in Gmail**
- **Edge cases:** If user requests sending/replying, main-agent system message forbids it; should refuse or explain briefly.

#### Node: OpenAI Chat Model2
- **Type / Role:** LLM for get-gmail.
- **Key config:** `gpt-4.1-mini`
- **Credentials:** OpenAI “moosa-n8n”

#### Node: Get many messages in Gmail
- **Type / Role:** Gmail tool node to retrieve many messages.
- **Key config:**
  - Operation: `getAll`
  - Filters populated via `$fromAI(...)`:
    - `q`, `sender`, `labelIds`, `receivedAfter`, `receivedBefore`
  - `returnAll` from `$fromAI('Return_All')` boolean
- **Credentials:** Gmail OAuth2 “abc1”
- **Edge cases:** Gmail query syntax errors, insufficient read scopes, large mailbox pagination if `returnAll=false`.

**Sticky note:** “## Sub Agent for Gmail”

---

### 2.8 Response Delivery (Text vs Voice)
**Overview:** Uses a flag (`audio`) to decide whether to return a text message or generate TTS audio and upload it back to WhatsApp.

**Nodes involved:**
- **If**
- **Generate audio**
- **send audio response**
- **send text response**

#### Node: If
- **Type / Role:** Conditional branch to choose response mode.
- **Key config:** Condition checks whether `$('set audio to one').item.json.audio` **exists**.
  - If true → treat as voice-input path → generate audio
  - Else → send text response
- **Connections:** True → **Generate audio**; False → **send text response**
- **Edge cases / logic warning:**
  - This condition depends on the *presence* of the “set audio to one” node in the execution. If execution started from text path, that node may not exist, so it routes to text.
  - More robust approach: check `$json.audio == "1"` rather than referencing another node.

#### Node: Generate audio
- **Type / Role:** OpenAI TTS generation.
- **Key config:**
  - Input: `{{$json.output}}` (expects main-agent output field named `output`)
  - Voice: `shimmer`
- **Credentials:** OpenAI “rizwan”
- **Connections:** → **send audio response**
- **Edge cases:** If main-agent output isn’t in `output`, audio generation will be blank.

#### Node: send text response
- **Type / Role:** HTTP Request to Green-API sendMessage endpoint.
- **Key config:**
  - URL is hardcoded to Green-API endpoint with placeholder `API_KEY_HERE`
  - Body:
    - `chatId = {{$item("0").$node["Webhook"].json["body"]["senderData"]["chatId"]}}`
    - `message = {{$json.output}}`
- **Edge cases:**
  - If execution started from **When chat message received**, there is no “Webhook” node; this expression will fail unless you adapt it.
  - Requires valid Green-API instance and API key.

#### Node: send audio response
- **Type / Role:** HTTP Request to Green-API sendFileByUpload endpoint (multipart).
- **Key config:**
  - URL hardcoded with `API_KEY_HERE`
  - Multipart fields:
    - `file` from binary input field `data`
    - `chatId` from Webhook senderData chatId (same dependency issue as above)
- **Edge cases:**
  - Ensure OpenAI TTS node outputs binary under expected field name (commonly `data`).
  - Same “Webhook not present in manual trigger run” issue.

**Sticky note:** “## send response according to input”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receive WhatsApp event from Green-API | — | Switch | ## Classify input |
| Switch | n8n-nodes-base.switch | Classify WhatsApp message type | Webhook | download-audio; audio to 0 | ## Classify input |
| download-audio | n8n-nodes-base.httpRequest | Download voice message audio from Green-API URL | Switch | Transcribe a recording | ## Classify input |
| Transcribe a recording | @n8n/n8n-nodes-langchain.openAi | Whisper transcription of audio | download-audio | set audio to one | ## Classify input |
| set audio to one | n8n-nodes-base.set | Normalize transcription and mark reply-as-audio | Transcribe a recording | main-agent | ## Classify input |
| audio to 0 | n8n-nodes-base.set | Normalize text message and mark reply-as-text | Switch | main-agent | ## Classify input |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Manual test entry point | — | main-agent | ## Chat Trigger for manual execution |
| main-agent | @n8n/n8n-nodes-langchain.agent | Central intent router and orchestrator | audio to 0; set audio to one; When chat message received | If | ## main Agent |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for main-agent | — | main-agent (ai_languageModel) | ## main Agent |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation memory for main-agent | — | main-agent (ai_memory) | ## main Agent |
| quick-web-search | @n8n/n8n-nodes-langchain.toolSerpApi | Web search tool for real-time info | — | main-agent (ai_tool) | ## Tools to do websearch and calculate |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Calculator tool for math | — | main-agent (ai_tool) | ## Tools to do websearch and calculate |
| expense-tracker | @n8n/n8n-nodes-langchain.agentTool | Sub-agent for expense tracking | — | main-agent (ai_tool) | ## Sub Agent for expense Tracking |
| OpenAI Chat Model4 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for expense-tracker | — | expense-tracker (ai_languageModel) | ## Sub Agent for expense Tracking |
| Append row in sheet in Google Sheets | n8n-nodes-base.googleSheetsTool | Append expense row | — | expense-tracker (ai_tool) | ## Sub Agent for expense Tracking |
| Get row(s) in sheet in Google Sheets | n8n-nodes-base.googleSheetsTool | Read/filter expense rows | — | expense-tracker (ai_tool) | ## Sub Agent for expense Tracking |
| Task-manager-agent | @n8n/n8n-nodes-langchain.agentTool | Sub-agent for Google Tasks execution | — | main-agent (ai_tool) | ## Sub Agent for Google Tasks |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Task-manager-agent | — | Task-manager-agent (ai_languageModel) | ## Sub Agent for Google Tasks |
| Create a task in Google Tasks | n8n-nodes-base.httpRequestTool | Google Tasks API: create task | — | Task-manager-agent (ai_tool) | ## Sub Agent for Google Tasks |
| create list | n8n-nodes-base.httpRequestTool | Google Tasks API: create list | — | Task-manager-agent (ai_tool) | ## Sub Agent for Google Tasks |
| update task | n8n-nodes-base.httpRequestTool | Google Tasks API: update task | — | Task-manager-agent (ai_tool) | ## Sub Agent for Google Tasks |
| get list | n8n-nodes-base.httpRequestTool | Google Tasks API: list task lists | — | Task-manager-agent (ai_tool) | ## Sub Agent for Google Tasks |
| get tasks | n8n-nodes-base.httpRequestTool | Google Tasks API: get tasks | — | Task-manager-agent (ai_tool) | ## Sub Agent for Google Tasks |
| Tweet-agent | @n8n/n8n-nodes-langchain.agentTool | Sub-agent for posting tweets | — | main-agent (ai_tool) | ## Sub Agent for Posting Tweet |
| OpenAI Chat Model3 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Tweet-agent | — | Tweet-agent (ai_languageModel) | ## Sub Agent for Posting Tweet |
| Create Tweet in X | n8n-nodes-base.twitterTool | Post tweet to X | — | Tweet-agent (ai_tool) | ## Sub Agent for Posting Tweet |
| get-gmail | @n8n/n8n-nodes-langchain.agentTool | Sub-agent for Gmail retrieval | — | main-agent (ai_tool) | ## Sub Agent for Gmail |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for get-gmail | — | get-gmail (ai_languageModel) | ## Sub Agent for Gmail |
| Get many messages in Gmail | n8n-nodes-base.gmailTool | Gmail getAll messages with filters | — | get-gmail (ai_tool) | ## Sub Agent for Gmail |
| If | n8n-nodes-base.if | Choose text vs voice reply | main-agent | Generate audio; send text response | ## send response according to input |
| Generate audio | @n8n/n8n-nodes-langchain.openAi | OpenAI TTS generation | If | send audio response | ## send response according to input |
| send text response | n8n-nodes-base.httpRequest | Green-API sendMessage | If | — | ## send response according to input |
| send audio response | n8n-nodes-base.httpRequest | Green-API sendFileByUpload | Generate audio | — | ## send response according to input |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace note (how it works / setup) | — | — | (See Notes in section 5) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Classify input |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment header | — | — | ## send response according to input |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment header | — | — | ## main Agent |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Sub Agent for expense Tracking |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Sub Agent for Google Tasks |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Sub Agent for Posting Tweet |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Sub Agent for Gmail |
| Sticky Note8 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Tools to do websearch and calculate |
| Sticky Note9 | n8n-nodes-base.stickyNote | Comment header | — | — | ## Chat Trigger for manual execution |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials (before building)**
   1. OpenAI API key (for Chat + Whisper + TTS).
   2. Google Sheets OAuth2 (access to the target spreadsheet).
   3. Google Tasks OAuth2 (tasks scope).
   4. Gmail OAuth2 (read-only scopes recommended, e.g. `gmail.readonly`).
   5. Twitter/X OAuth2 (permission to post tweets).
   6. SerpAPI credential (API key).
   7. Green-API: you will not store as n8n credential here (URLs are hardcoded), but you must have:
      - WhatsApp instance ID
      - Green-API key
      - Webhook configured to point to your n8n webhook URL.

2) **Create the WhatsApp inbound trigger**
   1. Add **Webhook** node:
      - Method: POST
      - Path: generate a unique path (n8n will provide test/prod URLs)
   2. In Green-API dashboard, set the webhook URL to the n8n production webhook URL for this node.

3) **Add message type classification**
   1. Add **Switch** node:
      - Rule 1: if `{{$json.body.messageData.typeMessage}} == "audioMessage"` → output named `audio`
      - Rule 2: if `... == "textMessage"` → output named `text`
      - (Optional) define image/doc outputs if you plan to implement them
   2. Connect **Webhook → Switch**.

4) **Build the audio path**
   1. Add **HTTP Request** node named `download-audio`:
      - URL: `{{$json.body.messageData.fileMessageData.downloadUrl}}`
      - Configure it to download/store binary if needed (depends on Green-API response).
   2. Add **OpenAI** node named `Transcribe a recording`:
      - Resource: Audio
      - Operation: Transcribe
      - Use your OpenAI credential
      - Ensure it reads the binary audio from the previous node.
   3. Add **Set** node named `set audio to one`:
      - Set `text = {{$json.text}}` (or map from the transcription field you receive)
      - Set `audio = "1"`
   4. Connect: **Switch(audio) → download-audio → Transcribe a recording → set audio to one**

5) **Build the text path**
   1. Add **Set** node named `audio to 0`:
      - Set `text = {{$json.body.messageData.textMessageData.textMessage}}`
      - Set `audio = "0"`
   2. Connect: **Switch(text) → audio to 0**

6) **Add a manual test entry (optional but present in workflow)**
   1. Add **When chat message received** (LangChain Chat Trigger).
   2. Connect it directly to **main-agent** (step 7).  
   3. Note: For full testing of WhatsApp reply sending nodes, you must also adapt chatId handling, since chat trigger won’t provide Webhook senderData.

7) **Create the main router agent**
   1. Add **OpenAI Chat Model** node named `OpenAI Chat Model1`:
      - Model: `gpt-4.1-mini`
   2. Add **Simple Memory** node.
   3. Add **Agent** node named `main-agent`:
      - Prompt type: “define”
      - Text input: `{{$json.text}}`
      - System message: include strict routing rules (expenses/tasks/tweets/gmail/web/calc), date injection, and safety constraints.
   4. Wire:
      - **OpenAI Chat Model1 → main-agent** via *ai_languageModel*
      - **Simple Memory → main-agent** via *ai_memory*
      - **audio to 0 → main-agent** (main connection)
      - **set audio to one → main-agent** (main connection)
      - **When chat message received → main-agent** (main connection)

8) **Add tools for web search and calculator**
   1. Add **SerpAPI tool** node (`quick-web-search`).
   2. Add **Calculator tool** node (`Calculator`).
   3. Connect each to **main-agent** via *ai_tool*.

9) **Create the Expense sub-agent**
   1. Add **OpenAI Chat Model** node `OpenAI Chat Model4` (gpt-4.1-mini).
   2. Add **Agent Tool** node `expense-tracker`:
      - Provide system message restricting it to expense read/append only and PKR formatting.
   3. Add **Google Sheets Tool** node `Append row...`:
      - Operation: Append
      - Document: your “Expense Tracker” spreadsheet
      - Sheet: “EXPENSES”
      - Map columns:
        - date: `{{$now.toFormat('d-M-yyyy HH:mm')}}`
        - type/amount/category/description from `$fromAI(...)` fields.
   4. Add **Google Sheets Tool** node `Get row(s)...`:
      - Operation: Get (rows)
      - Same document/sheet
      - Add filters; (recommended improvement: support date ranges properly rather than equality).
   5. Connect:
      - `OpenAI Chat Model4 → expense-tracker` (ai_languageModel)
      - Sheets nodes → expense-tracker (ai_tool)
      - `expense-tracker → main-agent` (ai_tool)

10) **Create the Google Tasks sub-agent**
   1. Add **OpenAI Chat Model** node `OpenAI Chat Model` (gpt-4.1-mini).
   2. Add **Agent Tool** node `Task-manager-agent` with the strict “execution-only” system message.
   3. Add HTTP Request Tool nodes (all authenticated with **Google Tasks OAuth2**):
      - `get list` (GET fixed URL)
      - `create list` (POST fixed URL, JSON from `$fromAI('JSON')`)
      - `Create a task in Google Tasks` (POST URL from `$fromAI('URL')`, JSON from `$fromAI('JSON')`)
      - `update task` (PATCH URL from `$fromAI('URL')`, JSON from `$fromAI('JSON')`)
      - `get tasks` (GET URL from `$fromAI('URL')`; avoid JSON body for GET if possible)
   4. Connect:
      - LLM → Task-manager-agent (ai_languageModel)
      - Each HTTP tool → Task-manager-agent (ai_tool)
      - Task-manager-agent → main-agent (ai_tool)
   5. (Recommended) Add a **delete task** HTTP tool if you want to match the agent’s stated capability.

11) **Create the Tweet sub-agent**
   1. Add **OpenAI Chat Model** node `OpenAI Chat Model3` (gpt-4.1-mini).
   2. Add **Agent Tool** node `Tweet-agent` with its posting rules.
   3. Add **Twitter/X tool** node `Create Tweet in X`:
      - Operation: create tweet
      - Text from `$fromAI('Text')`
   4. Connect:
      - LLM → Tweet-agent (ai_languageModel)
      - Create Tweet in X → Tweet-agent (ai_tool)
      - Tweet-agent → main-agent (ai_tool)

12) **Create the Gmail sub-agent**
   1. Add **OpenAI Chat Model** node `OpenAI Chat Model2` (gpt-4.1-mini).
   2. Add **Agent Tool** node `get-gmail` with “clean, short, structured” rules.
   3. Add **Gmail Tool** node `Get many messages in Gmail`:
      - Operation: getAll
      - Map filters (`q`, `sender`, `labelIds`, `receivedAfter`, `receivedBefore`) from `$fromAI(...)`
      - `returnAll` from `$fromAI('Return_All')`
   4. Connect:
      - LLM → get-gmail (ai_languageModel)
      - Gmail tool → get-gmail (ai_tool)
      - get-gmail → main-agent (ai_tool)

13) **Add response selection and sending**
   1. Add **If** node:
      - Condition currently checks existence of `$('set audio to one').item.json.audio`
      - (Recommended) Replace with: `{{$json.audio}} == "1"`
   2. Add **OpenAI** node `Generate audio` (TTS):
      - Resource: Audio
      - Input: `{{$json.output}}`
      - Voice: `shimmer`
   3. Add **HTTP Request** node `send text response`:
      - URL: Green-API sendMessage endpoint for your instance
      - Body: chatId from webhook payload; message from `{{$json.output}}`
   4. Add **HTTP Request** node `send audio response`:
      - URL: Green-API sendFileByUpload endpoint
      - multipart: binary file + chatId
   5. Connect:
      - main-agent → If
      - If(true) → Generate audio → send audio response
      - If(false) → send text response

14) **Test**
   - WhatsApp text: “I spent 2500 on lunch”
   - WhatsApp voice: speak a task; confirm you receive audio reply
   - Web search: “Weather in Karachi now”
   - Gmail: “Any emails from X since Friday?”

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How it works**: 1) Green-API webhook receives text/voice 2) Voice → download → Whisper 3) Main router selects one tool 4) Sub-agents act (Sheets/Tasks/X/Gmail/Web/Calc) 5) Reply as text or voice (TTS if voice input) | Sticky note “How it works” (workspace note) |
| **Setup steps (credentials):** OpenAI (chat+whisper+TTS), Green-API, Google Sheets OAuth, Google Tasks OAuth, Twitter/X OAuth2, Gmail OAuth (read scope), SerpAPI | Sticky note “How it works” (workspace note) |
| **Important:** Update hardcoded **chatId** handling in the two send-response nodes (they currently reference the Webhook payload). | Sticky note “How it works” (workspace note) |
| Suggested test commands: “I spent 2500 on lunch”, “Weather in Karachi now”, “Create task: Call dentist tomorrow” | Sticky note “How it works” (workspace note) |
| Comment headers present: “Classify input”, “main Agent”, “send response according to input”, and sub-agent section labels | Sticky Notes 1–9 |

