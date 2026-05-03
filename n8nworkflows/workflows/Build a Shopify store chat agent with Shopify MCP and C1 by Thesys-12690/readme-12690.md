Build a Shopify store chat agent with Shopify MCP and C1 by Thesys

https://n8nworkflows.xyz/workflows/build-a-shopify-store-chat-agent-with-shopify-mcp-and-c1-by-thesys-12690


# Build a Shopify store chat agent with Shopify MCP and C1 by Thesys

## 1. Workflow Overview

**Purpose:** This workflow creates a **public, embeddable Shopify storefront chat agent** in n8n. Users chat via the **n8n Chat UI**; an **LLM (C1 by Thesys)** powers an **AI Agent** that can call a **Shopify Storefront MCP** tool to retrieve live store/catalog/cart/checkout information and respond **as interactive UI** (streaming).

**Target use cases:**
- Storefront Q&A (“What products do you have?”)
- Guided shopping actions (“Purchase white shirt”, “Checkout my cart”)
- Dynamic UI responses (buttons/forms/cards) via Thesys C1 streaming

### 1.1 Input Reception (Public Chat)
Receives chat messages via an n8n-hosted public webhook endpoint with streaming responses enabled.

### 1.2 Agent Reasoning + Tool Use (Shopify MCP)
A LangChain-style agent receives the message, uses a system prompt, maintains short-term memory, and can call Shopify MCP tools.

### 1.3 Model + Memory + Streaming UI Output (Thesys C1)
The agent uses the **C1 Model** (OpenAI-compatible chat model endpoint via Thesys) with **streaming** enabled, returning UI-compatible output.

---

## 2. Block-by-Block Analysis

### Block A — Chat Entry Point (Public, Streaming)
**Overview:** Accepts user messages from the n8n Chat interface and forwards them to the AI Agent. Enables streaming responses back to the chat UI.

**Nodes Involved:**
- **When chat message received**

#### Node: When chat message received
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Public chat trigger (webhook-based).
- **Key configuration (interpreted):**
  - **Public:** enabled (`public: true`) → anyone with the URL can access.
  - **Response mode:** `streaming` → supports token/UI streaming back to the client.
  - **Webhook ID:** `814526b2-6ee5-498a-a2a7-472b0827c139` (used to form the chat URL).
- **Connections:**
  - **Output (main)** → **UI Agent**
- **Edge cases / failures:**
  - Public endpoint exposure: unsolicited traffic, abuse, rate spikes.
  - If streaming is unsupported by downstream nodes/model configuration, responses may fail or fall back to non-streaming behavior.
  - If the workflow is inactive, the webhook/chat URL will not respond.
- **Version notes:** Node `typeVersion 1.4` (ensure your n8n version includes this LangChain chat trigger).

---

### Block B — AI Agent Orchestration (Reasoning + Tools + Memory)
**Overview:** Central agent that interprets user intent, uses the Shopify MCP tool for storefront actions/data, and produces interactive UI responses using the connected C1 model.

**Nodes Involved:**
- **UI Agent**
- **Shopify Storefront MCP**
- **Simple Memory**
- **C1 Model**

#### Node: UI Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — Tool-using agent (LangChain-style) that can call connected tools and use connected memory and language model.
- **Key configuration (interpreted):**
  - **System message:** “You are a Shopify Store Assistant. Use the MCP to aid user in their shopping experience.”
  - **Streaming enabled:** `enableStreaming: true` (agent will stream model output when possible).
- **Connections:**
  - **Input (main)** from **When chat message received**
  - **Input (ai_languageModel)** from **C1 Model**
  - **Input (ai_memory)** from **Simple Memory**
  - **Input (ai_tool)** from **Shopify Storefront MCP**
  - **Output:** (implicit to chat trigger’s response stream; no explicit downstream node in this workflow)
- **Edge cases / failures:**
  - If the MCP tool is misconfigured/unreachable, the agent may fail tool calls or respond without store data.
  - If the model credential/base URL is wrong, the agent will fail at inference time (auth/404/timeout).
  - System prompt is minimal; ambiguous requests may lead to unintended tool usage or incomplete steps (cart/checkout flows may need more guardrails).
- **Version notes:** Node `typeVersion 3` (agent options and streaming behavior can differ across versions).

#### Node: Shopify Storefront MCP
- **Type / Role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — MCP client tool wrapper used by the agent to call Shopify Storefront MCP endpoints.
- **Key configuration (interpreted):**
  - Node shows only `options: {}` in the JSON; in practice you must set the **MCP Server URL** in the node UI (see sticky note guidance).
  - Expected MCP URL format: `https://<STOREFRONT_URL>/api/mcp`
- **Connections:**
  - **Output (ai_tool)** → **UI Agent** (tool becomes available to the agent)
- **Edge cases / failures:**
  - Wrong MCP URL (404), missing `/api/mcp`, TLS/cert issues.
  - Shopify permissions / app setup issues on the Storefront MCP server side.
  - Network timeouts or large responses causing latency.
- **Version notes:** Node `typeVersion 1.2`.

#### Node: Simple Memory
- **Type / Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Sliding-window conversation memory.
- **Key configuration (interpreted):**
  - Uses default buffer/window settings (none specified in JSON). This typically keeps a limited number of recent turns to provide context.
- **Connections:**
  - **Output (ai_memory)** → **UI Agent**
- **Edge cases / failures:**
  - Memory window too small can lose important context (e.g., cart state references).
  - Memory is not a durable store; restarting executions won’t preserve long-term history.
- **Version notes:** Node `typeVersion 1.3`.

#### Node: C1 Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI-compatible chat model connector (used here to call **Thesys C1**).
- **Key configuration (interpreted):**
  - **Model selected:** `c1/openai/gpt-5/v-20250930` (as listed in the node’s model selector)
  - **Responses API enabled:** `false` (uses standard chat-completions-style behavior).
  - **Credential:** `openAiApi` named **“Thesys N8N”** (points to Thesys base URL per sticky note).
  - Node has an internal note “ZZZ” but `notesInFlow: false` (so it may not be visible on canvas).
- **Connections:**
  - **Output (ai_languageModel)** → **UI Agent**
- **Edge cases / failures:**
  - Credential misconfiguration (wrong API key, wrong Base URL) → 401/403/404 errors.
  - Model name/version not available → request failure.
  - Streaming requires compatible upstream/downstream; if disabled, UI may not stream as expected.
- **Version notes:** Node `typeVersion 1.3` (OpenAI node options vary across n8n versions).

---

### Block C — On-Canvas Setup & Reference Notes (Sticky Notes)
**Overview:** These are documentation nodes embedded in the canvas describing how to obtain keys, configure Thesys and Shopify MCP, activate the workflow, and test via Thesys’ page.

**Nodes Involved (sticky notes):**
- Sticky Note2 (main description + links + video)
- Sticky Note5 (Step 1: Thesys setup)
- Sticky Note3 (Step 2: Shopify MCP URL)
- Sticky Note8 (Step 3/4: Activate + Try)
- Sticky Note (UI Agent explanation)
- Sticky Note1 (comparison image)

Sticky notes do not execute and have no runtime failures, but they contain critical setup steps and links.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Public chat entry point (streaming webhook) | — | UI Agent |  |
| UI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates reasoning, tool calls, memory, and streaming UI responses | When chat message received; C1 Model; Simple Memory; Shopify Storefront MCP | — | ## UI Agent; This agent can reason, use tools and returns results as interactive UI |
| C1 Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Thesys C1 OpenAI-compatible chat model used by agent | — | UI Agent | ### Step 1: Setup Thesys; 1. Go to Thesys Console and log in (https://console.thesys.dev/keys); 2. Generate a new API key...; 4. Update Base URL to https://api.thesys.dev/v1/embed; 5. Select model (example: c1/openai/gpt-5/v20251230) |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains short-term conversation context | — | UI Agent |  |
| Shopify Storefront MCP | @n8n/n8n-nodes-langchain.mcpClientTool | MCP tool to access Shopify storefront capabilities | — | UI Agent | ### Step 2: Add your Shopify MCP URL; MCP URL = https://<STOREFRONT_URL>/api/mcp; Shopify Docs: https://shopify.dev/docs/apps/build/storefront-mcp/servers/storefront |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation (overview, demo, setup, video, support) | — | — | ## Build your own Shopify Store Agent...; Demo: https://www.thesys.dev/n8n?url=https%3A%2F%2Fasd2224.app.n8n.cloud%2Fwebhook%2F814526b2-6ee5-498a-a2a7-472b0827c139%2Fchat; Video: @[youtube](0rtdVfjKJ-M); Community: https://discord.com/invite/Pbv5PsqUSv; support@thesys.dev |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation (Thesys credential setup) | — | — | ### Step 1: Setup Thesys; Base URL https://api.thesys.dev/v1/embed; Thesys keys: https://console.thesys.dev/keys |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation (Shopify MCP URL) | — | — | ### Step 2: Add your Shopify MCP URL; https://shopify.dev/docs/apps/build/storefront-mcp/servers/storefront |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation (activate + test) | — | — | ### Step 3: Activate the Workflow; Step 4: Try It Out; Thesys page: https://www.thesys.dev/n8n |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation (agent description) | — | — | ## UI Agent; This agent can reason, use tools and returns results as interactive UI |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas image/reference | — | — | https://www.thesys.dev/n8n/n8n-compare.png |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: **“Shopify Storefront Agent”**.

2. **Add Chat Trigger**
   - Create node: **When chat message received**
   - Type: **LangChain → Chat Trigger**
   - Set:
     - **Public:** enabled
     - **Response Mode:** `streaming`
   - Keep the generated chat URL/webhook for later testing.

3. **Add the Agent**
   - Create node: **UI Agent**
   - Type: **LangChain → Agent**
   - Set **Options**:
     - **System Message:** `You are a Shopify Store Assistant. Use the MCP to aid user in their shopping experience.`
     - **Enable Streaming:** on
   - Connect: **When chat message received → UI Agent** (main connection).

4. **Create Thesys (OpenAI-compatible) credentials**
   - Go to **https://console.thesys.dev/keys**
   - Generate an API key.
   - In n8n: **Credentials → New → OpenAI API**
     - Paste the Thesys API key
     - Set **Base URL** to: `https://api.thesys.dev/v1/embed`
     - Save (name it e.g. **“Thesys N8N”**)

5. **Add the C1 Model node**
   - Create node: **C1 Model**
   - Type: **LangChain → OpenAI Chat Model** (lmChatOpenAi)
   - Select credential: **Thesys N8N**
   - Choose a model from the list, matching your account availability (the template uses):  
     - `c1/openai/gpt-5/v-20250930`
   - Ensure streaming-compatible behavior (leave “Responses API” disabled if you’re matching this workflow).
   - Connect: **C1 Model → UI Agent** using the **AI Language Model** connection.

6. **Add Memory**
   - Create node: **Simple Memory**
   - Type: **LangChain → Memory Buffer Window**
   - Leave defaults (or configure window size to your needs).
   - Connect: **Simple Memory → UI Agent** using the **AI Memory** connection.

7. **Add Shopify Storefront MCP tool**
   - Create node: **Shopify Storefront MCP**
   - Type: **LangChain → MCP Client Tool**
   - Configure **MCP Server URL** as:  
     - `https://<STOREFRONT_URL>/api/mcp`
   - (Reference: https://shopify.dev/docs/apps/build/storefront-mcp/servers/storefront)
   - Connect: **Shopify Storefront MCP → UI Agent** using the **AI Tool** connection.

8. **Activate and test**
   - Toggle the workflow to **Active**.
   - Copy the public chat URL from the Chat Trigger.
   - Open: **https://www.thesys.dev/n8n**
   - Paste the chat URL to test the embedded experience.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Working demo link (template example) | https://www.thesys.dev/n8n?url=https%3A%2F%2Fasd2224.app.n8n.cloud%2Fwebhook%2F814526b2-6ee5-498a-a2a7-472b0827c139%2Fchat |
| Thesys API keys | https://console.thesys.dev/keys |
| Thesys n8n page for embedding/testing | https://www.thesys.dev/n8n |
| Shopify Storefront MCP documentation | https://shopify.dev/docs/apps/build/storefront-mcp/servers/storefront |
| Video (embedded in sticky note) | @[youtube](0rtdVfjKJ-M) |
| Thesys Community (Discord) | https://discord.com/invite/Pbv5PsqUSv |
| Support email | support@thesys.dev |
| Reference image used in canvas | https://www.thesys.dev/n8n/n8n-compare.png |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.