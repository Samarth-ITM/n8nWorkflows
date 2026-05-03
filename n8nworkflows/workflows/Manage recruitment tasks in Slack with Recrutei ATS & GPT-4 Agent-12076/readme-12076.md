Manage recruitment tasks in Slack with Recrutei ATS & GPT-4 Agent

https://n8nworkflows.xyz/workflows/manage-recruitment-tasks-in-slack-with-recrutei-ats---gpt-4-agent-12076


# Manage recruitment tasks in Slack with Recrutei ATS & GPT-4 Agent

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow turns Slack into an internal “AI recruitment agent” interface for **Recrutei ATS**. When someone mentions the bot/app in a Slack channel, an **AI Agent (LangChain)** interprets the request and can both **answer questions** and **perform actions** in Recrutei (via MCP and Recrutei API endpoints). The agent posts the result back to the same Slack channel.

**Typical use cases**
- Quickly fetch or reason about recruitment data from Recrutei using MCP.
- Manage vacancy comments (create/list/delete).
- Manage comment answers (list/create answers).
- Manage candidate tags (create/list/delete tags).
- Create “prospect” candidates in a vacancy.

### 1.1 Input Reception & Normalization (Slack → cleaned fields)
Receives Slack mentions and extracts only the useful fields (message text + channel id) to standardize the agent prompt and later route responses to the correct channel.

### 1.2 AI Reasoning + Conversational Memory (prompt orchestration)
A LangChain Agent uses an OpenAI chat model and a buffer memory keyed per Slack channel to keep context across multiple messages.

### 1.3 Toolset (Recrutei MCP + Recrutei HTTP API tools)
The agent can call multiple “Tool” nodes:
- MCP Client (for Recrutei data via MCP server)
- Get token (login API to retrieve Recrutei bearer token)
- Candidate prospection (create prospect candidate)
- Tag management (create/list/delete)
- Vacancy comment management (create/list/delete)
- Comment answer management (list/create answers)

### 1.4 Response Delivery (Agent → Slack)
Posts the agent’s final answer back into the originating Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Context
**Overview:** Captures a Slack app mention event and prepares minimal fields for the agent. Ensures later nodes can reliably access `text` and `channel`.

**Nodes involved:**  
- Slack Trigger  
- Separates the message

#### Node: Slack Trigger
- **Type / role:** `n8n-nodes-base.slackTrigger` — entry point; listens to Slack events.
- **Configuration (interpreted):**
  - Trigger event: **app_mention**
  - Channel selection: configured via “list”, but **channelId is empty** in the template (must be set during setup or left to allow mentions depending on Slack app/event config).
- **Inputs/Outputs:**
  - **Output:** Slack event payload (includes `text`, `channel`, etc.) → to **Separates the message**
- **Credentials:** Slack API credential required (OAuth/token managed by n8n).
- **Version notes:** typeVersion **1**.
- **Edge cases / failures:**
  - Missing/incorrect Slack credentials or scopes (cannot receive events).
  - Slack app not subscribed to `app_mention` events or bot not invited to the channel.
  - If channel filtering is required but `channelId` is blank, you may not receive events as expected (depends on Slack app/event subscription and workspace settings).

#### Node: Separates the message
- **Type / role:** `n8n-nodes-base.set` — normalizes the incoming payload.
- **Configuration (interpreted):**
  - Creates/keeps two fields:
    - `text` = `{{$json.text}}`
    - `channel` = `{{$json.channel}}`
- **Inputs/Outputs:**
  - **Input:** Slack Trigger event JSON
  - **Output:** Clean object with `text` and `channel` → to **AI Agent**
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**
  - If Slack payload does not include `text` (rare but possible in some event types), agent prompt becomes empty.
  - If `channel` is missing, memory and response routing will fail (session key and Slack reply depend on it).

---

### Block 2 — AI Reasoning & Memory
**Overview:** The AI Agent uses the OpenAI model for reasoning, and a windowed memory keyed by Slack channel to keep conversational continuity.

**Nodes involved:**  
- OpenAI  
- Simple Memory  
- AI Agent

#### Node: OpenAI
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — chat language model backing the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini** (selected from list)
  - No special options set.
  - Built-in tools: none enabled here (tools are provided via separate Tool nodes).
- **Inputs/Outputs:**
  - Provides `ai_languageModel` connection into **AI Agent**
- **Credentials:** OpenAI API credential required.
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - Invalid API key, quota exceeded, or model unavailable in the account.
  - Latency/timeouts on long tool interactions.
  - If you intended GPT-4.1 (full) but selected “mini”, reasoning quality may differ.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — stores recent conversation turns.
- **Configuration (interpreted):**
  - Session ID type: **customKey**
  - Session key: `={{ $('Separates the message').item.json.channel }}`
  - Context window length: **20** messages/turns (buffer window)
  - Meaning: memory is **shared per Slack channel**, not per user or per thread.
- **Inputs/Outputs:**
  - Provides `ai_memory` connection into **AI Agent**
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - Because the key is the channel id, conversations from multiple users in the same channel will mix context.
  - If the `channel` field is missing, memory session key evaluation fails.
  - Memory growth is bounded by 20 turns, but still may contain irrelevant context if multiple topics occur in the same channel.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrator that decides when to call tools and crafts final response.
- **Configuration (interpreted):**
  - Prompt input text: `={{ $json.text }}`
  - System message: HR internal assistant instructions + extensive guidance for configuring MCP and API tools + links to Recrutei developer docs.
  - `promptType: define` (explicit system prompt defined in node)
  - Retry on fail enabled.
- **Tools connected (via `ai_tool`):**
  - MCP Client
  - Get token
  - Creating a prospect candidate in a vacancy
  - Creating a tag in a candidate
  - Listing created tags
  - Listing candidates tags
  - Deleting candidate tags
  - Create a comment in a vacancy
  - Listing vacancy comments
  - Deleting a comment on a vacancy
  - Listing answers in a comment
  - Answering a comment
- **Inputs/Outputs:**
  - **Input (main):** from Separates the message
  - **Input (ai_languageModel):** from OpenAI
  - **Input (ai_memory):** from Simple Memory
  - **Output (main):** `output` field (agent final text) → Send a message
- **Version notes:** typeVersion **3**
- **Edge cases / failures:**
  - If tool headers are not configured (Bearer token placeholders), tool calls will fail with 401/403.
  - The system prompt expects the agent to help configure MCP/API; if the user asks to do actions before setup, tool calls likely fail—agent should guide but may still attempt calls.
  - Tool URL parameters are generated using `$fromAI(...)`; if the agent fails to construct valid endpoints or misses IDs, requests fail (400/404).

---

### Block 3 — Toolset: MCP & Token
**Overview:** Gives the agent read/execute capabilities via Recrutei MCP and a helper login endpoint to obtain a bearer token for other API calls.

**Nodes involved:**  
- MCP Client  
- Get token

#### Node: MCP Client
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — MCP tool connector (SSE/HTTP streamable).
- **Configuration (interpreted):**
  - Endpoint URL: **`[your_mcp_url]`** placeholder must be replaced with the user’s MCP URL from Recrutei (contains token in URL per system instructions).
- **Inputs/Outputs:**
  - Exposed to AI Agent as an `ai_tool`.
- **Version notes:** typeVersion **1.2**
- **Edge cases / failures:**
  - If endpoint URL is not replaced, calls will fail immediately.
  - MCP token expiration (depending on configured expiry) causes authentication failures.
  - System message notes a known issue: **`list_candidates_by_stage`** may be troublesome; agent should retrieve `pipe_stages` from vacancy search results (`pipeVacancy.pipe.pipe_stages`) and pass correct integer IDs.

#### Node: Get token
- **Type / role:** `n8n-nodes-base.httpRequestTool` — HTTP tool that logs in and returns Recrutei bearer token.
- **Configuration (interpreted):**
  - Method: POST
  - URL: `https://api.recrutei.com.br/api/v1/login-api`
  - Body: `email`, `password` gathered via `$fromAI(...)` (agent asks user)
  - Headers: `x-api-key`, `x-api-secret` placeholders must be filled with Recrutei API credentials.
  - Tool description instructs agent to return `token` field from response.
- **Inputs/Outputs:**
  - Exposed to AI Agent as an `ai_tool`.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - If `x-api-key`/`x-api-secret` wrong or missing → 401/403.
  - Wrong email/password → login failure.
  - Returning token in a response field may change; if API response schema differs, agent may not find `token`.

---

### Block 4 — Toolset: Candidate Prospection
**Overview:** Allows the agent to create a “prospect” candidate directly into a vacancy and stage, useful for sourcing external candidates.

**Nodes involved:**  
- Creating a prospect candidate in a vacancy

#### Node: Creating a prospect candidate in a vacancy
- **Type / role:** `n8n-nodes-base.httpRequestTool` — creates a prospect candidate (POST).
- **Configuration (interpreted):**
  - URL: `https://api.recrutei.com.br/api/v2/prospects`
  - Method: POST
  - Body fields:
    - `prospect_acquisition_channel_id` fixed as `"8"` (hard-coded)
    - `name`, `pipe_stage_id`, `vacancy_id`, `email`, `telephone` via `$fromAI(...)`
  - Headers:
    - `Authorization: Bearer [your_token]` (placeholder)
    - `Content-type: multipart/form-data`
- **Inputs/Outputs:** Exposed to AI Agent as `ai_tool`.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - Missing or expired bearer token → 401.
  - `pipe_stage_id` must exist for the vacancy pipeline; wrong ID → 400/404.
  - Email/telephone format validation may reject input.
  - Hard-coded acquisition channel id may not match the customer’s workspace configuration.

---

### Block 5 — Toolset: Tag Management
**Overview:** Lets the agent apply organization labels to candidates and query or remove those tags.

**Nodes involved:**  
- Creating a tag in a candidate  
- Listing created tags  
- Listing candidates tags  
- Deleting candidate tags

#### Node: Creating a tag in a candidate
- **Type / role:** HTTP Request Tool — creates/applies a tag to a talent/application.
- **Configuration (interpreted):**
  - URL: `https://api.recrutei.com.br/api/v2/tags/talent` (note: template includes a trailing tab character in JSON; should be removed in practice)
  - Method: POST
  - Body:
    - `application_id` via `$fromAI(...)`
    - `color` via `$fromAI(...)` (agent should convert user color to hex)
    - `key` fixed as `generic`
    - `talent_id` via `$fromAI(...)`
    - `value` (tag name) via `$fromAI(...)`
  - Header: `Authorization: Bearer [your_token]`
- **Inputs/Outputs:** Exposed to AI Agent as `ai_tool`.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - If the URL contains an invisible tab, the request can fail with “Invalid URL”.
  - If `application_id`/`talent_id` mismatch, API may reject.
  - Color must be valid hex if provided.

#### Node: Listing created tags
- **Type / role:** HTTP Request Tool — lists available tag values.
- **Configuration (interpreted):**
  - URL: `https://api.recrutei.com.br/api/v2/tags/available-values`
  - Method: GET (default since not set)
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:** token/auth errors; API pagination (if any) not handled explicitly.

#### Node: Listing candidates tags
- **Type / role:** HTTP Request Tool — lists tags for a specific talent.
- **Configuration (interpreted):**
  - URL is built by the agent via `$fromAI('URL', ...)` with template:
    - `https://api.recrutei.com.br/api/v2/tags/talents/{talent_id}`
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - Agent must correctly substitute `{talent_id}`.
  - 404 if talent id does not exist.

#### Node: Deleting candidate tags
- **Type / role:** HTTP Request Tool — deletes a talent tag by tag id.
- **Configuration (interpreted):**
  - Method: DELETE
  - URL built by agent via `$fromAI('URL', ...)` with template:
    - `https://api.recrutei.com.br/api/v2/tags/talents/{id_tag}`
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - Confusion between `talent_id` and `id_tag` can lead to accidental failures.
  - 404 if tag id not found; 403 if user lacks permission.

---

### Block 6 — Toolset: Vacancy Comment Management
**Overview:** Enables creating, listing, and deleting comments on a vacancy.

**Nodes involved:**  
- Create a comment in a vacancy  
- Listing vacancy comments  
- Deleting a comment on a vacancy

#### Node: Create a comment in a vacancy
- **Type / role:** HTTP Request Tool — creates a comment on a vacancy.
- **Configuration (interpreted):**
  - Method: POST
  - URL built by agent via `$fromAI('URL', ...)` using template:
    - `https://api.recrutei.com.br/api/v2/vacancycomments/[vacancy_id]/comments`
  - Body:
    - `text` (comment content)
    - `idVacancy` (vacancy id)
    - `private` (expects boolean semantics; template treats as string—agent should pass true/false)
  - Headers:
    - Authorization bearer placeholder
    - `content-type: multipart/form-data`
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - URL must include correct vacancy id; body also includes vacancy id—mismatch could error.
  - `private` type mismatch (string vs boolean) may break API validation.
  - Permissions: not all users can post public/private comments.

#### Node: Listing vacancy comments
- **Type / role:** HTTP Request Tool — fetches comments for a vacancy.
- **Configuration (interpreted):**
  - URL built by agent via `$fromAI('URL', ...)`:
    - `https://api.recrutei.com.br/api/v2/vacancycomments/[vacancy_id]/comments`
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:** wrong vacancy id → 404; large comment history might paginate (not handled explicitly).

#### Node: Deleting a comment on a vacancy
- **Type / role:** HTTP Request Tool — deletes a specific comment.
- **Configuration (interpreted):**
  - Method: DELETE
  - URL built by agent via `$fromAI('URL', ...)`:
    - `https://api.recrutei.com.br/api/v2/vacancycomments/[vacancy_id]/comments/[comment_id]`
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - Requires correct `comment_id` (often found by listing comments first).
  - Deleting may be restricted by permissions/ownership.

---

### Block 7 — Toolset: Comment Answer Management
**Overview:** Supports listing answers to a vacancy comment and posting new answers.

**Nodes involved:**  
- Listing answers in a comment  
- Answering a comment

#### Node: Listing answers in a comment
- **Type / role:** HTTP Request Tool — lists answers for a given comment.
- **Configuration (interpreted):**
  - URL built by agent via `$fromAI('URL', ...)`:
    - `https://api.recrutei.com.br/api/v2/vacancycomments/[idVacancy]/comments/[idComment]/answers`
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:** wrong vacancy/comment ids; permission restrictions.

#### Node: Answering a comment
- **Type / role:** HTTP Request Tool — posts a new answer to a comment.
- **Configuration (interpreted):**
  - Method: POST
  - URL built by agent via `$fromAI('URL', ...)`:
    - `https://api.recrutei.com.br/api/v2/vacancycomments/[vacancy_id]/comments/[comment_id]/answers`
  - Body:
    - `idVacancy`
    - `id` (comment id)
    - `text` (answer content)
  - Header: Authorization bearer placeholder
- **Inputs/Outputs:** Exposed to AI Agent.
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**
  - Confusing `id` field naming (comment id) can cause wrong payload.
  - URL and body must match same vacancy/comment ids.

---

### Block 8 — Response Delivery
**Overview:** Sends the agent’s final output back to Slack, closing the loop.

**Nodes involved:**  
- Send a message

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.slack` — posts message to Slack channel.
- **Configuration (interpreted):**
  - Operation: “send message” to a **channel**
  - Channel ID: `={{ $('Separates the message').item.json.channel }}`
  - Text: `={{ $json.output }}`
  - Option: `includeLinkToWorkflow: false`
- **Inputs/Outputs:**
  - **Input:** from AI Agent output
  - **Output:** Slack API response (not used further)
- **Credentials:** Slack API credential required.
- **Version notes:** typeVersion **2.3**
- **Edge cases / failures:**
  - Bot lacks permission to post in channel or is not invited.
  - If agent output is empty or too long, Slack may reject or truncate.
  - This posts to channel; it does **not explicitly reply in a thread** (depends on additional Slack parameters not set here).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Slack Trigger | slackTrigger | Entry point: receive Slack app mentions | — | Separates the message | ## Input & Context<br>Captures the incoming message from Slack. The "Edit Fields" node cleans and prepares the user's text prompt, ensuring the AI Agent receives the input in the correct format to start processing. |
| Separates the message | set | Extract `text` and `channel` for downstream usage | Slack Trigger | AI Agent | ## Input & Context<br>Captures the incoming message from Slack. The "Edit Fields" node cleans and prepares the user's text prompt, ensuring the AI Agent receives the input in the correct format to start processing. |
| OpenAI | lmChatOpenAi | LLM provider for agent reasoning | — | AI Agent | ## The AI Brain<br>This section powers the agent's reasoning. It combines the AI Agent to orchestrate the use of tools, Window Buffer Memory to maintain conversation context (history), and the OpenAI Model to interpret the recruiter's intent. |
| Simple Memory | memoryBufferWindow | Per-channel conversational memory (last 20 turns) | — | AI Agent | ## The AI Brain<br>This section powers the agent's reasoning. It combines the AI Agent to orchestrate the use of tools, Window Buffer Memory to maintain conversation context (history), and the OpenAI Model to interpret the recruiter's intent. |
| AI Agent | langchain.agent | Orchestrates tools + generates final answer | Separates the message; OpenAI (ai_languageModel); Simple Memory (ai_memory); all Tools (ai_tool) | Send a message | ## The AI Brain<br>This section powers the agent's reasoning. It combines the AI Agent to orchestrate the use of tools, Window Buffer Memory to maintain conversation context (history), and the OpenAI Model to interpret the recruiter's intent. |
| Send a message | slack | Deliver agent output to Slack channel | AI Agent | — | ## Response Delivery<br>Finally, the agent sends its generated response back to the original Slack thread, completing the interaction loop. |
| MCP Client | mcpClientTool | Tool: Recrutei MCP access | — | AI Agent (ai_tool) | ## Toolset: MCP& Token<br>These tools allow the agent to fetch real-time data from Recrutei, such as active vacancies, specific candidate details, or lists of applicants for a specific role, and others. The Get token node makes a easier way for the user to get their token |
| Get token | httpRequestTool | Tool: obtain Recrutei bearer token via login | — | AI Agent (ai_tool) | ## Toolset: MCP& Token<br>These tools allow the agent to fetch real-time data from Recrutei, such as active vacancies, specific candidate details, or lists of applicants for a specific role, and others. The Get token node makes a easier way for the user to get their token |
| Creating a prospect candidate in a vacancy | httpRequestTool | Tool: create prospect candidate in vacancy/stage | — | AI Agent (ai_tool) | ## Toolset: Candidate Prospection<br>This tool allows the agent to create a candidate in a vacancy as a prospect, which means that it will have limited informatons, but it is usefull if the recruiter finds a good candidate in a external environment |
| Creating a tag in a candidate | httpRequestTool | Tool: apply/create tag for a candidate | — | AI Agent (ai_tool) | ## Toolset: Tag Management<br>Enables the AI to organize your ATS data by creating, listing, or deleting tags dynamically based on user requests. |
| Listing created tags | httpRequestTool | Tool: list available tag values | — | AI Agent (ai_tool) | ## Toolset: Tag Management<br>Enables the AI to organize your ATS data by creating, listing, or deleting tags dynamically based on user requests. |
| Listing candidates tags | httpRequestTool | Tool: list tags on a talent | — | AI Agent (ai_tool) | ## Toolset: Tag Management<br>Enables the AI to organize your ATS data by creating, listing, or deleting tags dynamically based on user requests. |
| Deleting candidate tags | httpRequestTool | Tool: delete a tag from a talent by tag id | — | AI Agent (ai_tool) | ## Toolset: Tag Management<br>Enables the AI to organize your ATS data by creating, listing, or deleting tags dynamically based on user requests. |
| Create a comment in a vacancy | httpRequestTool | Tool: create vacancy comment (private/public) | — | AI Agent (ai_tool) | ## Toolset: Vacancy Comment Management<br>These tools allow the agent to make complex action involving comments on vacancies |
| Listing vacancy comments | httpRequestTool | Tool: list vacancy comments | — | AI Agent (ai_tool) | ## Toolset: Vacancy Comment Management<br>These tools allow the agent to make complex action involving comments on vacancies |
| Deleting a comment on a vacancy | httpRequestTool | Tool: delete vacancy comment | — | AI Agent (ai_tool) | ## Toolset: Vacancy Comment Management<br>These tools allow the agent to make complex action involving comments on vacancies |
| Listing answers in a comment | httpRequestTool | Tool: list answers for a specific comment | — | AI Agent (ai_tool) | ## Toolset: Comment's Answer Management<br>These tools allows the agent to make answers in vacancies's comments and also listing them |
| Answering a comment | httpRequestTool | Tool: post an answer to a comment | — | AI Agent (ai_tool) | ## Toolset: Comment's Answer Management<br>These tools allows the agent to make answers in vacancies's comments and also listing them |
| Sticky Note | stickyNote | Documentation/overview in canvas | — | — | ## Overview: Recrutei ATS - Ultimate Internal AI Agent<br>This workflow transforms Slack into a powerful command center for recruitment. Using an AI Agent (LangChain) integrated with the Recrutei ATS API, your team can manage candidates, vacancies, and tags directly through Slack messages.<br><br>## Setup Instructions<br>1- **Slack Trigger:** Connect your Slack account and invite the bot to your desired channel.<br>2- **OpenAI:** Configure your API Key. This agent uses GPT-4.1 for high-reasoning capabilities.<br>3- **HTTP Request Tools:** Every "Tool" node (Pink nodes) needs your Recrutei API Token. <br>   - Replace the `Authorization` header with your `Bearer YOUR_TOKEN_HERE`.<br>   - Update the Base URL if necessary.<br>4- **Slack Post:** Ensure the bot has permissions to write in the channel.<br><br>The agent can help you to configure the workflow, just connect the Chat Trigger Node in it, more informations, access the notes in the AI Agente Node |
| Sticky Note1 | stickyNote | Documentation/section label | — | — | |
| Sticky Note2 | stickyNote | Documentation/section label | — | — | |
| Sticky Note3 | stickyNote | Documentation/section label | — | — | |
| Sticky Note4 | stickyNote | Documentation/section label | — | — | |
| Sticky Note5 | stickyNote | Documentation/section label | — | — | |
| Sticky Note6 | stickyNote | Documentation/section label | — | — | |
| Sticky Note7 | stickyNote | Documentation/section label | — | — | |
| Sticky Note8 | stickyNote | Documentation/section label | — | — | |

> Note: Sticky Note nodes are included as nodes in the workflow, but they do not connect to execution paths.

---

## 4. Reproducing the Workflow from Scratch

1) **Create Slack Trigger**
   1. Add node: **Slack Trigger**
   2. Trigger event: **App Mention (`app_mention`)**
   3. Connect Slack credentials (create Slack app / OAuth in n8n).
   4. If desired, select the specific **Channel ID**; otherwise ensure Slack app event subscription is configured to deliver events your workflow expects.

2) **Add Set node to normalize input**
   1. Add node: **Set** (name it “Separates the message”)
   2. Add fields:
      - `text` (String) = `{{$json.text}}`
      - `channel` (String) = `{{$json.channel}}`
   3. Connect: **Slack Trigger → Separates the message**

3) **Add OpenAI chat model**
   1. Add node: **OpenAI Chat Model** (LangChain) (`lmChatOpenAi`)
   2. Choose model: **gpt-4.1-mini** (or your preferred GPT-4.1 variant)
   3. Attach OpenAI credentials (API key).

4) **Add Memory (per Slack channel)**
   1. Add node: **Window Buffer Memory** (`memoryBufferWindow`)
   2. Session ID type: **Custom Key**
   3. Session Key expression: `{{$('Separates the message').item.json.channel}}`
   4. Context window length: **20**

5) **Add AI Agent**
   1. Add node: **AI Agent** (`langchain.agent`)
   2. Prompt input: `{{$json.text}}`
   3. Set **System Message** to the HR assistant instructions (include MCP + API setup guidance and doc link).
   4. Connect:
      - **Separates the message → AI Agent** (main)
      - **OpenAI → AI Agent** (ai_languageModel)
      - **Simple Memory → AI Agent** (ai_memory)

6) **Add MCP tool**
   1. Add node: **MCP Client Tool** (`mcpClientTool`)
   2. Set Endpoint URL to the user’s Recrutei MCP URL (from Recrutei ATS Marketplace → MCP).  
      - Replace `[your_mcp_url]` with the real URL.
   3. Connect: **MCP Client (ai_tool) → AI Agent**

7) **Add “Get token” tool**
   1. Add node: **HTTP Request Tool** (name: “Get token”)
   2. Method: **POST**
   3. URL: `https://api.recrutei.com.br/api/v1/login-api`
   4. Enable: Send Headers + Send Body
   5. Headers:
      - `x-api-key`: your Recrutei value
      - `x-api-secret`: your Recrutei value
   6. Body parameters:
      - `email`: allow agent to fill (in your build, you can keep it as an AI-provided field or map from user input)
      - `password`: same
   7. Connect: **Get token (ai_tool) → AI Agent**

8) **Add remaining Recrutei API tools (HTTP Request Tool nodes)**
   - For each tool below, add an **HTTP Request Tool** and configure:
     - **Authorization header**: `Bearer <RECRUTEI_TOKEN>`
     - If using multipart: set `Content-Type: multipart/form-data` where required
     - Where the template uses dynamic URLs (`$fromAI('URL', ...)`), ensure the agent is allowed to construct the URL; alternatively, replace with a fixed base and pass IDs as parameters if you prefer stricter control.

   A. **Creating a prospect candidate in a vacancy**
   - POST `https://api.recrutei.com.br/api/v2/prospects`
   - Body: acquisition channel id (fixed or configurable), name, pipe_stage_id, vacancy_id, email, telephone
   - Connect as `ai_tool` to the agent.

   B. **Creating a tag in a candidate**
   - POST `https://api.recrutei.com.br/api/v2/tags/talent`  
   - Ensure there is **no trailing whitespace/tab** in the URL.
   - Body: application_id, color (hex), key=generic, talent_id, value
   - Connect as `ai_tool`.

   C. **Listing created tags**
   - GET `https://api.recrutei.com.br/api/v2/tags/available-values`
   - Connect as `ai_tool`.

   D. **Listing candidates tags**
   - GET `https://api.recrutei.com.br/api/v2/tags/talents/{talent_id}`
   - Connect as `ai_tool`.

   E. **Deleting candidate tags**
   - DELETE `https://api.recrutei.com.br/api/v2/tags/talents/{id_tag}`
   - Connect as `ai_tool`.

   F. **Create a comment in a vacancy**
   - POST `https://api.recrutei.com.br/api/v2/vacancycomments/{vacancy_id}/comments`
   - Body: text, idVacancy, private (true/false)
   - Connect as `ai_tool`.

   G. **Listing vacancy comments**
   - GET `https://api.recrutei.com.br/api/v2/vacancycomments/{vacancy_id}/comments`
   - Connect as `ai_tool`.

   H. **Deleting a comment on a vacancy**
   - DELETE `https://api.recrutei.com.br/api/v2/vacancycomments/{vacancy_id}/comments/{comment_id}`
   - Connect as `ai_tool`.

   I. **Listing answers in a comment**
   - GET `https://api.recrutei.com.br/api/v2/vacancycomments/{idVacancy}/comments/{idComment}/answers`
   - Connect as `ai_tool`.

   J. **Answering a comment**
   - POST `https://api.recrutei.com.br/api/v2/vacancycomments/{vacancy_id}/comments/{comment_id}/answers`
   - Body: idVacancy, id (comment id), text
   - Connect as `ai_tool`.

9) **Add Slack “Send a message” node**
   1. Add node: **Slack** (operation: send message)
   2. Channel ID: `{{$('Separates the message').item.json.channel}}`
   3. Text: `{{$json.output}}`
   4. Connect: **AI Agent → Send a message**

10) **Permissions & credentials checklist**
   - Slack App:
     - Must receive `app_mention` events
     - Bot invited to channel
     - Permission to write messages
   - OpenAI:
     - Valid API key; model accessible
   - Recrutei:
     - MCP URL filled
     - API key/secret configured on **Get token**
     - Bearer token inserted for all other HTTP tools (or implement a secure token storage strategy)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Recrutei API documentation | https://developers.recrutei.com.br/docs/getting-started |
| MCP setup guidance is embedded in the AI Agent system message (Recrutei ATS → Configurations → Marketplace → MCP; URL contains token; token can’t be copied again after creation). | From AI Agent node notes/system message |
| Workflow canvas overview includes setup reminders for Slack, OpenAI, and updating Authorization headers in all Tool nodes. | From “Overview: Recrutei ATS - Ultimate Internal AI Agent” sticky note |