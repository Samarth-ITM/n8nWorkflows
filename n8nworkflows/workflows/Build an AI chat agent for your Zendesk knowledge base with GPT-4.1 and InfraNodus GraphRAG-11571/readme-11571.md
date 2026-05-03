Build an AI chat agent for your Zendesk knowledge base with GPT-4.1 and InfraNodus GraphRAG

https://n8nworkflows.xyz/workflows/build-an-ai-chat-agent-for-your-zendesk-knowledge-base-with-gpt-4-1-and-infranodus-graphrag-11571


# Build an AI chat agent for your Zendesk knowledge base with GPT-4.1 and InfraNodus GraphRAG

## 1. Workflow Overview

**Workflow name:** AI Chat Agent for Zendesk Knowledge Base  
**Purpose:** Provide a chat-based support agent that answers user questions by combining:
1) **GraphRAG context retrieval** from an InfraNodus “expert ontology” graph, and  
2) **Zendesk Help Center article search**,  
then synthesizing a concise response with **markdown references** to the support articles used.

**Primary use cases**
- Website / support portal chat widget answering FAQs from a Zendesk knowledge base.
- Support deflection and faster self-service answers with a GraphRAG-guided query rewrite.

### 1.1 Input Reception (Webhook entry)
Receives chat messages (and a sessionId) via a public webhook endpoint (commonly from the n8n Chat Widget).

### 1.2 Agent Orchestration (LLM + Memory)
A LangChain Agent node uses:
- an OpenAI chat model (GPT-4.1-mini),
- a windowed conversation memory keyed by sessionId,
and is instructed (system prompt) to first call GraphRAG, then craft a better Zendesk search query, then respond.

### 1.3 Knowledge Retrieval Tools (GraphRAG + Zendesk Search)
Two “tool” nodes are exposed to the agent:
- InfraNodus GraphRAG tool for ontology-based retrieval/summary.
- HTTP Request Tool calling Zendesk Search API to fetch relevant articles.

### 1.4 Output Delivery (Webhook response)
Returns the agent’s final answer to the original webhook request.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Webhook)
**Overview:** Accepts inbound chat requests (message + sessionId) and triggers the agent.  
**Nodes involved:** `Webhook`

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point HTTP endpoint.
- **Configuration choices:**
  - **Path:** a fixed UUID-like path (`486b2675-a4d1-4050-ad3a-7d92a57c084b`).
  - **Response mode:** “Response Node” (workflow must end with Respond to Webhook).
  - **Allowed origins:** `*` (CORS open).
- **Key expressions / variables used downstream:**
  - Expects query parameters such as:
    - `query.message` (user message)
    - `query.sessionId` (conversation/session key)
- **Connections:**
  - **Output (main)** → `Support Agent` (main)
- **Edge cases / failures:**
  - Missing `query.message` or `query.sessionId` leads to downstream expression issues (empty prompt or memory key).
  - Open CORS may be undesirable in production; consider restricting origins.
  - If the workflow errors before Respond to Webhook runs, the caller may see a timeout/5xx.

---

### Block 2 — Agent Orchestration (LLM + Memory)
**Overview:** The agent receives the user message, consults tools in the required order (GraphRAG then Zendesk), and composes the final answer. Memory maintains chat context across turns per sessionId.  
**Nodes involved:** `Support Agent`, `OpenAI Chat Model`, `Simple Memory`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM backend to the agent.
- **Configuration choices:**
  - **Model:** `gpt-4.1-mini`
  - No special options configured.
- **Credentials:** OpenAI API credential (“OpenAi account 2”).
- **Connections:**
  - **Output (ai_languageModel)** → `Support Agent` (ai_languageModel)
- **Edge cases / failures:**
  - Invalid/expired OpenAI key, quota exceeded, model unavailable.
  - Latency/timeouts on long tool runs or large contexts.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — chat memory buffer (windowed).
- **Configuration choices:**
  - **Session ID type:** Custom key
  - **Session key expression:** `={{ $('Webhook').item.json.query.sessionId }}`
    - This binds memory to the inbound sessionId, enabling multi-turn chat context.
- **Connections:**
  - **Output (ai_memory)** → `Support Agent` (ai_memory)
- **Edge cases / failures:**
  - If `query.sessionId` is missing/empty, all users may share a session or memory may fail to resolve.
  - Memory window size is not explicitly shown; defaults apply (could truncate earlier context).

#### Node: Support Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool calling + response generation.
- **Configuration choices:**
  - **User input text:** `={{ $json.query.message }}`
  - **Prompt type:** “define” (custom system message)
  - **System message (core logic):**
    - If first run / no memory: **always run GraphRAG** to extract context (`graphSummary`).
    - Use GraphRAG knowledge to **generate an enhanced search query**.
    - **Only then** query Zendesk via the Zendesk tool.
    - Produce a **concise response** using both sources.
    - Provide **references to support articles in markdown**.
- **Tooling available to the agent (via connections):**
  - InfraNodus GraphRAG tool node
  - Zendesk search HTTP tool node
- **Connections:**
  - **Inputs:**
    - From `Webhook` (main)
    - From `OpenAI Chat Model` (ai_languageModel)
    - From `Simple Memory` (ai_memory)
    - From `Get a response... InfraNodus Graph RAG` (ai_tool)
    - From `Zendek Search` (ai_tool)
  - **Output (main)** → `Respond to Webhook`
- **Edge cases / failures:**
  - If the LLM does not follow the “GraphRAG first” instruction, Zendesk search may be less relevant (prompt adherence risk).
  - If tool schemas/outputs differ from what the agent expects, it may hallucinate or fail to cite sources.
  - If `query.message` is missing, the agent may respond generically or error on empty input.
- **Version-specific notes:** Agent node is typeVersion `2.2`; tool wiring uses n8n’s LangChain tool ports (`ai_tool`, `ai_memory`, `ai_languageModel`).

---

### Block 3 — Knowledge Retrieval Tools (GraphRAG + Zendesk Search)
**Overview:** These nodes are not executed linearly; they are exposed as callable tools to the agent. The agent decides when/how to call them (guided by the system message).  
**Nodes involved:** `Get a response from knowledge base in InfraNodus Graph RAG`, `Zendek Search`

#### Node: Get a response from knowledge base in InfraNodus Graph RAG
- **Type / role:** `n8n-nodes-infranodus.infranodusTool` — tool wrapper to query InfraNodus GraphRAG.
- **Configuration choices:**
  - **Tool name:** `infranodus_support` (this is what the agent “sees”)
  - **Prompt input:** Uses an AI-mapped parameter:
    - `={{ $fromAI('Prompt', ``, 'string') }}`
    - This indicates the agent fills the “Prompt” argument when invoking the tool.
- **Credentials:** InfraNodus API credential (“InfraNodus API / user expert”).
- **Connections:**
  - **Output (ai_tool)** → `Support Agent` (ai_tool)
- **Edge cases / failures:**
  - Invalid InfraNodus API key or insufficient permissions to access the ontology/graph.
  - GraphRAG latency/timeouts, especially on complex prompts.
  - If the agent passes an empty prompt, results may be irrelevant or empty.

#### Node: Zendek Search
- **Type / role:** `n8n-nodes-base.httpRequestTool` — HTTP tool callable by the agent to query Zendesk Help Center Search API.
- **Configuration choices:**
  - **Method:** (not explicitly shown; typically GET for this endpoint)
  - **URL:** `https://noduslabs.zendesk.com/api/v2/help_center/articles/search`
  - **Query params:**
    - `query` = `={{ $fromAI('parameters0_Value', ``, 'string') }}`
      - Agent supplies a simplified/augmented search phrase.
    - `per_page` = `5`
  - **Authentication:** Predefined credential type `zendeskApi`
  - **Tool description:** “Search through the Zendesk articles using a simplified search phrase as a query parameter”
- **Credentials:** Zendesk credential (“Zendesk account”); typically API token-based.
- **Connections:**
  - **Output (ai_tool)** → `Support Agent` (ai_tool)
- **Edge cases / failures:**
  - Wrong Zendesk subdomain in URL (replace `noduslabs` with your instance).
  - 401/403 if API token invalid, missing scopes, or account lacks Help Center API access.
  - Rate limiting (429) on frequent searches.
  - The agent may pass overly long or malformed queries; consider sanitizing or limiting length.

---

### Block 4 — Output Delivery (Return to caller)
**Overview:** Sends the agent’s final response back to the original webhook caller (e.g., n8n Chat Widget).  
**Nodes involved:** `Respond to Webhook`

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — final HTTP response for “responseNode” mode.
- **Configuration choices:** Default options (no custom headers/body mapping shown).
  - In practice, it returns whatever the incoming item contains at this point (agent output).
- **Connections:**
  - **Input (main)** from `Support Agent`
  - No further outputs.
- **Edge cases / failures:**
  - If the agent output is not in the expected structure, the widget/client may not display properly.
  - If earlier nodes error, this node won’t run and the caller gets a timeout.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry point to receive chat requests | — | Support Agent | ## 1. Create Webhook\nThis will activate your workflow every time a user message arrives to the URL provided.\n\n### You can connect it to the [n8n Chat Widget](https://n8n-chat-widget.com) to trigger it via a popup chat that you can embed to your support portal. |
| Support Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates tool calls + final answer | Webhook; OpenAI Chat Model (ai_languageModel); Simple Memory (ai_memory); Zendek Search (ai_tool); Get a response from knowledge base in InfraNodus Graph RAG (ai_tool) | Respond to Webhook | ## 2. Add AI Agent Node\n\n🚨 Edit the prompt here to be more suitable for your content, but keep the structure — it works pretty well!\n\nThis AI agent is instructed to: \n1) Consult the [InfraNodus GraphRAG](https://infranodus.com/docs/graph-rag-knowledge-graph) node to retrieve an authoritative response to the user's query.\n2) Then rewrite user's query with that context obtained to get the most relevant results from the Zendesk support portal\n3) Then synthesize these responses and deliver them to the user. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for the agent | — | Support Agent | ## 3. LLM Model to Use\n\nConnect it to your favorite LLM and set up API authentication for it. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains chat context per session | — | Support Agent (ai_memory) | ## 4. Chat Memory Node\n\nThis maintains the context of the conversation. |
| Get a response from knowledge base in InfraNodus Graph RAG | n8n-nodes-infranodus.infranodusTool | GraphRAG retrieval tool (ontology expert) | — (called by agent) | Support Agent (ai_tool) | ## 5. Expert Ontology\n\nUses [InfraNodus GraphRAG node](https://n8n.io/integrations/infranodus-graph-rag/) to extract a relevant response on a query based on prebuilt |
| Zendek Search | n8n-nodes-base.httpRequestTool | Tool to search Zendesk Help Center articles | — (called by agent) | Support Agent (ai_tool) | ## 5. Zendesk Knowledge Search\n\nUses [Zendesk search API](https://developer.zendesk.com/api-reference/help_center/help-center-api/search/) to find relevant articles based on the search query augmented using the InfraNodus expert ontology data (5). \n\nGet the Zendesk API key at your support portal Admin > Apps & Integrations > API Tokens. Usually it's located at [https://noduslabs.zendesk.com/admin/apps-integrations/apis/api-tokens](https://noduslabs.zendesk.com/admin/apps-integrations/apis/api-tokens) where instead of `noduslabs` you need to put the name of your support portal. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns agent response to the webhook caller | Support Agent | — | ## 6. Send Response Back to the User\n\nSends the final response, based on the InfraNodus ontology expert graph and on the search results from Zendesk support portal back to the user.\n\n### The message is sent to the webhook, which is then picked up by the [n8n Chat Widget](https://n8n-chat-widget.com) embedded on your website. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | # n8n Zendesk Support AI Chat Agent\n\n## Add a widget to your Zendesk support portal or any website that uses n8n and [InfraNodus](https://infranodus.com) expert ontology to provide high-quality GraphRAG  powered responses to your users.\n\n### This widget can be embedded on any website using the [n8n Chat Widget](https://n8n-chat-widget.com) generator. Here's an example from our own support portal: [support.noduslabs.com](https://support.noduslabs.com)\n\n## Video Tutorial:\n\n[![Video tutorial](https://img.youtube.com/vi/aYoPSEmGJbc/sddefault.jpg)](https://www.youtube.com/watch?v=aYoPSEmGJbc)\n\n## Integration Example:\n\n[![InfraNodus Ontology](https://support.noduslabs.com/hc/article_attachments/24080339347484)](https://support.noduslabs.com/hc/en-us) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “Support Agent” for content) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “OpenAI Chat Model” for content) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “Simple Memory” for content) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “Get a response from knowledge base in InfraNodus Graph RAG” for content) |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “Zendek Search” for content) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “Respond to Webhook” for content) |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | ### The Ontology is an [InfraNodus](https://infranodus.com) Graph Used to Help Model Rewrite Prompt and Extract Better Results from Zendesk:\n\n![InfraNodus Ontology](https://support.noduslabs.com/hc/article_attachments/24080217545116) |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation/commentary | — | — | (see table row “Sticky Note” for content) |

> Note: Sticky Notes are nodes in the JSON, but they don’t participate in execution. They are included here to satisfy “Do not skip any nodes”.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **AI Chat Agent for Zendesk Knowledge Base** (or your preferred name).

2) **Add the Webhook node**
   - Node type: **Webhook**
   - Set **Response Mode** = **Using ‘Respond to Webhook’ Node**
   - Set **Path** to any unique string (keep it stable for the widget integration).
   - (Optional) Set **Allowed Origins** (CORS) appropriately; the example uses `*`.
   - This node must output data shaped like:
     - `query.message` (string)
     - `query.sessionId` (string)

3) **Add the AI Agent node**
   - Node type: **AI Agent** (LangChain Agent in n8n)
   - Set **Text** (user input) to an expression that reads the webhook message:
     - `{{$json.query.message}}`
   - Set **Prompt Type** to **Define**
   - Paste/author a **System Message** with the same structure:
     - First consult GraphRAG on first run/no memory
     - Use GraphRAG result to rewrite/augment the Zendesk search query
     - Call Zendesk search tool
     - Answer concisely and cite articles in markdown
   - Connect: **Webhook (main)** → **Support Agent (main)**

4) **Add the OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model**
   - Choose model: **gpt-4.1-mini** (or your choice)
   - Configure **OpenAI credentials**:
     - Create/Open an OpenAI API credential in n8n and paste the API key.
   - Connect: **OpenAI Chat Model (ai_languageModel)** → **Support Agent (ai_languageModel)**

5) **Add the Memory node**
   - Node type: **Simple Memory** (Buffer Window Memory)
   - Configure:
     - **Session ID type**: Custom key
     - **Session key** expression: `{{$('Webhook').item.json.query.sessionId}}`
       - Ensure your webhook indeed sends `sessionId`.
   - Connect: **Simple Memory (ai_memory)** → **Support Agent (ai_memory)**

6) **Add the InfraNodus GraphRAG tool node**
   - Node type: **InfraNodus GraphRAG Tool** (InfraNodus Tool node)
   - Configure:
     - **Tool name**: `infranodus_support` (or your preferred tool name)
     - **Prompt**: leave as an AI-filled parameter (n8n will map it); in the example it uses:
       - `$fromAI('Prompt', '', 'string')`
   - Configure **InfraNodus credentials**:
     - Add InfraNodus API credentials (API key / account with access to the expert ontology graph).
   - Connect: **InfraNodus tool (ai_tool)** → **Support Agent (ai_tool)**

7) **Add the Zendesk search tool node**
   - Node type: **HTTP Request Tool**
   - Configure:
     - **URL:** `https://<your_subdomain>.zendesk.com/api/v2/help_center/articles/search`
     - Enable **Send Query Parameters**
     - Query params:
       - `query` = AI-filled parameter (example uses `$fromAI(...)`)
       - `per_page` = `5`
     - Set **Authentication** to **Predefined Credential Type** = `zendeskApi`
     - Add a **Tool Description** explaining it searches Zendesk articles.
   - Configure **Zendesk credentials**:
     - Create a Zendesk API token in Zendesk Admin: **Apps & Integrations → APIs → API tokens**
     - In n8n, set Zendesk credential (commonly email + API token).
   - Connect: **Zendesk Search (ai_tool)** → **Support Agent (ai_tool)**

8) **Add Respond to Webhook node**
   - Node type: **Respond to Webhook**
   - Connect: **Support Agent (main)** → **Respond to Webhook (main)**
   - Keep default options unless your chat widget expects a specific JSON schema.

9) **(Optional) Add Sticky Notes**
   - Add sticky notes with your operational instructions/links (not required for execution).

10) **Activate and test**
   - Call the webhook with query parameters, e.g.:
     - `?sessionId=abc123&message=How%20do%20I%20...`
   - Verify the agent calls InfraNodus first (especially on new sessionId), then Zendesk, and returns markdown references.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| n8n Chat Widget for embedding the chat on your site | https://n8n-chat-widget.com |
| InfraNodus GraphRAG documentation | https://infranodus.com/docs/graph-rag-knowledge-graph |
| InfraNodus GraphRAG node integration page (n8n) | https://n8n.io/integrations/infranodus-graph-rag/ |
| Zendesk Help Center Search API reference | https://developer.zendesk.com/api-reference/help_center/help-center-api/search/ |
| Example Zendesk API tokens page (replace subdomain) | https://noduslabs.zendesk.com/admin/apps-integrations/apis/api-tokens |
| Example support portal | https://support.noduslabs.com |
| Video (YouTube) | https://www.youtube.com/watch?v=aYoPSEmGJbc |
| Ontology image (example) | https://support.noduslabs.com/hc/article_attachments/24080217545116 |
| Integration example page | https://support.noduslabs.com/hc/en-us |
| Disclaimer (FR): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Provided by user prompt (compliance note) |