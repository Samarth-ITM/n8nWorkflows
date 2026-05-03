Discover buying-intent leads on Twitter and Instagram with GPT-4o-mini and send summaries to Slack and Notion

https://n8nworkflows.xyz/workflows/discover-buying-intent-leads-on-twitter-and-instagram-with-gpt-4o-mini-and-send-summaries-to-slack-and-notion-12110


# Discover buying-intent leads on Twitter and Instagram with GPT-4o-mini and send summaries to Slack and Notion

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow discovers **buying-intent leads** on **Twitter and Instagram** from a user’s natural-language query, qualifies and structures the results with **GPT-4o-mini (Azure OpenAI)**, then distributes an executive summary to **Slack** and stores a concise record in **Notion (CRM database)**. It also includes an **error alert path via Gmail**.

**Target use cases:**
- Growth/sales teams monitoring social platforms for “looking for tool / recommendation / help” signals.
- On-demand, chat-driven lead discovery that produces both human-readable and CRM-ready outputs.
- Lightweight “social listening” without building custom scrapers, using an external MCP tool.

### 1.1 Logical Blocks
1. **User Input & Context**: Chat trigger receives the query; short-term memory is maintained for follow-ups.  
2. **AI Lead Discovery & Qualification**: An AI agent uses an external MCP social search tool and GPT-4o-mini to filter and classify buying-intent leads.  
3. **CRM-Ready Response Preparation**: Converts the agent output into a clean response payload.  
4. **Insight Structuring & Validation**: A second AI agent produces strict JSON with Slack + Notion fields; schema is enforced.  
5. **Distribution & CRM Storage**: Sends message to Slack and creates a Notion database page.  
6. **Error Handling**: Any workflow error triggers an email alert via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — User Input & Context
**Overview:**  
Accepts a user’s lead discovery query via chat and maintains short-term conversational context to improve follow-up searches.

**Nodes involved:**
- Receive User Lead Discovery Query (Chat Trigger)
- Maintain Short-Term Conversation Context

#### Node: Receive User Lead Discovery Query (Chat Trigger)
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point (chat/webhook-style trigger).
- **Configuration choices:**
  - **Response mode:** `responseNodes` (responses are expected to be produced by downstream nodes rather than directly by the trigger).
  - **Disabled:** **true** (workflow won’t start from this trigger until enabled).
- **Key variables/expressions:**
  - User input later referenced as:  
    `$('Receive User Lead Discovery Query (Chat Trigger)').item.json.chatInput`
- **Connections:**
  - **Output:** to **Discover Buying-Intent Leads from Social Platforms (AI)** (main).
- **Edge cases / failures:**
  - If disabled, workflow won’t accept chat inputs.
  - Missing/empty `chatInput` may cause low-quality or null results in AI prompts.

#### Node: Maintain Short-Term Conversation Context
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term memory for an agent.
- **Configuration choices:**
  - **contextWindowLength:** `7` (keeps last 7 turns/messages).
- **Connections:**
  - **ai_memory output:** to **Discover Buying-Intent Leads from Social Platforms (AI)**.
- **Edge cases / failures:**
  - If the chat trigger is disabled or not used, memory may not accumulate meaningful context.
  - Context window may include irrelevant earlier turns; adjust window length if drift occurs.

**Sticky note context (applies to this block):**  
“## User Input & Context — Receives user lead queries and maintains short-term conversation memory”

---

### Block 2 — AI Lead Discovery & Qualification
**Overview:**  
An AI agent searches social platforms via an external MCP tool and uses GPT-4o-mini to filter to genuine buying-intent posts and classify intent.

**Nodes involved:**
- Discover Buying-Intent Leads from Social Platforms (AI)
- External Social Search & Enrichment Tool (MCP Client)
- LLM Reasoning Engine for Lead Qualification

#### Node: Discover Buying-Intent Leads from Social Platforms (AI)
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool use + LLM reasoning to produce a structured lead list.
- **Configuration choices:**
  - **Max iterations:** `30` (agent can loop through tool calls and reasoning up to 30 steps).
  - **System message:** instructs the agent to:
    - Search Twitter/Instagram for posts asking for tools/recommendations/help or complaining about problems
    - Extract **only real buying-intent posts**
    - Return per-lead fields: platform, username, post text, problem summary, buying intent (Low/Medium/High)
    - Ignore spam/ads/promotions
    - If none found: output **“No qualified leads found”**
    - Output must be **clean, structured bullet points** suitable for CRM
- **Connections:**
  - **Input:** from Chat Trigger (main) and Memory (ai_memory).
  - **Tooling:** uses **External Social Search & Enrichment Tool (MCP Client)** via `ai_tool`.
  - **Language model:** uses **LLM Reasoning Engine for Lead Qualification** via `ai_languageModel`.
  - **Output:** to **Generate CRM-Ready Lead Discovery Response**.
- **Edge cases / failures:**
  - MCP endpoint may return empty results, rate limits, or authentication failures.
  - The agent is instructed to output bullet points, which may be less machine-parseable later; ensure downstream steps can handle plain text.
  - Long queries or too-broad queries can cause iteration overuse or timeouts.

#### Node: External Social Search & Enrichment Tool (MCP Client)
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — external tool connector for social search/enrichment.
- **Configuration choices:**
  - **Endpoint URL:** `https://mcp.xpoz.ai/mcp`
  - **Auth:** Bearer token (`bearerAuth`)
  - **Credentials:** HTTP Bearer Auth credential named “saurabh xpoz”
- **Connections:**
  - **ai_tool output:** to **Discover Buying-Intent Leads from Social Platforms (AI)**.
- **Edge cases / failures:**
  - 401/403 if bearer token invalid/expired.
  - Network timeouts, service downtime, or schema changes in tool responses.
  - Data quality issues (spammy results) must be handled by agent prompt rules.

#### Node: LLM Reasoning Engine for Lead Qualification
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` — Azure OpenAI chat model backing the discovery agent.
- **Configuration choices:**
  - **Model:** `gpt-4o-mini`
  - **Credentials:** “Azure Open AI account”
- **Connections:**
  - **ai_languageModel:** to **Discover Buying-Intent Leads from Social Platforms (AI)**.
- **Version-specific notes:**
  - Node is `typeVersion: 1`; Azure OpenAI resource/deployment settings must be correctly configured in credentials.
- **Edge cases / failures:**
  - Azure OpenAI quota/rate limits, deployment misconfiguration, or content filter responses.
  - Prompt-following variance: agent may still output less structured text than desired.

**Sticky note context (applies to this block):**  
“## AI Lead Discovery & Qualification — Identifies real buying-intent leads and classifies intent”  
“## Social Search & Enrichment — Fetches and enriches social posts from external platforms”

---

### Block 3 — CRM-Ready Response Preparation
**Overview:**  
Takes the agent’s output and produces a clean response message suitable for returning to the chat interface and for downstream processing.

**Nodes involved:**
- Generate CRM-Ready Lead Discovery Response

#### Node: Generate CRM-Ready Lead Discovery Response
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — formats a response message (non-interactive).
- **Configuration choices:**
  - **Message:** `={{ $json.output }}` (passes through the prior node’s `output`)
  - **waitUserReply:** `false`
- **Connections:**
  - **Input:** from **Discover Buying-Intent Leads from Social Platforms (AI)**
  - **Output:** to **Generate Slack & Notion Lead Insight Summary (AI)**
- **Edge cases / failures:**
  - If upstream doesn’t return `output`, this expression may resolve empty and downstream summarization will degrade.

**Sticky note context:**  
“## CRM-Ready Response Preparation — Converts discovered leads into a clean, user-facing response”

---

### Block 4 — Insight Structuring & Validation
**Overview:**  
A second AI agent converts discovered leads into **strict JSON** containing both a Slack-ready summary and a Notion-ready record, enforced by a schema parser.

**Nodes involved:**
- Generate Slack & Notion Lead Insight Summary (AI)
- LLM Reasoning Engine for Insight Structuring
- Enforce Structured Lead Insight Output Schema

#### Node: Generate Slack & Notion Lead Insight Summary (AI)
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — transforms lead results into validated JSON for Slack + Notion.
- **Configuration choices:**
  - **Input text (composed prompt):**
    - Includes the original user query:  
      `{{ $('Receive User Lead Discovery Query (Chat Trigger)').item.json.chatInput }}`
    - Includes “qualified leads” as JSON:  
      `{{ JSON.stringify($json.leads || $input.all().map(i => i.json), null, 2) }}`
      - This attempts to use `$json.leads` if present; otherwise it serializes all incoming items.
  - **System message constraints (critical):**
    - Must produce **TWO outputs**: Slack summary + Notion-ready entry
    - **Notion output:** no emojis, concise, structured, no markdown tables
    - **Output must be valid JSON only**, no markdown, no explanation
    - Use neutral language; if missing data, use empty strings/zero values
    - “Follow the output schema exactly”
  - **Output parser enabled:** `hasOutputParser: true`
- **Connections:**
  - **Input:** from **Generate CRM-Ready Lead Discovery Response**
  - **Language model:** from **LLM Reasoning Engine for Insight Structuring** (ai_languageModel)
  - **Schema parser:** from **Enforce Structured Lead Insight Output Schema** (ai_outputParser)
  - **Output:** to both Slack and Notion nodes (fan-out)
- **Edge cases / failures:**
  - If the Chat Trigger is disabled, the expression referencing it may fail or be empty depending on execution context.
  - If the upstream format is plain bullet text (not structured leads), `$json.leads` likely won’t exist; fallback `$input.all()` may include unexpected shapes.
  - Strict JSON requirement: model may occasionally emit invalid JSON; the structured output parser should catch it and error.

#### Node: LLM Reasoning Engine for Insight Structuring
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` — Azure OpenAI model for summarization/structuring.
- **Configuration choices:**
  - **Model:** `gpt-4o-mini`
  - **Credentials:** “Azure Open AI account”
- **Connections:**
  - **ai_languageModel:** to **Generate Slack & Notion Lead Insight Summary (AI)**
- **Edge cases / failures:**
  - Same Azure concerns: quota, deployment config, rate limits.

#### Node: Enforce Structured Lead Insight Output Schema
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates/forces model output to match a JSON schema example.
- **Configuration choices:**
  - **Schema example includes:**
    - `query_context`: user_query, source_platforms[], execution_time
    - `results_overview`: totals + intent counts
    - `slack_summary`: title, summary, key_insight, recommended_action
    - `notion_record`: user_query, total_items_found, primary_themes[], overall_intent (“High | Medium | Low | Mixed”), status (“New”)
- **Connections:**
  - **ai_outputParser:** to **Generate Slack & Notion Lead Insight Summary (AI)**
- **Edge cases / failures:**
  - If model output deviates (extra keys, missing keys, wrong types), parser can error and stop the workflow.

**Sticky note context:**  
“## Insight Structuring & Validation — Transforms raw leads into structured, validated insight”

---

### Block 5 — Distribution & CRM Storage
**Overview:**  
Sends the structured Slack summary to a chosen Slack channel and stores a compact record into a Notion database.

**Nodes involved:**
- Send Lead Discovery Summary to Slack
- Store Lead Discovery Insight in Notion CRM

#### Node: Send Lead Discovery Summary to Slack
- **Type / role:** `n8n-nodes-base.slack` — posts a message to Slack channel.
- **Configuration choices:**
  - **Channel:** selected by ID `C09GNB90TED` (cached name: `general-information`)
  - **Message text expression:**
    - Uses structured output:
      - `{{ $json.output.slack_summary.title }}`
      - `{{ $json.output.slack_summary.summary }}`
      - `{{ $json.output.slack_summary.key_insight }}`
      - `{{ $json.output.slack_summary.recommended_action }}`
- **Connections:**
  - **Input:** from **Generate Slack & Notion Lead Insight Summary (AI)**
- **Credentials:** Slack API credential “Slack account vivek”
- **Edge cases / failures:**
  - Missing `output.slack_summary.*` keys (if schema enforcement fails upstream).
  - Slack auth revoked, channel not found, bot not in channel, or rate limits.

#### Node: Store Lead Discovery Insight in Notion CRM
- **Type / role:** `n8n-nodes-base.notion` — creates a database page in Notion.
- **Configuration choices:**
  - **Resource:** `databasePage`
  - **Database:** ID `2d8802b9-1fa0-804a-be3c-d4b876c0f228` (cached name “lead gen tool db”), URL: https://www.notion.so/2d8802b91fa0804abe3cd4b876c0f228
  - **Properties mapped:**
    - `search_query|title` = `{{ $json.output.notion_record.user_query }}`
    - `overall_intent|rich_text` = `{{ $json.output.notion_record.overall_intent }}`
    - `primary_themes|rich_text` = `{{ $json.output.notion_record.primary_themes.join(", ") }}`
  - **Status:** schema indicates `"status": "New"` (but Notion node does not map status field here; only stored if you add a property mapping).
- **Connections:**
  - **Input:** from **Generate Slack & Notion Lead Insight Summary (AI)**
- **Credentials:** Notion API credential “Notion account 2”
- **Edge cases / failures:**
  - Database property keys must match exactly (Notion schema changes will break inserts).
  - If `primary_themes` is missing or not an array, `.join()` will fail.
  - Notion integration lacks access to the database, or rate limits.

**Sticky note context:**  
“## Distribution & CRM Storage — Delivers insights to Slack and stores them in Notion”

---

### Block 6 — Error Handling
**Overview:**  
If any node errors during execution, an error trigger fires and sends an email notification.

**Nodes involved:**
- Workflow Error Handler
- Send a message1 (Gmail)

#### Node: Workflow Error Handler
- **Type / role:** `n8n-nodes-base.errorTrigger` — workflow-level error entry point.
- **Configuration choices:** none (default).
- **Connections:**
  - **Output:** to **Send a message1**.
- **Edge cases / failures:**
  - Only catches errors during workflow execution; doesn’t prevent failures.

#### Node: Send a message1
- **Type / role:** `n8n-nodes-base.gmail` — sends an email alert.
- **Configuration choices:**
  - **To:** `user@example.com` (placeholder; must be replaced)
  - **Subject:** “Workflow Error Alert”
  - **Body (text):** includes:
    - Error node: `{{ $json.node.name }}`
    - Error message: `{{ $json.error.message }}`
    - Timestamp: `{{ $now.toISO() }}`
  - **Note:** Message includes a siren emoji; that’s fine for email.
- **Connections:**
  - **Input:** from **Workflow Error Handler**
- **Credentials:** Gmail OAuth2 credential “jyothi”
- **Edge cases / failures:**
  - Gmail OAuth expired, restricted scopes, or sending limits.
  - If `$json.error.message` missing, email may be less informative.

**Sticky note context:**  
“## Error Handling — Sends alerts when the workflow fails”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive User Lead Discovery Query (Chat Trigger) | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for user query | — | Discover Buying-Intent Leads from Social Platforms (AI) | ## User Input & Context<br>Receives user lead queries and maintains short-term conversation memory |
| Maintain Short-Term Conversation Context | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores short-term memory for the agent | — (memory is attached via ai_memory) | Discover Buying-Intent Leads from Social Platforms (AI) | ## User Input & Context<br>Receives user lead queries and maintains short-term conversation memory |
| Discover Buying-Intent Leads from Social Platforms (AI) | @n8n/n8n-nodes-langchain.agent | Agent: searches + qualifies buying intent leads | Receive User Lead Discovery Query (Chat Trigger); Maintain Short-Term Conversation Context (ai_memory); LLM Reasoning Engine for Lead Qualification; External Social Search & Enrichment Tool (MCP Client) | Generate CRM-Ready Lead Discovery Response | ## AI Lead Discovery & Qualification<br>Identifies real buying-intent leads and classifies intent |
| External Social Search & Enrichment Tool (MCP Client) | @n8n/n8n-nodes-langchain.mcpClientTool | Tool connector to external social search/enrichment | — | Discover Buying-Intent Leads from Social Platforms (AI) (ai_tool) | ## Social Search & Enrichment<br>Fetches and enriches social posts from external platforms |
| LLM Reasoning Engine for Lead Qualification | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI model for discovery agent | — | Discover Buying-Intent Leads from Social Platforms (AI) (ai_languageModel) | ## AI Lead Discovery & Qualification<br>Identifies real buying-intent leads and classifies intent |
| Generate CRM-Ready Lead Discovery Response | @n8n/n8n-nodes-langchain.chat | Formats/passes agent output as response | Discover Buying-Intent Leads from Social Platforms (AI) | Generate Slack & Notion Lead Insight Summary (AI) | ## CRM-Ready Response Preparation<br>Converts discovered leads into a clean, user-facing response |
| Generate Slack & Notion Lead Insight Summary (AI) | @n8n/n8n-nodes-langchain.agent | Produces strict JSON: Slack + Notion record | Generate CRM-Ready Lead Discovery Response; LLM Reasoning Engine for Insight Structuring; Enforce Structured Lead Insight Output Schema | Send Lead Discovery Summary to Slack; Store Lead Discovery Insight in Notion CRM | ## Insight Structuring & Validation<br>Transforms raw leads into structured, validated insight |
| LLM Reasoning Engine for Insight Structuring | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI model for structuring agent | — | Generate Slack & Notion Lead Insight Summary (AI) (ai_languageModel) | ## Insight Structuring & Validation<br>Transforms raw leads into structured, validated insight |
| Enforce Structured Lead Insight Output Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Validates/enforces JSON schema | — | Generate Slack & Notion Lead Insight Summary (AI) (ai_outputParser) | ## Insight Structuring & Validation<br>Transforms raw leads into structured, validated insight |
| Send Lead Discovery Summary to Slack | n8n-nodes-base.slack | Sends Slack summary to channel | Generate Slack & Notion Lead Insight Summary (AI) | — | ## Distribution & CRM Storage<br>Delivers insights to Slack and stores them in Notion |
| Store Lead Discovery Insight in Notion CRM | n8n-nodes-base.notion | Creates Notion database page (CRM record) | Generate Slack & Notion Lead Insight Summary (AI) | — | ## Distribution & CRM Storage<br>Delivers insights to Slack and stores them in Notion |
| Workflow Error Handler | n8n-nodes-base.errorTrigger | Catches workflow failures | — | Send a message1 | ## Error Handling<br>Sends alerts when the workflow fails |
| Send a message1 | n8n-nodes-base.gmail | Sends email alert on errors | Workflow Error Handler | — | ## Error Handling<br>Sends alerts when the workflow fails |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🔎 Identify Buying-Intent Leads on Twitter & Instagram using GPT-4o-mini with Slack Alerts and Notion CRM (contains setup/customization notes) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## User Input & Context<br>Receives user lead queries and maintains short-term conversation memory |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Social Search & Enrichment<br>Fetches and enriches social posts from external platforms |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Distribution & CRM Storage<br>Delivers insights to Slack and stores them in Notion |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Insight Structuring & Validation<br>Transforms raw leads into structured, validated insight |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## AI Lead Discovery & Qualification<br>Identifies real buying-intent leads and classifies intent |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## CRM-Ready Response Preparation<br>Converts discovered leads into a clean, user-facing response |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Error Handling<br>Sends alerts when the workflow fails |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🎥 Workflow Demo Video<br>https://www.youtube.com/watch?v=pj4GfvdUa6k |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Identify Buying-Intent Leads on Twitter & Instagram using GPT-4o-mini with Slack Alerts and Notion CRM* (or your preferred title).

2. **Add Chat Trigger (entry point)**
   - Add node: **LangChain → Chat Trigger**
   - Set **Response Mode** to **Response Nodes**.
   - Enable the node (in the provided workflow it is disabled).
   - This node will provide `chatInput` in its output JSON.

3. **Add Conversation Memory**
   - Add node: **LangChain → Memory Buffer Window**
   - Set **Context Window Length** = `7`.
   - Connect **Memory Buffer Window** to the agent using the **ai_memory** connection (not main).

4. **Add Azure OpenAI model node for discovery**
   - Add node: **LangChain → Azure OpenAI Chat Model**
   - Select model: **gpt-4o-mini**
   - Configure **Azure OpenAI credentials**:
     - Resource/endpoint, API key, deployment (as required by your n8n Azure OpenAI credential form).
   - This node will connect to the discovery agent via **ai_languageModel**.

5. **Add MCP Client Tool for social search**
   - Add node: **LangChain → MCP Client Tool**
   - Set **Endpoint URL**: `https://mcp.xpoz.ai/mcp`
   - Authentication: **Bearer Auth**
   - Create/attach **HTTP Bearer Auth credential** (token from your MCP provider).
   - Connect this node to the discovery agent via **ai_tool**.

6. **Add Lead Discovery Agent**
   - Add node: **LangChain → Agent**
   - Set:
     - **Max Iterations** = `30`
     - **System message** describing qualification rules (Twitter/Instagram search; ignore spam; output platform/username/text/problem/intent; “No qualified leads found” if none; bullet points).
   - Connections:
     - Chat Trigger → Agent (**main**)
     - Memory Buffer Window → Agent (**ai_memory**)
     - Azure OpenAI model (discovery) → Agent (**ai_languageModel**)
     - MCP Client Tool → Agent (**ai_tool**)

7. **Add “CRM-ready response” chat node**
   - Add node: **LangChain → Chat**
   - Set **Message** to expression: `{{ $json.output }}`
   - Set **Wait for user reply** = `false`
   - Connect: Lead Discovery Agent → this node (**main**)

8. **Add Azure OpenAI model node for insight structuring**
   - Add node: **LangChain → Azure OpenAI Chat Model**
   - Model: **gpt-4o-mini**
   - Use the same Azure OpenAI credentials (or a separate one).
   - Connect it later via **ai_languageModel** to the second agent.

9. **Add Structured Output Parser**
   - Add node: **LangChain → Structured Output Parser**
   - Provide a JSON schema example with these top-level keys:
     - `query_context`, `results_overview`, `slack_summary`, `notion_record`
   - Ensure `notion_record.overall_intent` is constrained to: `High | Medium | Low | Mixed` and status defaults to `New`.
   - Connect it later via **ai_outputParser** to the second agent.

10. **Add Slack+Notion structuring agent**
    - Add node: **LangChain → Agent**
    - In its prompt/text, include:
      - The original query via expression referencing the Chat Trigger output (`chatInput`).
      - The leads payload (either from `$json.leads` if you standardize it, or serialize `$input.all()`).
    - In **System message**, enforce:
      - Output **valid JSON only**
      - Slack summary fields + Notion record fields
      - No emojis/markdown tables in Notion output
      - Use empty strings/zeros when missing data
    - Enable output parser on the agent and connect:
      - Azure OpenAI model (structuring) → agent (**ai_languageModel**)
      - Structured output parser → agent (**ai_outputParser**)
    - Connect: “CRM-ready response” chat node → this structuring agent (**main**)

11. **Add Slack node**
    - Add node: **Slack → Send Message** (or equivalent Slack node operation)
    - Choose **Channel** by ID (e.g., `C09GNB90TED`) and ensure the bot/app is in the channel.
    - Set message text using expressions:
      - `{{$json.output.slack_summary.title}}` etc.
    - Connect: Structuring agent → Slack node (**main**)
    - Configure Slack credential (OAuth/token) in n8n.

12. **Add Notion node**
    - Add node: **Notion → Database Page → Create**
    - Select your database (e.g., `2d8802b9-1fa0-804a-be3c-d4b876c0f228`).
    - Map properties:
      - Title property (e.g., `search_query`) ← `{{$json.output.notion_record.user_query}}`
      - Rich text properties for intent/themes
    - Connect: Structuring agent → Notion node (**main**)
    - Configure Notion integration credential and ensure DB is shared with the integration.

13. **Add error handling**
    - Add node: **Error Trigger**
    - Add node: **Gmail → Send Email**
      - Set recipient to your real address (replace `user@example.com`)
      - Subject: “Workflow Error Alert”
      - Body includes:
        - `{{$json.node.name}}`, `{{$json.error.message}}`, `{{$now.toISO()}}`
    - Connect: Error Trigger → Gmail (**main**)
    - Configure Gmail OAuth2 credential.

14. **Test**
    - Enable the Chat Trigger.
    - Run a test query such as: “Looking for alternatives to {tool} for {use case}” and verify:
      - Discovery agent returns qualified leads (or “No qualified leads found”)
      - Structuring agent returns valid JSON passing the schema parser
      - Slack message posts
      - Notion page is created

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow description note includes setup and customization guidance (Azure OpenAI, MCP bearer auth, Slack/Notion credentials, enable chat trigger, test query). | Embedded in the workflow’s main sticky note. |
| Demo video reference: “Workflow Demo Video” with YouTube id `pj4GfvdUa6k`. | https://www.youtube.com/watch?v=pj4GfvdUa6k |
| Notion database referenced as “lead gen tool db”. | https://www.notion.so/2d8802b91fa0804abe3cd4b876c0f228 |