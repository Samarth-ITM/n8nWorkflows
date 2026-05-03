Store AI chat conversations with OpenAI GPT-4.1-mini and Pinecone

https://n8nworkflows.xyz/workflows/store-ai-chat-conversations-with-openai-gpt-4-1-mini-and-pinecone-13091


# Store AI chat conversations with OpenAI GPT-4.1-mini and Pinecone

## 1. Workflow Overview

**Purpose:**  
This workflow implements an AI chat endpoint in n8n that (1) receives user chat messages, (2) generates responses using an **AI Agent** powered by **OpenAI (gpt-4.1-mini)**, (3) enriches answers with context retrieved from a **Pinecone Assistant Tool**, and (4) **logs the conversation payload to an external database** via an HTTP POST request. Finally, it returns a formatted response including an output string and a timestamp.

**Target use cases:**  
- AI chat applications that require **persistent conversation logging** (audit/compliance, analytics, QA)  
- Support or product chatbots needing **retrieval-augmented generation (RAG)** via Pinecone Assistant  
- Teams wanting a simple “chat + context + storage” pattern with an extensible database sink

### Logical Blocks
**1.1 Input Reception (Chat Trigger)**  
Receives chat messages via n8n’s chat interface/trigger and passes them into the agent.

**1.2 AI Orchestration (Agent + Model + Tool)**  
AI Agent coordinates the OpenAI chat model and Pinecone Assistant Tool to produce an answer with citations.

**1.3 Persistence (HTTP POST to DB)**  
Posts the full agent output payload to a database REST endpoint.

**1.4 Response Shaping (Set node)**  
Extracts the final text output and adds a timestamp for downstream consumption.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Chat Trigger)

**Overview:**  
Starts the workflow when a user sends a message through the n8n chat trigger. This provides the initial text and any chat-session metadata that n8n includes.

**Nodes Involved:**  
- **Chat input**

#### Node: Chat input
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for chat-based executions.
- **Configuration (interpreted):**
  - Uses a **webhook-backed chat trigger** (has a `webhookId`).
  - No special options configured (defaults).
- **Key expressions / variables:** None explicitly configured.
- **Inputs:** None (trigger node).
- **Outputs:**
  - **Main output → AI Agent**
- **Version notes:** `typeVersion 1.3` (behavior depends on n8n LangChain chat trigger implementation; ensure your n8n instance supports this node/version).
- **Edge cases / failures:**
  - Trigger not reachable / disabled workflow.
  - Chat payload shape differences between n8n versions could affect downstream assumptions (agent generally tolerates it).
  - If used outside n8n’s intended chat UI context, the incoming structure may differ.

---

### 2.2 AI Orchestration (Agent + OpenAI model + Pinecone tool)

**Overview:**  
Takes the incoming message and uses an AI Agent to produce a response. The agent can call a Pinecone Assistant Tool to retrieve relevant release information and is instructed to include file name and URL citations where referenced.

**Nodes Involved:**  
- **AI Agent**
- **OpenAI Chat Model**
- **Get context from Assistant**

#### Node: AI Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates reasoning, tool usage, and final response generation.
- **Configuration (interpreted):**
  - **System message**:  
    “You are a helpful assistant. Use the Pinecone Assistant Tool to retrieve data about Pinecone releases. Include the file name and file url in citations wherever referenced in output.”
  - This strongly nudges the agent to perform RAG through Pinecone and to return citations.
- **Key expressions / variables:** None shown; relies on upstream chat input and connected model/tool channels.
- **Inputs:**
  - **Main input** from **Chat input**
  - **AI language model** input from **OpenAI Chat Model**
  - **AI tool** input from **Get context from Assistant**
- **Outputs:**
  - **Main output → Posting Responses to DB**
- **Version notes:** `typeVersion 2.2` (agent behavior/tool calling may vary across versions).
- **Edge cases / failures:**
  - Missing model connection (agent cannot respond).
  - Tool returns empty/irrelevant context; agent output may be generic.
  - Token/context limitations (long chats or large snippets).
  - Prompt instruction conflicts (citations requested but tool results may not include expected metadata).

#### Node: OpenAI Chat Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM used by the agent.
- **Configuration (interpreted):**
  - Model set to **`gpt-4.1-mini`**
  - No additional options set (temperature, max tokens, etc. remain default for this node).
- **Credentials:**
  - Requires **OpenAI API** credentials (configured as “OpenAi account”).
- **Key expressions / variables:** None.
- **Inputs:** None (this node provides a model “resource” to the agent via the special connection type).
- **Outputs:**
  - **ai_languageModel → AI Agent**
- **Version notes:** `typeVersion 1.2`.
- **Edge cases / failures:**
  - Invalid/expired OpenAI API key, quota exceeded.
  - Model name not available for your org/region.
  - OpenAI API timeouts or rate limits.
  - If you later enable streaming or advanced parameters, payload/behavior can change.

#### Node: Get context from Assistant
- **Type / Role:** `@pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistantTool` — tool callable by the agent for retrieving contextual snippets from a Pinecone Assistant.
- **Configuration (interpreted):**
  - Assistant target (as JSON string in the field):
    - **name:** `n8n-assistant`
    - **host:** `https://prod-1-data.ke.pinecone.io`
  - Retrieval tuning:
    - `topK: 16` (returns up to 16 relevant items)
    - `snippetSize: 2048` (larger excerpts per match)
    - `sourceTag: n8n:n8n_nodes_pinecone_assistant:quickstart` (telemetry/source tag)
- **Credentials:**
  - Requires **Pinecone Assistant API** credentials (configured as “Pinecone Assistant account”).
- **Key expressions / variables:** None.
- **Inputs:** None directly (tool is invoked by the agent when needed).
- **Outputs:**
  - **ai_tool → AI Agent**
- **Version notes:** `typeVersion 1`.
- **Edge cases / failures:**
  - Wrong host/assistant name → tool errors or empty results.
  - Invalid API key / permission issues.
  - Large `topK` or `snippetSize` can increase latency and token usage downstream.
  - If the assistant does not store “file name / file url” metadata, citations requested by the system prompt may be incomplete.

---

### 2.3 Persistence (HTTP POST to Database)

**Overview:**  
Sends the agent’s resulting JSON payload to an external database endpoint for storage. This is the history logging step.

**Nodes Involved:**  
- **Posting Responses to DB**

#### Node: Posting Responses to DB
- **Type / Role:** `n8n-nodes-base.httpRequest` — performs an HTTP POST to your database REST API.
- **Configuration (interpreted):**
  - **Method:** POST  
  - **URL:** `https://your-database-api.com/endpoint` (placeholder; must be replaced)
  - **Headers:** `Content-Type: application/json`
  - **Body:** sends a field named `conversation_data` with value `={{ $json }}`
    - i.e., the entire incoming JSON from the agent is embedded as the value of `conversation_data`.
  - `sendHeaders: true`, `sendBody: true`.
- **Key expressions / variables:**
  - `conversation_data = {{$json}}`
- **Inputs:**
  - Main input from **AI Agent**
- **Outputs:**
  - Main output → **Formatting Answers**
- **Version notes:** `typeVersion 4.3`.
- **Edge cases / failures:**
  - Placeholder URL not replaced → request fails.
  - Auth missing (Bearer/API key) → 401/403.
  - Endpoint expects a different schema (e.g., expects raw JSON, not wrapped in `conversation_data`).
  - If `$json` contains nested objects, ensure your DB endpoint can parse it as intended.
  - Network failures/timeouts; consider retry settings if needed.

---

### 2.4 Response Shaping (Set)

**Overview:**  
Extracts a clean `output` string from the post-DB step result and appends a timestamp. This shapes the workflow’s final response payload.

**Nodes Involved:**  
- **Formatting Answers**

#### Node: Formatting Answers
- **Type / Role:** `n8n-nodes-base.set` — maps/creates fields for the final output.
- **Configuration (interpreted):**
  - Sets:
    - `output` = `{{ $json.output }}`
    - `timestamp` = `{{ $now }}`
- **Key expressions / variables:**
  - `$json.output` (assumes an `output` field exists in the incoming JSON at this point)
  - `$now` (current execution time)
- **Inputs:**
  - Main input from **Posting Responses to DB**
- **Outputs:** (end of workflow)
- **Version notes:** `typeVersion 3.4`.
- **Edge cases / failures:**
  - If the HTTP Request node returns a payload that does **not** contain `output`, then `output` becomes empty/undefined.
    - Common issue: the `output` is produced by the **AI Agent**, but after the HTTP call, `$json` is whatever the DB returns (often `{ id: ..., inserted: ... }`).
  - If you want the agent’s output reliably, you may need to store it before the HTTP node or merge streams.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Chat input | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat entry point / trigger | — | AI Agent | ## AI Chat Agent with Database History Logging |
| AI Agent | `@n8n/n8n-nodes-langchain.agent` | Agent orchestration (LLM + tool usage) | Chat input; OpenAI Chat Model (ai_languageModel); Get context from Assistant (ai_tool) | Posting Responses to DB | ## AI Chat Agent with Database History Logging |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM provider for agent | — | AI Agent (ai_languageModel) | ## AI Chat Agent with Database History Logging |
| Get context from Assistant | `@pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistantTool` | Pinecone Assistant retrieval tool for agent | — | AI Agent (ai_tool) | ## AI Chat Agent with Database History Logging |
| Posting Responses to DB | `n8n-nodes-base.httpRequest` | Persist conversation payload to DB via REST API | AI Agent | Formatting Answers | ## AI Chat Agent with Database History Logging |
| Formatting Answers | `n8n-nodes-base.set` | Shape final output + timestamp | Posting Responses to DB | — | ## AI Chat Agent with Database History Logging |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation (audience/problem/value) | — | — | ## Who is this for? (full note content in canvas) |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation (setup requirements) | — | — | ## Setup Requirements (full note content in canvas) |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Title banner | — | — | ## AI Chat Agent with Database History Logging |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation (customization ideas) | — | — | ## How to customize this workflow (full note content in canvas) |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation (resources/links) | — | — | ## Related Resources (includes links) |

> Note: Sticky notes are canvas annotations; they don’t connect via wires. The “Sticky Note” column above lists the relevant visible comment text.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“AI Chat Agent with Database History Logging”** (or your preferred name).

2. **Add a Chat Trigger node**
   - Node: **Chat input**
   - Type: **Chat Trigger** (`@n8n/n8n-nodes-langchain.chatTrigger`)
   - Keep defaults unless you need specific chat options.
   - This will act as the entry point.

3. **Add an AI Agent node**
   - Node: **AI Agent**
   - Type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
   - Set **System Message** to:  
     “You are a helpful assistant. Use the Pinecone Assistant Tool to retrieve data about Pinecone releases. Include the file name and file url in citations wherever referenced in output.”
   - Connect **Chat input → AI Agent** (main connection).

4. **Add the OpenAI Chat Model node**
   - Node: **OpenAI Chat Model**
   - Type: **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
   - Select model: **gpt-4.1-mini**
   - Credentials:
     - Create/select **OpenAI API** credentials (API key).
   - Connect **OpenAI Chat Model → AI Agent** using the **ai_languageModel** connection type.

5. **Add the Pinecone Assistant Tool node**
   - Node: **Get context from Assistant**
   - Type: **Pinecone Assistant Tool** (`@pinecone-database/n8n-nodes-pinecone-assistant.pineconeAssistantTool`)
   - Configure Assistant Data:
     - Name: `n8n-assistant` (or your assistant’s name)
     - Host: `https://prod-1-data.ke.pinecone.io` (replace with your Pinecone project host if different)
   - Set optional retrieval fields:
     - `topK` = 16
     - `snippetSize` = 2048
     - `sourceTag` = `n8n:n8n_nodes_pinecone_assistant:quickstart`
   - Credentials:
     - Create/select **Pinecone Assistant API** credentials (API key).
   - Connect **Get context from Assistant → AI Agent** using the **ai_tool** connection type.

6. **Add an HTTP Request node for database logging**
   - Node: **Posting Responses to DB**
   - Type: **HTTP Request** (`n8n-nodes-base.httpRequest`)
   - Method: **POST**
   - URL: replace placeholder with your real endpoint, e.g. `https://<your-service>/api/conversations`
   - Enable “Send Headers” and add:
     - `Content-Type: application/json`
     - Add auth headers as required (e.g., `Authorization: Bearer <token>`).
   - Enable “Send Body” and set body parameters:
     - `conversation_data` = `{{ $json }}`
   - Connect **AI Agent → Posting Responses to DB** (main).

7. **Add a Set node to format the final output**
   - Node: **Formatting Answers**
   - Type: **Set** (`n8n-nodes-base.set`)
   - Add fields:
     - `output` (string) = `{{ $json.output }}`
     - `timestamp` (string) = `{{ $now }}`
   - Connect **Posting Responses to DB → Formatting Answers** (main).

8. **(Recommended) Validate the “output” field behavior**
   - If your DB endpoint returns something that does not include `output`, change the design:
     - Option A: Move **Formatting Answers** *before* the HTTP Request, then send that formatted payload to DB.
     - Option B: Use a **Merge** node to keep the agent output alongside the DB response.

9. **Test execution**
   - Use the chat UI/trigger test to send a prompt.
   - Confirm:
     - The agent can call Pinecone tool successfully.
     - The POST request logs data correctly.
     - The final output is correctly populated.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Pinecone account + API key required | https://app.pinecone.io/ and API keys at https://app.pinecone.io/organizations/-/projects/-/keys |
| OpenAI account + API key required | https://auth.openai.com/create-account and keys at https://platform.openai.com/settings/organization/api-keys |
| Related video | https://youtu.be/ZWBn_OxellE |
| Example app prompt | https://swiy.co/prompt-volume-app-revewer |
| n8n AI Agents documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/ |
| Pinecone Assistant guide | https://docs.pinecone.io/guides/assistant/understanding-assistant |
| OpenAI API documentation | https://platform.openai.com/docs/ |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.