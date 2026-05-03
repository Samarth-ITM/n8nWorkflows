Design UI projects with Google Stitch via Telegram using MCP and Gemini

https://n8nworkflows.xyz/workflows/design-ui-projects-with-google-stitch-via-telegram-using-mcp-and-gemini-13269


# Design UI projects with Google Stitch via Telegram using MCP and Gemini

## 1. Workflow Overview

**Workflow name:** MCP Google Stitch Agent via Telegram  
**Purpose:** A conversational UI/design agent that listens to Telegram messages, authorizes the sender, requires the `/stitch` command, then uses **Google Gemini** + **Google Stitch (MCP endpoint)** to manage Stitch projects and generate UI screens. It can optionally use **Perplexity** for web context. The final response is formatted into **Telegram-compatible HTML** and sent back to the user.

**Typical use cases**
- Create or retrieve Google Stitch projects and screens
- Generate UI screens from text prompts (“design a checkout flow…”, “make a mobile settings page…”)
- Query the web for extra context (patterns, requirements) when needed

### Logical blocks
**1.1 Telegram Intake + Authorization + Command Gate**  
Telegram trigger → verify authorized Telegram user ID → only proceed if message starts with `/stitch`.

**1.2 Input Normalization**  
Extract message text and session ID → remove `/stitch ` prefix so the agent receives a clean prompt.

**1.3 AI Agent Orchestration (Gemini + Memory + MCP Tools + Web Search)**  
Gemini powers the agent; memory keeps short conversation context; agent selectively calls Stitch MCP tools and optionally Perplexity.

**1.4 Response Formatting + Delivery**  
Agent outputs Markdown → convert to Telegram-safe HTML → send Telegram message.

**1.5 Optional/Secondary Entry Point (n8n Chat Trigger)**  
A separate “When chat message received” entry feeds directly into the same agent (useful for n8n’s internal chat), but downstream logic is optimized for Telegram delivery.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Intake + Authorization + Command Gate

**Overview:** Receives Telegram messages, blocks unauthorized users, and only continues when the message starts with `/stitch`.

**Nodes involved:**  
- **Get Message** (telegramTrigger)  
- **Code** (code)  
- **Search with MCP?** (if)  
- **Sticky Note8** (documentation)  
- **Sticky Note1** (documentation, global overview)

#### Node: Get Message
- **Type / role:** `telegramTrigger` — webhook trigger for Telegram bot updates.
- **Config (interpreted):**
  - Listens to **updates: message**
  - Uses Telegram bot credentials “Telegram account Fastewb”.
- **Inputs/Outputs:**
  - No input (trigger).
  - Output goes to **Code**.
- **Failure/edge cases:**
  - Telegram webhook not registered until workflow is active.
  - Bot token invalid/revoked; permission issues; network timeouts.
  - Messages other than standard “message” update won’t trigger.

#### Node: Code (authorization)
- **Type / role:** `code` — hard authorization gate.
- **Config (interpreted):**
  - Checks: `if ($input.first().json.message.from.id !== xxx)` where `xxx` must be replaced with your Telegram user ID.
  - If unauthorized: returns `{ unauthorized: true }`
  - If authorized: passes through all input items unchanged.
- **Key variables/expressions:**
  - `$input.first().json.message.from.id`
- **Inputs/Outputs:**
  - Input from **Get Message**.
  - Output to **Search with MCP?**.
- **Failure/edge cases:**
  - If `message.from.id` is missing (uncommon), expression will throw.
  - Unauthorized path still continues to the IF node (no explicit stop); you may want an additional IF to terminate unauthorized requests.

#### Node: Search with MCP?
- **Type / role:** `if` — command filter.
- **Config (interpreted):**
  - Condition: `{{ $json.message.text }}` **startsWith** `/stitch`
  - Case-sensitive (default in this node config).
- **Inputs/Outputs:**
  - Input from **Code**.
  - **True** output connected to **Get Text**.
  - False path is not connected (messages without `/stitch` are effectively ignored).
- **Failure/edge cases:**
  - If unauthorized code path outputs `{ unauthorized: true }` without `message.text`, the condition can error depending on n8n evaluation; safer to guard for missing `message.text`.
  - `/Stitch` (capital S) won’t match due to case sensitivity.

#### Sticky Note8 (covers intake/gating)
Content:
- “## STEP 1 - Set Telegram Bot  
  Set your Telegram ID here. the search only occurs when the command "/stitch" is present in the message”

#### Sticky Note1 (global explanation)
Contains the high-level description and setup guidance for Telegram, Stitch API key header auth, Gemini, and Perplexity.

---

### 2.2 Input Normalization

**Overview:** Extracts the Telegram message into standard fields expected by the agent and strips the `/stitch ` command prefix.

**Nodes involved:**  
- **Get Text** (set)  
- **Clean query** (code)

#### Node: Get Text
- **Type / role:** `set` — maps Telegram payload to agent-friendly fields.
- **Config (interpreted):**
  - Sets `chatInput` = `{{ $json.message.text }}`
  - Sets `sessionId` = `{{ $json.message.from.id }}`
- **Inputs/Outputs:**
  - Input from **Search with MCP?** (true path).
  - Output to **Clean query**.
- **Failure/edge cases:**
  - If Telegram message has no `text` (e.g., photo-only message), `chatInput` becomes undefined.

#### Node: Clean query
- **Type / role:** `code` — removes command prefix.
- **Config (interpreted):**
  - For each item: `originalText.replace("/stitch ", "")`
  - Overwrites `item.json.chatInput` with the cleaned query.
- **Inputs/Outputs:**
  - Input from **Get Text**.
  - Output to **Google Stitch Agent** (main).
- **Failure/edge cases:**
  - Only removes the exact substring `"/stitch "` (with trailing space). If user sends `/stitch` with no space, the query remains `/stitch`.
  - If the user writes `/stitch   something` multiple spaces, only the first exact `"/stitch "` occurrence is removed; extra spaces remain.

---

### 2.3 AI Agent Orchestration (Gemini + Memory + MCP Tools + Web Search)

**Overview:** The agent receives the cleaned prompt, uses Gemini as its LLM, retains short-term memory, and can call Google Stitch MCP tools (create/list/get projects and screens, generate screens) and Perplexity web search.

**Nodes involved:**  
- **Google Stitch Agent** (agent)  
- **Google Gemini Chat Model** (lmChatGoogleGemini)  
- **Simple Memory** (memoryBufferWindow)  
- **Create Project** (mcpClientTool)  
- **Get Project** (mcpClientTool)  
- **List projects** (mcpClientTool)  
- **List screen** (mcpClientTool)  
- **Get screen** (mcpClientTool)  
- **Generate Screen** (mcpClientTool)  
- **Search on web** (perplexityTool)  
- **Sticky Note** (Stitch setup documentation)

#### Node: Google Stitch Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — tool-using agent that orchestrates calls.
- **Config (interpreted):**
  - Has a detailed **system message** instructing:
    - enumerate available Stitch MCP functions
    - choose best tool(s) per request
    - make assumptions explicit if needed
    - summarize outputs and explain tool usage
    - “Do not hallucinate Stitch features not exposed”
    - “Use search on web tool if necessary”
- **Key inputs:**
  - Main input: items containing `chatInput` and `sessionId` (from **Clean query** or from **When chat message received** entry).
  - Language model input: **Google Gemini Chat Model** via `ai_languageModel`.
  - Memory input: **Simple Memory** via `ai_memory`.
  - Tools available via `ai_tool` connections: all Stitch MCP nodes + Perplexity.
- **Outputs:**
  - Main output goes to **is Telegram?** to decide formatting and sending.
- **Failure/edge cases:**
  - If tool credentials fail (Stitch/Perplexity), the agent may error or produce partial response.
  - If the agent output is not placed in the expected field used downstream (`$json.output` in the Markdown→HTML step), formatting can break (depends on agent node’s output schema/version).
- **Version considerations:**
  - Node typeVersion **3.1**; tool wiring uses LangChain ports (`ai_tool`, `ai_memory`, `ai_languageModel`). Requires n8n version that supports these ports for LangChain agent nodes.

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` — primary LLM for the agent.
- **Config (interpreted):**
  - Uses “Google Gemini(PaLM) (Eure)” credentials.
  - Default options (no explicit model/temperature shown in JSON).
- **Connections:**
  - Output (`ai_languageModel`) connected to **Google Stitch Agent**.
- **Failure/edge cases:**
  - Invalid API key / project restrictions / quota exhaustion.
  - Safety filters or content policy blocks could refuse generation.

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow` — conversational memory store.
- **Config (interpreted):**
  - Default configuration (buffer window size not specified in JSON; n8n default applies).
- **Connections:**
  - Output (`ai_memory`) connected to **Google Stitch Agent**.
- **Failure/edge cases:**
  - If `sessionId` is missing/unstable, memory may not thread correctly across messages.

#### Nodes: Google Stitch MCP tools (Create/Get/List Projects; List/Get/Generate Screens)
All are `@n8n/n8n-nodes-langchain.mcpClientTool` and share the same pattern:

- **Common role:** Expose specific Stitch MCP functions to the agent as callable tools.
- **Common config (interpreted):**
  - `endpointUrl`: `https://stitch.googleapis.com/mcp`
  - `authentication`: `headerAuth`
  - `timeout`: 600,000 ms (10 minutes) to allow slower generation operations
  - `include: selected` with one tool each:
    - **Create Project** → `create_project`
    - **Get Project** → `get_project`
    - **List projects** → `list_projects`
    - **List screen** → `list_screens`
    - **Get screen** → `get_screen`
    - **Generate Screen** → `generate_screen_from_text`
  - Credentials: HTTP Header Auth “Google Stitch”
- **Connections:**
  - Each node’s `ai_tool` output is connected into **Google Stitch Agent**.
- **Failure/edge cases:**
  - Wrong/expired API key in header; missing required header name.
  - MCP endpoint downtime, latency; tool timeouts beyond 10 minutes.
  - Invalid tool inputs generated by the LLM (schema mismatch) causing tool call failures.
  - Stitch API permissions or project/screen IDs not found.

#### Node: Search on web
- **Type / role:** `perplexityTool` — web search tool for the agent.
- **Config (interpreted):**
  - Sends a single message whose content comes from an AI-injected field:  
    `{{ $fromAI('message0_Text', ``, 'string') }}`
  - Uses Perplexity credentials “Perplexity account”.
- **Connections:**
  - `ai_tool` output to **Google Stitch Agent**.
- **Failure/edge cases:**
  - If the agent does not populate `message0_Text`, the tool may send an empty query.
  - Perplexity API quota/auth errors.

#### Sticky Note (Stitch setup)
Content:
- “## STEP 2 - Google Stitch  
  [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) and set Header Auth with name: "X-Goog-Api-Key" and value "YOUR-API-KEY"”

---

### 2.4 Response Formatting + Delivery (Telegram)

**Overview:** Only if the run originated from Telegram, convert the agent’s Markdown output to Telegram-compatible HTML and send it back to the originating chat.

**Nodes involved:**  
- **is Telegram?** (if)  
- **From MD to HTML** (chainLlm)  
- **Google Gemini Chat Model1** (lmChatGoogleGemini)  
- **Send a text message** (telegram)  
- **Sticky Note9** (documentation)

#### Node: is Telegram?
- **Type / role:** `if` — checks whether Telegram trigger executed.
- **Config (interpreted):**
  - Condition checks boolean `true` for: `{{ $('Get Message').isExecuted }}`
  - Ignore case enabled, but irrelevant for boolean check.
- **Connections:**
  - True output goes to **From MD to HTML**.
  - False output not connected (so non-Telegram entry won’t send a Telegram message).
- **Failure/edge cases:**
  - If executed from the alternate entry point (“When chat message received”), this will be false and no message is sent (expected).

#### Node: From MD to HTML
- **Type / role:** `chainLlm` — LLM-based formatting step.
- **Config (interpreted):**
  - Input text: `{{ $json.output }}`
  - Prompt instructs: convert Markdown to Telegram-safe HTML, **without** code fences, only allowed tags.
  - Uses **Google Gemini Chat Model1** as its `ai_languageModel`.
- **Connections:**
  - Main output to **Send a text message**.
- **Failure/edge cases:**
  - If agent output is not in `$json.output`, conversion receives empty/undefined.
  - LLM may output unsupported HTML tags; Telegram may reject message or strip formatting.
  - Very long responses may exceed Telegram message limits.

#### Node: Google Gemini Chat Model1
- **Type / role:** `lmChatGoogleGemini` — LLM dedicated to Markdown→HTML conversion.
- **Config (interpreted):**
  - Uses different Gemini credentials: “Google Gemini(PaLM) Api account”.
- **Connections:**
  - `ai_languageModel` to **From MD to HTML**.
- **Failure/edge cases:**
  - Same as other Gemini node (quota/auth).

#### Node: Send a text message
- **Type / role:** `telegram` — sends response back to the user in Telegram.
- **Config (interpreted):**
  - `text`: `{{ $json.text }}` (expects the conversion node to output `text`)
  - `chatId`: `{{ $('Get Message').item.json.message.from.id }}`
  - `parse_mode`: `HTML`
- **Connections:**
  - Input from **From MD to HTML**.
- **Failure/edge cases:**
  - If `chatId` missing (non-Telegram runs), expression fails (but gated by `is Telegram?`).
  - If HTML is invalid per Telegram rules, send may fail.
  - Bot blocked by user or chat permissions prevent sending.

#### Sticky Note9 (covers response block)
Content:
- “## STEP 3- Send Response  
  Set response message for Telegram and send it”

---

### 2.5 Secondary Entry Point (n8n Chat)

**Overview:** Allows triggering the agent via n8n’s LangChain chat trigger. It bypasses Telegram authorization and command gating, and will not send Telegram replies.

**Nodes involved:**  
- **When chat message received** (chatTrigger)

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for n8n chat-based interactions.
- **Config (interpreted):**
  - Default options; provides chat input payload to the workflow.
- **Connections:**
  - Main output to **Google Stitch Agent**.
- **Failure/edge cases:**
  - Because `is Telegram?` will be false, responses won’t be delivered anywhere unless you add additional output handling for this path.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Secondary entry point (n8n chat) | — | Google Stitch Agent | ## AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini… (full note applies) |
| Get Message | n8n-nodes-base.telegramTrigger | Telegram message intake | — | Code | ## STEP 1 - Set Telegram Bot… |
| Code | n8n-nodes-base.code | Authorize sender by Telegram ID | Get Message | Search with MCP? | ## STEP 1 - Set Telegram Bot… |
| Search with MCP? | n8n-nodes-base.if | Gate: only `/stitch…` messages | Code | Get Text | ## STEP 1 - Set Telegram Bot… |
| Get Text | n8n-nodes-base.set | Map Telegram payload to `chatInput` and `sessionId` | Search with MCP? | Clean query | ## STEP 1 - Set Telegram Bot… |
| Clean query | n8n-nodes-base.code | Strip `/stitch ` prefix | Get Text | Google Stitch Agent | ## STEP 1 - Set Telegram Bot… |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Main LLM for agent | — | Google Stitch Agent | ## AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini… |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation memory | — | Google Stitch Agent | ## AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini… |
| Create Project | @n8n/n8n-nodes-langchain.mcpClientTool | Stitch MCP tool: create_project | — | Google Stitch Agent | ## STEP 2 - Google Stitch… [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) … |
| Get Project | @n8n/n8n-nodes-langchain.mcpClientTool | Stitch MCP tool: get_project | — | Google Stitch Agent | ## STEP 2 - Google Stitch… [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) … |
| List projects | @n8n/n8n-nodes-langchain.mcpClientTool | Stitch MCP tool: list_projects | — | Google Stitch Agent | ## STEP 2 - Google Stitch… [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) … |
| List screen | @n8n/n8n-nodes-langchain.mcpClientTool | Stitch MCP tool: list_screens | — | Google Stitch Agent | ## STEP 2 - Google Stitch… [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) … |
| Get screen | @n8n/n8n-nodes-langchain.mcpClientTool | Stitch MCP tool: get_screen | — | Google Stitch Agent | ## STEP 2 - Google Stitch… [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) … |
| Generate Screen | @n8n/n8n-nodes-langchain.mcpClientTool | Stitch MCP tool: generate_screen_from_text | — | Google Stitch Agent | ## STEP 2 - Google Stitch… [Get your API Key](https://stitch.withgoogle.com/docs/mcp/setup) … |
| Search on web | n8n-nodes-base.perplexityTool | Tool: web search for extra context | — | Google Stitch Agent | ## AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini… |
| Google Stitch Agent | @n8n/n8n-nodes-langchain.agent | Orchestrator: decides tools, calls Stitch, composes answer | Clean query; When chat message received | is Telegram? | ## AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini… |
| is Telegram? | n8n-nodes-base.if | Gate: only send Telegram reply on Telegram runs | Google Stitch Agent | From MD to HTML | ## STEP 3- Send Response… |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for Markdown→HTML conversion | — | From MD to HTML | ## STEP 3- Send Response… |
| From MD to HTML | @n8n/n8n-nodes-langchain.chainLlm | Convert Markdown to Telegram-safe HTML | is Telegram? | Send a text message | ## STEP 3- Send Response… |
| Send a text message | n8n-nodes-base.telegram | Send final HTML response to user | From MD to HTML | — | ## STEP 3- Send Response… |
| Sticky Note | n8n-nodes-base.stickyNote | Comment: Stitch API key setup | — | — | ## STEP 2 - Google Stitch… |
| Sticky Note8 | n8n-nodes-base.stickyNote | Comment: Telegram setup step | — | — | ## STEP 1 - Set Telegram Bot… |
| Sticky Note9 | n8n-nodes-base.stickyNote | Comment: Response sending step | — | — | ## STEP 3- Send Response… |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment: full workflow description & setup | — | — | ## AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   1. Add node: **Telegram Trigger** named **Get Message**
   2. Updates: **Message**
   3. Credentials: connect your **Telegram Bot API** credentials (token).
   4. Activate later to register webhook.

2) **Add Authorization Gate**
   1. Add node: **Code** named **Code**
   2. Paste logic to allow only your Telegram user ID:
      - Compare `message.from.id` to your numeric Telegram ID.
      - If not equal, return `{ unauthorized: true }` (or stop execution by returning empty array).
   3. Connect: **Get Message → Code**

3) **Add `/stitch` Command Filter**
   1. Add node: **IF** named **Search with MCP?**
   2. Condition: String → `{{$json.message.text}}` **startsWith** `/stitch`
   3. Connect: **Code → Search with MCP?** (main)

4) **Extract Text + Session**
   1. Add node: **Set** named **Get Text**
   2. Add fields:
      - `chatInput` (String) = `{{$json.message.text}}`
      - `sessionId` (Number) = `{{$json.message.from.id}}`
   3. Connect: **Search with MCP? (true) → Get Text**

5) **Clean the Prompt**
   1. Add node: **Code** named **Clean query**
   2. Replace `/stitch ` prefix in `chatInput`.
   3. Connect: **Get Text → Clean query**

6) **Add Gemini LLM for the Agent**
   1. Add node: **Google Gemini Chat Model** named **Google Gemini Chat Model**
   2. Credentials: set up **Google Gemini / PaLM API** credentials.
   3. (Optional) choose model/temperature as desired.

7) **Add Memory**
   1. Add node: **Simple Memory** (Buffer Window Memory)
   2. Keep defaults (or set window size).
   3. Ensure your agent uses `sessionId` consistently (the upstream Set node provides it).

8) **Add Google Stitch MCP Tools (Header Auth)**
   1. Create credential: **HTTP Header Auth**
      - Header name: `X-Goog-Api-Key`
      - Value: your Stitch API key from: https://stitch.withgoogle.com/docs/mcp/setup
   2. Add 6 nodes of type **MCP Client Tool** with:
      - Endpoint URL: `https://stitch.googleapis.com/mcp`
      - Authentication: **Header Auth** (select the credential above)
      - Timeout: `600000` ms
      - Include tools: select exactly one per node:
        - `create_project`
        - `get_project`
        - `list_projects`
        - `list_screens`
        - `get_screen`
        - `generate_screen_from_text`

9) **Add Web Search Tool (Perplexity)**
   1. Add node: **Perplexity Tool** named **Search on web**
   2. Credentials: Perplexity API key
   3. Message content should be AI-provided (as in the original): use the node’s AI-input mapping so the agent can pass the query.

10) **Create the Agent**
   1. Add node: **AI Agent** named **Google Stitch Agent**
   2. Paste the system message that instructs tool enumeration, correct tool usage, and “no hallucination”.
   3. Connect agent ports:
      - **Google Gemini Chat Model** → Agent (`ai_languageModel`)
      - **Simple Memory** → Agent (`ai_memory`)
      - Each MCP tool node → Agent (`ai_tool`)
      - **Search on web** → Agent (`ai_tool`)
   4. Connect main data:
      - **Clean query → Google Stitch Agent** (main)

11) **Gate Telegram Output**
   1. Add node: **IF** named **is Telegram?**
   2. Condition: Boolean true for `{{$('Get Message').isExecuted}}`
   3. Connect: **Google Stitch Agent → is Telegram?**

12) **Convert Markdown to Telegram HTML**
   1. Add node: **Chain LLM** named **From MD to HTML**
   2. Input text: `{{$json.output}}` (adjust if your agent outputs a different field)
   3. Prompt: instruct conversion to Telegram-supported HTML tags only.
   4. Add a second **Google Gemini Chat Model** named **Google Gemini Chat Model1**
      - Credentials can be same or different.
   5. Connect: **Google Gemini Chat Model1 → From MD to HTML** (`ai_languageModel`)
   6. Connect: **is Telegram? (true) → From MD to HTML**

13) **Send Telegram Message**
   1. Add node: **Telegram** named **Send a text message**
   2. Operation: send message
   3. `chatId`: `{{$('Get Message').item.json.message.from.id}}`
   4. `text`: `{{$json.text}}` (output of conversion step)
   5. Parse mode: **HTML**
   6. Connect: **From MD to HTML → Send a text message**

14) **(Optional) Add n8n Chat Trigger entry**
   1. Add node: **When chat message received** (LangChain Chat Trigger)
   2. Connect: **When chat message received → Google Stitch Agent**
   3. If you want non-Telegram runs to return somewhere, add a separate output path (e.g., respond in chat UI), since the current flow only sends to Telegram when Telegram trigger executed.

15) **Activate the workflow**
   - Activating registers the Telegram webhook.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your API Key and configure Header Auth with `X-Goog-Api-Key` | https://stitch.withgoogle.com/docs/mcp/setup |
| Telegram runs are authorized by hardcoding your Telegram user ID in the Code node; processing only happens when `/stitch` is present | Sticky note “STEP 1 - Set Telegram Bot” |
| Responses are converted from Markdown to Telegram-compatible HTML and sent back to the user | Sticky note “STEP 3- Send Response” |
| Full workflow concept: Telegram-based AI design assistant using Gemini + Stitch MCP, with optional web search | Sticky note “AI-powered Google Stitch Design Agent via Telegram using MCP and Gemini” |
| Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Provided in the request (global) |