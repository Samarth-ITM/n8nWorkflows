Analyze crypto markets with CoinGecko MCP and C1

https://n8nworkflows.xyz/workflows/analyze-crypto-markets-with-coingecko-mcp-and-c1-12548


# Analyze crypto markets with CoinGecko MCP and C1

## 1. Workflow Overview

**Purpose:**  
This workflow exposes a **public n8n Chat endpoint** where users can ask questions about crypto markets. An **AI Agent** (running on **C1 by Thesys**, via an OpenAI-compatible credential) interprets each prompt, optionally calls the **CoinGecko Free MCP** tool to fetch live market data, and returns a **streaming response designed for interactive UI rendering** (charts/cards/buttons) on the Thesys front-end.

**Primary use cases:**
- Real-time price checks (BTC, ETH, etc.)
- Market summaries (market cap, dominance, movers)
- Token comparisons over time (intended to be shown as charts in UI)

### 1.1 Input Reception (Public Chat)
Receives user prompts via n8n’s Chat Trigger in **public mode** with **streaming responses**.

### 1.2 AI Orchestration (Agent + Memory)
Routes the prompt to a LangChain Agent configured with a crypto-specific system message, with short-term conversational memory.

### 1.3 Market Data Retrieval (CoinGecko MCP Tool)
Enables the agent to query CoinGecko’s MCP endpoint for market/trending/token data.

### 1.4 Response Generation (C1 Model via Thesys)
Uses the **C1** model (OpenAI-compatible chat model) with streaming enabled, intended to output UI-schema-like content for Thesys GenUI rendering.

---

## 2. Block-by-Block Analysis

### Block 1 — Public Chat Entry
**Overview:** Accepts incoming chat messages through a public endpoint and streams responses back to the client UI. This is the workflow’s entry point.  
**Nodes involved:**  
- When chat message received

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Workflow trigger for n8n Chat.
- **Key configuration (interpreted):**
  - **Public:** enabled (`public: true`) so anyone with the URL can access.
  - **Response mode:** **streaming** to support partial/real-time outputs.
  - **Allowed origins:** `*` (CORS open).
  - Generates a unique **chat webhook URL** based on `webhookId`.
- **Connections:**
  - **Output (main) → UI Agent (main)**.
- **Edge cases / failures:**
  - Public endpoint can be abused (rate limiting and auth not present here).
  - `allowedOrigins: *` is permissive; embedding in other sites is possible.
  - If workflow is inactive, the endpoint will not respond.
  - Streaming depends on client support; some clients may not render partial updates cleanly.
- **Version notes:** Node `typeVersion 1.4` (ensure compatible n8n + LangChain nodes package).

---

### Block 2 — Agent Orchestration + Short-Term Memory
**Overview:** The agent receives the prompt, keeps a buffer of recent conversation context, and decides whether to call CoinGecko MCP.  
**Nodes involved:**  
- UI Agent  
- Simple Memory

#### Node: UI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Reasoning agent that can use tools + memory + a chat model.
- **Key configuration (interpreted):**
  - **System message:** “You are a crypto copilot”
  - **Streaming:** enabled (`enableStreaming: true`) so responses can stream back to the chat trigger.
- **Connections (inputs):**
  - **Main input:** from **When chat message received**
  - **AI language model input:** from **C1 Model**
  - **AI memory input:** from **Simple Memory**
  - **AI tool input:** from **CoinGecko Free MCP**
- **Connections (outputs):**
  - The node’s `main` output is present but not wired to any next node (end of workflow).
- **Edge cases / failures:**
  - If the model credential is misconfigured (wrong base URL / invalid key), the agent fails to respond.
  - If the agent chooses to call the tool and the MCP endpoint is down/slow, you may see timeouts or partial responses.
  - With minimal system instructions, answers may vary in format; UI rendering depends on the C1/Thesys behavior.
- **Version notes:** Node `typeVersion 3` (agent behavior/options can differ across versions).

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Maintains a rolling conversation window for context.
- **Key configuration (interpreted):**
  - Uses default buffer/window settings (no explicit parameters provided).
- **Connections:**
  - **Output (ai_memory) → UI Agent (ai_memory)**.
- **Edge cases / failures:**
  - Memory window size defaults may be too small/large depending on n8n defaults; long chats may lose older context.
  - If multiple users share the same public chat endpoint, ensure session separation is handled by the chat trigger/client; otherwise context leakage is possible (depends on n8n chat session handling).
- **Version notes:** Node `typeVersion 1.3`.

---

### Block 3 — CoinGecko Market Data Tooling (MCP)
**Overview:** Provides the agent with access to CoinGecko’s “Free MCP” endpoint so it can fetch live crypto market data on demand.  
**Nodes involved:**  
- CoinGecko Free MCP

#### Node: CoinGecko Free MCP
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — MCP client tool exposed to the agent.
- **Key configuration (interpreted):**
  - **Endpoint URL:** `https://mcp.api.coingecko.com/mcp`
  - No additional options set.
- **Connections:**
  - **Output (ai_tool) → UI Agent (ai_tool)**.
- **Edge cases / failures:**
  - Network failures / DNS / TLS issues reaching `mcp.api.coingecko.com`.
  - API rate limits on CoinGecko Free tier (requests may be throttled/blocked).
  - If CoinGecko changes MCP schema/tools, the agent’s tool calls may fail or degrade.
- **Version notes:** Node `typeVersion 1.2`.

---

### Block 4 — Model Layer (C1 by Thesys via OpenAI-Compatible Node)
**Overview:** Supplies the agent with the C1 model (Thesys middleware) using the OpenAI-compatible chat model node and a configured API key/base URL.  
**Nodes involved:**  
- C1 Model

#### Node: C1 Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Chat LLM connector (OpenAI-compatible) used by the agent.
- **Key configuration (interpreted):**
  - **Model selected:** `c1/openai/gpt-5/v-20250930`
  - **Responses API enabled:** disabled (`responsesApiEnabled: false`)
  - Additional model options: none specified.
  - **Credential:** “OpenAi account 2” (but intended to be configured for **Thesys** per sticky note: API key + base URL override).
- **Connections:**
  - **Output (ai_languageModel) → UI Agent (ai_languageModel)**.
- **Edge cases / failures:**
  - If you use a normal OpenAI key/base URL, `c1/...` model names will not resolve; must use Thesys base URL per notes.
  - Auth errors (401/403) if API key invalid/expired.
  - Model name/version may be unavailable; requires updating to a valid Thesys-supported identifier.
  - Streaming requires both model endpoint + client to support streamed tokens/events.
- **Version notes:** Node `typeVersion 1.3`.
- **Internal note field:** Node contains note text “ZZZ” (not shown in flow since `notesInFlow: false`).

---

### Block 5 — Operator Notes / Embedded Documentation (Sticky Notes)
**Overview:** Sticky notes provide setup instructions, links, and a demo reference. They do not affect execution but are important for correct deployment.  
**Nodes involved:**  
- Sticky Note2  
- Sticky Note5  
- Sticky Note8  
- Sticky Note  
- Sticky Note1

#### Node: Sticky Note2
- **Type / role:** `n8n-nodes-base.stickyNote` — Template overview + links.
- **Key content highlights:**
  - Explains the workflow behavior (chat → agent → CoinGecko MCP → C1 UI response).
  - Demo link:  
    https://www.thesys.dev/n8n?url=https://www.thesys.dev/n8n?url=https%3A%2F%2Fasd2224.app.n8n.cloud%2Fwebhook%2F51638b0c-7765-4fa8-9b95-a0422128e203%2Fchat
  - Thesys API keys: https://console.thesys.dev/keys
  - Thesys site: https://www.thesys.dev/
  - Video embed: `@[youtube](0rtdVfjKJ-M)`
  - Community/support:
    - https://discord.com/invite/Pbv5PsqUSv
    - support@thesys.dev
- **Edge cases:** None (non-executable).

#### Node: Sticky Note5
- **Type / role:** `n8n-nodes-base.stickyNote` — Step-by-step setup for Chat + Thesys credential.
- **Key instructions (interpreted):**
  - Enable public chat, copy chat URL.
  - Create Thesys API key.
  - In the **Model node**, create a new credential and set:
    - **API Key:** Thesys key
    - **Base URL:** `https://api.thesys.dev/v1/embed`
    - Select a model (example given in note: `c1/openai/gpt-5/v20251230`).
- **Edge cases:** None (non-executable).

#### Node: Sticky Note8
- **Type / role:** `n8n-nodes-base.stickyNote` — Activation + testing steps.
- **Key links:**
  - https://www.thesys.dev/n8n
- **Edge cases:** None (non-executable).

#### Node: Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — Brief description of the UI agent.
- **Content:** “This agent can reason, use tools and returns results as interactive UI”

#### Node: Sticky Note1
- **Type / role:** `n8n-nodes-base.stickyNote` — Image reference.
- **Content:** Image: https://www.thesys.dev/n8n/n8n-compare.png

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) | Public chat entry point with streaming | — | UI Agent | ### Step 1: Enable Chat / ### Step 2: Setup Thesys (instructions include https://console.thesys.dev/keys and base URL https://api.thesys.dev/v1/embed) |
| UI Agent | Agent (`@n8n/n8n-nodes-langchain.agent`) | Interprets prompt, calls tools, streams UI-oriented response | When chat message received; C1 Model; Simple Memory; CoinGecko Free MCP | — | ## UI Agent — This agent can reason, use tools and returns results as interactive UI |
| Simple Memory | Memory Buffer Window (`@n8n/n8n-nodes-langchain.memoryBufferWindow`) | Short-term conversation context | — | UI Agent |  |
| CoinGecko Free MCP | MCP Client Tool (`@n8n/n8n-nodes-langchain.mcpClientTool`) | Live crypto data tool via CoinGecko MCP | — | UI Agent |  |
| C1 Model | OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) | LLM provider (Thesys C1) for UI responses | — | UI Agent | ### Step 1: Enable Chat / ### Step 2: Setup Thesys (set base URL https://api.thesys.dev/v1/embed; choose c1 model) |
| Sticky Note2 | Sticky Note | Template overview + demo/support links | — | — |  |
| Sticky Note5 | Sticky Note | Setup instructions (Chat public + Thesys key + base URL) | — | — |  |
| Sticky Note8 | Sticky Note | Activate workflow + test on Thesys page | — | — |  |
| Sticky Note | Sticky Note | Explains UI Agent purpose | — | — |  |
| Sticky Note1 | Sticky Note | Image reference | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger node**
   1. Add node: **Chat Trigger** (LangChain) → name it **When chat message received**.
   2. Set **Public** = enabled.
   3. Set **Response Mode** = **Streaming**.
   4. Set **Allowed Origins** = `*` (or restrict to your domain if desired).
   5. Copy the generated **Chat URL** (you’ll use it on the Thesys n8n page).

2) **Create the AI Agent**
   1. Add node: **AI Agent** (LangChain) → name it **UI Agent**.
   2. In Agent **Options**:
      - **System message**: `You are a crypto copilot`
      - **Enable Streaming**: enabled
   3. Connect: **When chat message received (main)** → **UI Agent (main)**.

3) **Add memory**
   1. Add node: **Simple Memory** → type **Memory Buffer Window**.
   2. Keep defaults (or configure window size if your n8n UI exposes it).
   3. Connect: **Simple Memory (AI Memory output)** → **UI Agent (AI Memory input)**.

4) **Add CoinGecko MCP tool**
   1. Add node: **MCP Client Tool** → name it **CoinGecko Free MCP**.
   2. Set **Endpoint URL** to: `https://mcp.api.coingecko.com/mcp`
   3. Connect: **CoinGecko Free MCP (AI Tool output)** → **UI Agent (AI Tool input)**.

5) **Add the C1 model (Thesys)**
   1. Add node: **OpenAI Chat Model** (LangChain) → name it **C1 Model**.
   2. Set **Model** to: `c1/openai/gpt-5/v-20250930` (or another C1 model you have access to).
   3. Ensure **Responses API** is disabled (match the workflow setting).
   4. **Create credentials** for this node:
      - Credential type: OpenAI-compatible (as used by the OpenAI Chat Model node in n8n)
      - **API Key:** your Thesys API key from https://console.thesys.dev/keys
      - **Base URL:** `https://api.thesys.dev/v1/embed`
   5. Connect: **C1 Model (AI Language Model output)** → **UI Agent (AI Language Model input)**.

6) **Activate and test**
   1. Toggle workflow to **Active**.
   2. Go to https://www.thesys.dev/n8n
   3. Paste your **Chat URL** and send prompts like:
      - “What’s the current price of Bitcoin and Ethereum?”
      - “Compare ETH vs SOL over 30 days with a chart.”

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo link (preconfigured chat URL example) | https://www.thesys.dev/n8n?url=https://www.thesys.dev/n8n?url=https%3A%2F%2Fasd2224.app.n8n.cloud%2Fwebhook%2F51638b0c-7765-4fa8-9b95-a0422128e203%2Fchat |
| Thesys API key management | https://console.thesys.dev/keys |
| Thesys n8n front-end (paste Chat URL here to test) | https://www.thesys.dev/n8n |
| Thesys product site | https://www.thesys.dev/ |
| Thesys community/support | https://discord.com/invite/Pbv5PsqUSv ; support@thesys.dev |
| Reference image in sticky note | https://www.thesys.dev/n8n/n8n-compare.png |
| Embedded video reference | `@[youtube](0rtdVfjKJ-M)` |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.