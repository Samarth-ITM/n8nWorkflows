Identify buying-intent leads on Twitter and Instagram with Slack and Notion CRM

https://n8nworkflows.xyz/workflows/identify-buying-intent-leads-on-twitter-and-instagram-with-slack-and-notion-crm-12600


# Identify buying-intent leads on Twitter and Instagram with Slack and Notion CRM

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow provides **on-demand lead discovery** from social platforms (Twitter/X and Instagram) through a **chat-based input**. It uses an **AI agent** to search for posts that show **genuine buying intent** (requests for tools, recommendations, problem complaints), filters out spam/promotions, and classifies intent (Low/Medium/High). It then outputs:
- A **human-readable Slack summary** for quick review
- A **structured Notion CRM record** for tracking

It also keeps **short-term conversation memory** to improve relevance across follow-up queries, and includes **error handling** that emails an alert when the workflow fails.

**Important deployment constraint:** The design relies on an **external MCP tool** and custom AI orchestration; it is intended for **self-hosted n8n**, not n8n Cloud.

### 1.1 Input Reception & Context
Receives a user’s lead discovery request via Chat Trigger and maintains short-term memory for follow-ups.

### 1.2 AI Lead Discovery & Qualification (with External Search Tool)
An AI agent orchestrates discovery, calls an external MCP client tool to fetch social posts, and uses an Azure OpenAI chat model to reason about and qualify leads.

### 1.3 Response Preparation
Converts the agent output into a clean response payload that downstream steps can consume.

### 1.4 Insight Structuring, Validation, and Dual Output (Slack + Notion)
A second AI agent transforms results into a strict JSON object (Slack summary + Notion record), validated by a structured output parser, then sends Slack message and creates a Notion database page.

### 1.5 Error Handling
If any node fails, an Error Trigger fires and sends an email alert via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Context

**Overview:**  
Accepts a user’s natural-language lead discovery query via chat, and retains recent context (last N messages) to improve relevance for follow-up requests.

**Nodes Involved:**
- Receive User Lead Discovery Query (Chat Trigger)
- Maintain Short-Term Conversation Context

#### Node: Receive User Lead Discovery Query (Chat Trigger)
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point for chat-based executions.
- **Key configuration:**
  - Response mode: **responseNodes** (the workflow is expected to respond via later nodes rather than directly in the trigger).
  - **Disabled:** `true` in the workflow JSON, meaning it will not execute until manually enabled.
- **Data produced:** Typically includes a `chatInput` field used later by other nodes.
- **Connections:**
  - **Output →** Discover Buying-Intent Leads from Social Platforms (AI) (main).
- **Version-specific notes:** TypeVersion **1.3**.
- **Edge cases / failure modes:**
  - Disabled trigger prevents any normal executions.
  - If chat payload does not include expected fields (like `chatInput`), downstream expressions referencing it may fail.

#### Node: Maintain Short-Term Conversation Context
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Provides short-term conversation memory for AI steps.
- **Key configuration:**
  - Context window length: **7** (keeps the last 7 exchanges/turns).
- **Connections:**
  - **AI memory output →** Discover Buying-Intent Leads from Social Platforms (AI) (ai_memory).
- **Version-specific notes:** TypeVersion **1.3**.
- **Edge cases / failure modes:**
  - Memory may be empty on first run; AI agent should still function.
  - If running in stateless environments without persistence assumptions, memory only applies within supported chat/memory context.

---

### Block 2 — AI Lead Discovery & Qualification (with External Search Tool)

**Overview:**  
An AI agent receives the user query + memory, uses an external MCP tool to fetch relevant social posts, then filters/qualifies leads with an Azure OpenAI model.

**Nodes Involved:**
- Discover Buying-Intent Leads from Social Platforms (AI)
- External Social Search & Enrichment Tool (MCP Client)
- LLM Reasoning Engine for Lead Qualification

#### Node: Discover Buying-Intent Leads from Social Platforms (AI)
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates multi-step reasoning and tool usage.
- **Key configuration choices:**
  - Max iterations: **30** (allows the agent to iterate tool calls / reasoning steps).
  - System message defines the lead-gen task:
    - Search Twitter and Instagram for requests for tools/recommendations or problem complaints
    - Extract only **real buying-intent** posts
    - Output per lead: platform, username, post text, problem summary, buying intent (Low/Medium/High)
    - Ignore spam/ads/promotions
    - If none: return “No qualified leads found”
    - Output should be structured and bullet-pointed (note: later steps attempt to convert output to JSON).
- **Inputs:**
  - From Chat Trigger (main)
  - From Memory Buffer (ai_memory)
  - From Azure OpenAI model node (ai_languageModel)
  - From MCP Tool node (ai_tool)
- **Outputs:**
  - **Main output →** Generate CRM-Ready Lead Discovery Response
- **Version-specific notes:** TypeVersion **3**.
- **Edge cases / failure modes:**
  - Tool unavailability or MCP auth failure will reduce or block discovery.
  - The agent may return unstructured text; downstream steps assume a field like `$json.output` exists (depending on how the Agent node formats results in your n8n version).
  - Iteration limit could be hit if the agent loops; results may be partial.

#### Node: External Social Search & Enrichment Tool (MCP Client)
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — External tool the agent can call to fetch/enrich social posts.
- **Key configuration:**
  - Endpoint: `https://mcp.xpoz.ai/mcp`
  - Authentication: **Bearer Auth** via `httpBearerAuth` credential (“saurabh xpoz”)
- **Connections:**
  - **Tool output →** Discover Buying-Intent Leads from Social Platforms (AI) (ai_tool)
- **Version-specific notes:** TypeVersion **1.2**.
- **Edge cases / failure modes:**
  - Network/timeouts, endpoint changes, rate limits.
  - Bearer token expiry/invalid token.
  - Returned schema may not match what the agent expects (causing poor extraction).

#### Node: LLM Reasoning Engine for Lead Qualification
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` — Azure OpenAI chat model used by the discovery agent.
- **Key configuration:**
  - Model: **gpt-4o-mini**
  - Credentials: Azure OpenAI (“Azure Open AI account”)
- **Connections:**
  - **Language model →** Discover Buying-Intent Leads from Social Platforms (AI) (ai_languageModel)
- **Version-specific notes:** TypeVersion **1**.
- **Edge cases / failure modes:**
  - Azure deployment/model name mismatch with your Azure configuration.
  - Quota/rate limit issues and content-length limits (long social payloads).
  - Latency/timeouts for large searches.

---

### Block 3 — Response Preparation

**Overview:**  
Takes the discovery agent’s output and formats it into a response message payload for downstream steps.

**Nodes Involved:**
- Generate CRM-Ready Lead Discovery Response

#### Node: Generate CRM-Ready Lead Discovery Response
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — Chat response node used as a “response node” in chat-trigger mode.
- **Key configuration:**
  - Message: `={{ $json.output }}`
  - Wait for user reply: **false**
- **Connections:**
  - **Input ←** Discover Buying-Intent Leads from Social Platforms (AI)
  - **Output →** Generate Slack & Notion Lead Insight Summary (AI)
- **Version-specific notes:** TypeVersion **1**.
- **Key expression risks:**
  - If the agent output isn’t at `$json.output` (could be `$json.text`, `$json.result`, etc. depending on node/version), the expression resolves to empty/throws.
- **Edge cases / failure modes:**
  - Missing `$json.output` breaks the response content and may cause downstream AI summary to operate on incomplete data.

---

### Block 4 — Insight Structuring & Validation + Distribution (Slack & Notion)

**Overview:**  
A second AI agent converts qualified leads into a strictly validated JSON object containing both a Slack summary and a Notion-ready record. The workflow then posts the Slack message and creates a Notion database page.

**Nodes Involved:**
- Generate Slack & Notion Lead Insight Summary (AI)
- LLM Reasoning Engine for Insight Structuring
- Enforce Structured Lead Insight Output Schema
- Send Lead Discovery Summary to Slack
- Store Lead Discovery Insight in Notion CRM

#### Node: Generate Slack & Notion Lead Insight Summary (AI)
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Produces two outputs (Slack + Notion) in a single structured JSON response.
- **Key configuration:**
  - **Input text** (constructed with expressions):
    - User search query pulled from the Chat Trigger:  
      `{{ $('Receive User Lead Discovery Query (Chat Trigger)').item.json.chatInput }}`
    - Lead list built from either `$json.leads` or all input items:  
      `{{ JSON.stringify($json.leads || $input.all().map(i => i.json), null, 2) }}`
  - System message enforces:
    - Output must be **valid JSON only**
    - No markdown, no explanations
    - Slack summary must include: what searched, count, intent insights
    - Notion text concise, structured; **no emojis**; **no markdown tables**
    - Missing data → empty strings/zero values
    - Must follow schema “exactly” (paired with output parser below)
  - `hasOutputParser: true` indicates the structured output parser is attached.
- **Connections:**
  - **Input ←** Generate CRM-Ready Lead Discovery Response
  - **Language model input ←** LLM Reasoning Engine for Insight Structuring (ai_languageModel)
  - **Output parser input ←** Enforce Structured Lead Insight Output Schema (ai_outputParser)
  - **Outputs →**
    - Send Lead Discovery Summary to Slack
    - Store Lead Discovery Insight in Notion CRM
- **Version-specific notes:** TypeVersion **2**.
- **Edge cases / failure modes:**
  - Expression dependency on the disabled Chat Trigger node: if you replace the trigger, update the node reference in the expression.
  - If the upstream doesn’t provide `leads` and `$input.all()` is not in the expected shape, the lead list may be noisy or huge.
  - If model outputs invalid JSON, the output parser will fail (and the workflow will error).

#### Node: LLM Reasoning Engine for Insight Structuring
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` — Azure OpenAI model for structured summarization.
- **Key configuration:**
  - Model: **gpt-4o-mini**
  - Credentials: Azure OpenAI
- **Connections:**
  - **Language model →** Generate Slack & Notion Lead Insight Summary (AI) (ai_languageModel)
- **Version-specific notes:** TypeVersion **1**.
- **Edge cases / failure modes:** Same as the other Azure model node (quota, deployment mismatch, large context).

#### Node: Enforce Structured Lead Insight Output Schema
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Validates/coerces the AI output to match a specified JSON schema example.
- **Key configuration:**
  - Schema example includes:
    - `query_context` (user_query, source_platforms, execution_time)
    - `results_overview` counts (total/high/medium/low)
    - `slack_summary` (title, summary, key_insight, recommended_action)
    - `notion_record` (user_query, total_items_found, primary_themes, overall_intent, status)
- **Connections:**
  - **Output parser →** Generate Slack & Notion Lead Insight Summary (AI) (ai_outputParser)
- **Version-specific notes:** TypeVersion **1.3**.
- **Edge cases / failure modes:**
  - If the agent returns fields that don’t match, parsing/validation may fail.
  - Ambiguity: schema example is not a strict JSON Schema draft, but n8n’s structured parser uses it as a template—still can fail on malformed JSON.

#### Node: Send Lead Discovery Summary to Slack
- **Type / role:** `n8n-nodes-base.slack` — Sends the final Slack message.
- **Key configuration:**
  - Channel: selected by ID `C09GNB90TED` (cached name “general-information”)
  - Message text uses structured output fields:
    - `{{$json.output.slack_summary.title}}`
    - `{{$json.output.slack_summary.summary}}`
    - `{{$json.output.slack_summary.key_insight}}`
    - `{{$json.output.slack_summary.recommended_action}}`
- **Connections:**
  - **Input ←** Generate Slack & Notion Lead Insight Summary (AI)
- **Version-specific notes:** TypeVersion **2.1**.
- **Edge cases / failure modes:**
  - Slack credential revoked/expired.
  - Bot/user missing permission to post in the channel.
  - If `output.slack_summary.*` is missing due to parser/agent mismatch, message will be incomplete or fail.

#### Node: Store Lead Discovery Insight in Notion CRM
- **Type / role:** `n8n-nodes-base.notion` — Creates a page in a Notion database (CRM logging).
- **Key configuration:**
  - Resource: **databasePage**
  - Database: `2d8802b9-1fa0-804a-be3c-d4b876c0f228` (cached name “lead gen tool db”)
  - Properties mapping:
    - `search_query|title` ← `{{$json.output.notion_record.user_query}}`
    - `overall_intent|rich_text` ← `{{$json.output.notion_record.overall_intent}}`
    - `primary_themes|rich_text` ← `{{$json.output.notion_record.primary_themes.join(", ")}}`
- **Connections:**
  - **Input ←** Generate Slack & Notion Lead Insight Summary (AI)
- **Version-specific notes:** TypeVersion **2.2**.
- **Edge cases / failure modes:**
  - Notion database property keys must match exactly (`search_query`, `overall_intent`, `primary_themes`). If the DB schema differs, creation fails.
  - `primary_themes` must be an array; if empty or not an array, `.join()` may error (unless coerced by the parser/agent).

---

### Block 5 — Error Handling

**Overview:**  
Captures workflow execution errors and sends an email alert with node name, error message, and timestamp.

**Nodes Involved:**
- Workflow Error Handler
- Send a message1

#### Node: Workflow Error Handler
- **Type / role:** `n8n-nodes-base.errorTrigger` — Triggers when the workflow errors.
- **Connections:**
  - **Output →** Send a message1
- **Version-specific notes:** TypeVersion **1**.
- **Edge cases / failure modes:**
  - Only triggers on workflow errors; it does not handle “soft failures” where nodes return empty results.

#### Node: Send a message1
- **Type / role:** `n8n-nodes-base.gmail` — Sends an error notification email.
- **Key configuration:**
  - To: `user@example.com` (should be replaced)
  - Subject: “Workflow Error Alert”
  - Message body contains:
    - Error node: `{{ $json.node.name }}`
    - Error message: `{{ $json.error.message }}`
    - Timestamp: `{{ $now.toISO() }}`
  - Email type: text
- **Connections:**
  - **Input ←** Workflow Error Handler
- **Version-specific notes:** TypeVersion **2.2**.
- **Edge cases / failure modes:**
  - Gmail OAuth2 credential expired / requires re-consent.
  - Sending limits / spam policies if many failures occur.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive User Lead Discovery Query (Chat Trigger) | @n8n/n8n-nodes-langchain.chatTrigger | Chat-based entry point | — | Discover Buying-Intent Leads from Social Platforms (AI) | ## User Input & Context; Receives user lead queries and maintains short-term conversation memory |
| Maintain Short-Term Conversation Context | @n8n/n8n-nodes-langchain.memoryBufferWindow | Short-term memory for AI | — (runtime memory) | Discover Buying-Intent Leads from Social Platforms (AI) (ai_memory) | ## User Input & Context; Receives user lead queries and maintains short-term conversation memory |
| Discover Buying-Intent Leads from Social Platforms (AI) | @n8n/n8n-nodes-langchain.agent | Agentic discovery + lead qualification orchestration | Receive User Lead Discovery Query (Chat Trigger); Maintain Short-Term Conversation Context; External Social Search & Enrichment Tool (MCP Client); LLM Reasoning Engine for Lead Qualification | Generate CRM-Ready Lead Discovery Response | ## AI Lead Discovery & Qualification; Identifies real buying-intent leads and classifies intent |
| External Social Search & Enrichment Tool (MCP Client) | @n8n/n8n-nodes-langchain.mcpClientTool | External tool for social search | — | Discover Buying-Intent Leads from Social Platforms (AI) (ai_tool) | ## Social Search & Enrichment; Fetches and enriches social posts from external platforms |
| LLM Reasoning Engine for Lead Qualification | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI model for discovery agent | — | Discover Buying-Intent Leads from Social Platforms (AI) (ai_languageModel) | ## AI Lead Discovery & Qualification; Identifies real buying-intent leads and classifies intent |
| Generate CRM-Ready Lead Discovery Response | @n8n/n8n-nodes-langchain.chat | Chat response formatting / handoff | Discover Buying-Intent Leads from Social Platforms (AI) | Generate Slack & Notion Lead Insight Summary (AI) | ## CRM-Ready Response Preparation; Converts discovered leads into a clean, user-facing response |
| Generate Slack & Notion Lead Insight Summary (AI) | @n8n/n8n-nodes-langchain.agent | Produces validated JSON: Slack summary + Notion record | Generate CRM-Ready Lead Discovery Response; LLM Reasoning Engine for Insight Structuring; Enforce Structured Lead Insight Output Schema | Send Lead Discovery Summary to Slack; Store Lead Discovery Insight in Notion CRM | ## Insight Structuring & Validation; Transforms raw leads into structured, validated insight |
| LLM Reasoning Engine for Insight Structuring | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI model for summary/structuring | — | Generate Slack & Notion Lead Insight Summary (AI) (ai_languageModel) | ## Insight Structuring & Validation; Transforms raw leads into structured, validated insight |
| Enforce Structured Lead Insight Output Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces output JSON shape | — | Generate Slack & Notion Lead Insight Summary (AI) (ai_outputParser) | ## Insight Structuring & Validation; Transforms raw leads into structured, validated insight |
| Send Lead Discovery Summary to Slack | n8n-nodes-base.slack | Sends Slack summary | Generate Slack & Notion Lead Insight Summary (AI) | — | ## Distribution & CRM Storage; Delivers insights to Slack and stores them in Notion |
| Store Lead Discovery Insight in Notion CRM | n8n-nodes-base.notion | Stores insight in Notion database | Generate Slack & Notion Lead Insight Summary (AI) | — | ## Distribution & CRM Storage; Delivers insights to Slack and stores them in Notion |
| Workflow Error Handler | n8n-nodes-base.errorTrigger | Global error entry point | — | Send a message1 | ## Error Handling; Sends alerts when the workflow fails |
| Send a message1 | n8n-nodes-base.gmail | Emails error alert | Workflow Error Handler | — | ## Error Handling; Sends alerts when the workflow fails |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | ## 🔎 Identify Buying-Intent Leads on Twitter & Instagram using GPT-4o-mini with Slack Alerts and Notion CRM; Setup/customization notes (applies to workflow overall) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation | — | — | ## User Input & Context; Receives user lead queries and maintains short-term conversation memory |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Social Search & Enrichment; Fetches and enriches social posts from external platforms |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Distribution & CRM Storage; Delivers insights to Slack and stores them in Notion |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Insight Structuring & Validation; Transforms raw leads into structured, validated insight |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation | — | — | ## AI Lead Discovery & Qualification; Identifies real buying-intent leads and classifies intent |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation | — | — | ## CRM-Ready Response Preparation; Converts discovered leads into a clean, user-facing response |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Error Handling; Sends alerts when the workflow fails |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation | — | — | ## 🎥 Workflow Demo Video; @[youtube](pj4GfvdUa6k) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your **self-hosted n8n** instance.
2. **Add node:** *Chat Trigger* (`@n8n/n8n-nodes-langchain.chatTrigger`)
   - Set **Response Mode** to **Response Nodes**.
   - Enable the node (in the provided JSON it is disabled; you’ll likely want it enabled).
3. **Add node:** *Memory Buffer Window* (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)
   - Set **Context Window Length = 7**.
4. **Add node:** *Azure OpenAI Chat Model* (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`) named “LLM Reasoning Engine for Lead Qualification”
   - Model: **gpt-4o-mini**
   - Configure **Azure OpenAI credentials**:
     - Azure OpenAI Resource/Endpoint
     - API key
     - Deployment/model mapping as required by your Azure setup
5. **Add node:** *MCP Client Tool* (`@n8n/n8n-nodes-langchain.mcpClientTool`)
   - Endpoint URL: `https://mcp.xpoz.ai/mcp`
   - Authentication: **Bearer Auth**
   - Create/configure **HTTP Bearer Auth credential** (token provided by your MCP provider).
6. **Add node:** *AI Agent* (`@n8n/n8n-nodes-langchain.agent`) named “Discover Buying-Intent Leads from Social Platforms (AI)”
   - Set **Max Iterations = 30**
   - System message: instruct it to search Twitter/Instagram, filter spam, extract structured lead fields, and return “No qualified leads found” when applicable (use the same intent rules as in the workflow).
7. **Wire the discovery block connections:**
   - Chat Trigger **main →** Discovery Agent **main**
   - Memory Buffer **ai_memory →** Discovery Agent **ai_memory**
   - Azure OpenAI (qualification) **ai_languageModel →** Discovery Agent **ai_languageModel**
   - MCP Client Tool **ai_tool →** Discovery Agent **ai_tool**
8. **Add node:** *Chat* (`@n8n/n8n-nodes-langchain.chat`) named “Generate CRM-Ready Lead Discovery Response”
   - Message: `{{ $json.output }}` (adjust if your Agent node outputs a different field in your n8n version)
   - Wait for user reply: **false**
9. **Connect:** Discovery Agent **main →** Generate CRM-Ready Lead Discovery Response **main**
10. **Add node:** *Azure OpenAI Chat Model* (`@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`) named “LLM Reasoning Engine for Insight Structuring”
    - Model: **gpt-4o-mini**
    - Use same (or separate) Azure OpenAI credentials.
11. **Add node:** *Structured Output Parser* (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - Paste the schema example for:
      - `query_context`, `results_overview`, `slack_summary`, `notion_record`
12. **Add node:** *AI Agent* (`@n8n/n8n-nodes-langchain.agent`) named “Generate Slack & Notion Lead Insight Summary (AI)”
    - Text prompt should include:
      - User query from Chat Trigger (reference your trigger node):  
        `{{ $('Receive User Lead Discovery Query (Chat Trigger)').item.json.chatInput }}`
      - Leads JSON (use either a known `leads` array or stringify all input items):  
        `{{ JSON.stringify($json.leads || $input.all().map(i => i.json), null, 2) }}`
    - System message: require **JSON-only output**, Slack summary requirements, Notion constraints (no emojis, no markdown tables), and strict schema compliance.
    - Ensure **Output Parser** is enabled/attached (the node should use the structured parser).
13. **Wire the structuring connections:**
    - Generate CRM-Ready Lead Discovery Response **main →** Generate Slack & Notion Lead Insight Summary (AI) **main**
    - Insight Structuring Azure OpenAI **ai_languageModel →** Summary Agent **ai_languageModel**
    - Structured Output Parser **ai_outputParser →** Summary Agent **ai_outputParser**
14. **Add node:** *Slack* (`n8n-nodes-base.slack`) named “Send Lead Discovery Summary to Slack”
    - Choose operation that sends a message to a channel.
    - Channel: select your target channel.
    - Message text mapping (from structured output):
      - `{{$json.output.slack_summary.title}}`, `summary`, `key_insight`, `recommended_action`
    - Configure **Slack API credential** (OAuth token / bot token with chat:write, channel access).
15. **Add node:** *Notion* (`n8n-nodes-base.notion`) named “Store Lead Discovery Insight in Notion CRM”
    - Resource: **Database Page**
    - Select your database ID.
    - Map properties:
      - Title property (e.g., `search_query`) ← `{{$json.output.notion_record.user_query}}`
      - Rich text properties for intent and themes
    - Configure **Notion API credential** and ensure the integration has access to the database.
16. **Connect outputs:**
    - Summary Agent **main →** Slack node **main**
    - Summary Agent **main →** Notion node **main**
17. **Add error handling:**
    - Add node: *Error Trigger* (`n8n-nodes-base.errorTrigger`)
    - Add node: *Gmail* (`n8n-nodes-base.gmail`) set to “send email”
      - To: your alert email
      - Subject: “Workflow Error Alert”
      - Body referencing `{{$json.node.name}}`, `{{$json.error.message}}`, `{{$now.toISO()}}`
      - Configure **Gmail OAuth2** credential.
    - Connect: Error Trigger **main →** Gmail **main**
18. **Validate with a test query** (e.g., “Find people asking for a lightweight CRM for freelancers”) and confirm:
    - MCP tool returns results
    - Summary agent produces valid JSON matching the parser
    - Slack message posts
    - Notion page is created with correct properties

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This template is intended for **self-hosted n8n** due to reliance on external MCP tools and custom AI orchestration not supported on n8n Cloud. | Deployment constraint (from provided description) |
| Demo video reference: `@[youtube](pj4GfvdUa6k)` | From “🎥 Workflow Demo Video” sticky note |
| Workflow behavior summary: chat-based lead discovery, MCP social retrieval, AI spam filtering + intent classification, Slack + Notion outputs, short-term memory, error email alerts. | From main workflow sticky note content |
| Customization suggestions: adjust intent rules in AI prompt; modify output fields to match your CRM schema; extend to more platforms via MCP tools. | From main workflow sticky note content and description |

