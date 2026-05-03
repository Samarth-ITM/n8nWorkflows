Test WAF security interactively with an AI agent and WAFtester MCP

https://n8nworkflows.xyz/workflows/test-waf-security-interactively-with-an-ai-agent-and-waftester-mcp-13443


# Test WAF security interactively with an AI agent and WAFtester MCP

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow exposes an **interactive n8n Chat UI** where a user can ask for **WAF (Web Application Firewall) testing tasks** in natural language. An **AI Agent** interprets the request, then calls **WAFtester** tools via **MCP (Model Context Protocol)** to perform WAF detection, discovery, scanning, bypass search, and assessment.

**Target use cases:**
- Ad-hoc WAF evaluation for owned/authorized targets
- AI-assisted recon and test planning for penetration testing
- DevSecOps exploratory validation of WAF coverage

### 1.1 Input Reception (Chat UI)
User messages enter through n8n’s built-in chat interface.

### 1.2 Agent Reasoning & Orchestration
An AI Agent receives the chat message, follows a prescribed methodology (detect → discover → learn → scan → bypass → assess), and decides which WAFtester tools to run.

### 1.3 Tool Execution via MCP (WAFtester)
The agent invokes WAFtester’s MCP tools over **SSE (Server-Sent Events)**. Async tools return a `task_id`, and the agent is instructed to poll status with `get_task_status`.

### 1.4 LLM Provider (OpenAI)
The AI Agent uses an OpenAI chat model as its reasoning layer.

---

## 2. Block-by-Block Analysis

### Block 1 — Chat Entry Point
**Overview:**  
Receives user requests in the n8n chat interface and forwards them to the AI Agent.

**Nodes Involved:**
- **Chat Trigger**

#### Node: Chat Trigger
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point that opens/handles the n8n Chat UI.
- **Configuration choices:** No custom parameters set; relies on default chat trigger behavior.
- **Key expressions / variables:** None.
- **Connections:**
  - **Output →** `AI Agent` (main, index 0)
- **Version-specific requirements:** Node typeVersion `1.1`.
- **Edge cases / failures:**
  - If the workflow is not **Active**, the chat interface won’t route messages to this workflow.
  - If n8n chat is not enabled/available in the deployment context, users cannot interact.
- **Sub-workflow reference:** None.

---

### Block 2 — Agent Reasoning & Control Plane
**Overview:**  
Interprets chat messages, applies a structured WAF-testing methodology via its system prompt, and dispatches calls to the WAFtester MCP tools while using OpenAI as the language model.

**Nodes Involved:**
- **AI Agent**
- **OpenAI Chat Model**
- **WAFtester MCP**

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Central orchestrator that receives chat input, uses an LLM, and calls external tools.
- **Configuration choices (interpreted):**
  - Uses a **system message** that:
    - Defines the assistant as a “WAF security testing assistant powered by WAFtester”.
    - Enforces a typical workflow order: `detect_waf → discover → learn → scan → bypass → assess`.
    - Requires starting with `detect_waf` for new targets.
    - Explains async behavior: `scan`, `assess`, `bypass`, `discover`, `discover_bypasses`, `event_crawl` return `task_id`, and the agent should poll via `get_task_status`.
    - Lists supported attack categories (e.g., `sqli`, `xss`, `ssrf`, `cmdi`, etc.).
- **Key expressions / variables:** System prompt only (no n8n expressions).
- **Connections:**
  - **Input ←** `Chat Trigger` (main)
  - **Tooling ←** `WAFtester MCP` (ai_tool)
  - **Language model ←** `OpenAI Chat Model` (ai_languageModel)
- **Version-specific requirements:** Node typeVersion `2`.
- **Edge cases / failures:**
  - If OpenAI credentials are missing/invalid, agent reasoning fails (auth/429/rate limits).
  - If WAFtester MCP SSE endpoint is unreachable, tool calls fail or time out.
  - If the user provides an ambiguous target (no URL/host), the agent may need clarification; otherwise tools may error.
  - Async task polling depends on WAFtester providing consistent `task_id` and status responses.
- **Sub-workflow reference:** None (single-workflow design).

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the chat LLM used by the AI Agent.
- **Configuration choices (interpreted):**
  - Uses OpenAI API credentials named **“OpenAI account”**.
  - No explicit model/options are set in parameters (defaults apply in node UI unless selected).
  - Note suggests “GPT-4o recommended”.
- **Key expressions / variables:** None.
- **Connections:**
  - **Output →** `AI Agent` (ai_languageModel)
- **Version-specific requirements:** Node typeVersion `1.2`.
- **Edge cases / failures:**
  - Credential misconfiguration (invalid key, wrong org/project) → authentication errors.
  - Rate limiting or quota exhaustion → 429 errors.
  - Model not available in the account/region → model errors.
- **Sub-workflow reference:** None.

#### Node: WAFtester MCP
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — Connects to an MCP server and exposes its tools to the AI Agent.
- **Configuration choices (interpreted):**
  - Uses **SSE endpoint** parameter:
    - `{{ $env.WAFTESTER_SSE_URL || 'http://waftester:8080/sse' }}`
  - This means:
    - Prefer the environment variable `WAFTESTER_SSE_URL` if set
    - Otherwise default to `http://waftester:8080/sse` (typical Docker network hostname)
  - Note indicates “All 18 WAFtester tools are auto-discovered.”
- **Key expressions / variables:**
  - `$env.WAFTESTER_SSE_URL`
- **Connections:**
  - **Output (tools) →** `AI Agent` (ai_tool)
- **Version-specific requirements:** Node typeVersion `1`.
- **Edge cases / failures:**
  - Wrong/missing SSE URL → connection refused, DNS failure, or timeouts.
  - If WAFtester container isn’t on the same Docker network as n8n, `http://waftester:8080/sse` will not resolve.
  - SSE proxies/load balancers may terminate idle connections; tool discovery/calls can fail.
  - If WAFtester MCP server is not started in MCP/SSE mode, tools won’t be discoverable.
- **Sub-workflow reference:** None (external service dependency: WAFtester MCP server).

---

### Block 3 — Documentation / On-Canvas Notes
**Overview:**  
Two Sticky Notes document how the workflow works and how to set it up. They do not affect execution.

**Nodes Involved:**
- **Sticky Note**
- **Sticky Note1**

#### Node: Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — Canvas annotation.
- **Configuration choices (interpreted):**
  - Contains “How it works”, setup steps (Docker run command), and customization tips.
- **Connections:** None.
- **Version-specific requirements:** typeVersion `1`.
- **Edge cases / failures:** None (non-executing).

#### Node: Sticky Note1
- **Type / role:** `n8n-nodes-base.stickyNote` — Canvas annotation.
- **Configuration choices (interpreted):**
  - Describes the agent chain: Chat Trigger → AI Agent → WAFtester tools, with OpenAI as reasoning.
- **Connections:** None.
- **Version-specific requirements:** typeVersion `1`.
- **Edge cases / failures:** None (non-executing).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Chat Trigger | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point / receives user messages | — | AI Agent | ### How it works; Setup steps (Docker + OpenAI creds + env var); Customization tips (example prompts). / ## AI Agent chain: Chat Trigger → AI Agent → WAFtester MCP with OpenAI reasoning. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates reasoning and tool calls | Chat Trigger; OpenAI Chat Model; WAFtester MCP | (Chat response back through chat UI) | ### How it works; Setup steps (Docker + OpenAI creds + env var); Customization tips (example prompts). / ## AI Agent chain: Chat Trigger → AI Agent → WAFtester MCP with OpenAI reasoning. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for the agent | — | AI Agent | ### How it works; Setup steps (Docker + OpenAI creds + env var); Customization tips (example prompts). / ## AI Agent chain: Chat Trigger → AI Agent → WAFtester MCP with OpenAI reasoning. |
| WAFtester MCP | @n8n/n8n-nodes-langchain.mcpClientTool | MCP tool connector (SSE) to WAFtester tools | — | AI Agent | ### How it works; Setup steps (Docker + OpenAI creds + env var); Customization tips (example prompts). / ## AI Agent chain: Chat Trigger → AI Agent → WAFtester MCP with OpenAI reasoning. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ### How it works\n\nChat with an AI agent that has access to WAFtester's 18 security testing tools via MCP (Model Context Protocol).\n\n**Chat Trigger** receives your message, **AI Agent** decides which tools to call, **WAFtester MCP** executes the tests and returns results.\n\nThe agent follows a standard testing workflow:\n1. Detect the WAF vendor protecting the target\n2. Discover endpoints and parameters\n3. Run attack payload scans across 18 categories\n4. Find WAF bypass techniques\n5. Generate a formal security assessment with grading\n\nAsync operations (scans, assessments) are automatically polled.\n\n### Setup steps\n\n1. Start WAFtester MCP server:\n```\ndocker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080\n```\n2. Add OpenAI credentials: **Settings → Credentials → New → OpenAI API**\n3. Click the OpenAI Chat Model node and select your credential\n4. If WAFtester runs on a different host, set `WAFTESTER_SSE_URL` env variable\n5. Activate the workflow and open the chat interface\n\n### Customization tips\n\n- Try: \"Scan https://example.com for SQLi and XSS\"\n- Try: \"Find WAF bypasses for https://example.com\"\n- Customize the system prompt in the AI Agent node |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## AI Agent chain\n\nChat Trigger sends messages to the AI Agent, which calls WAFtester MCP tools with OpenAI as the reasoning layer. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **“Test WAF security interactively with AI agent and WAFtester MCP”**
   - Keep it **Inactive** until setup is complete.

2. **Add “Chat Trigger”**
   - Node type: **LangChain → Chat Trigger**
   - Leave defaults.
   - This is the workflow entry point.

3. **Add “AI Agent”**
   - Node type: **LangChain → AI Agent**
   - In **Options / System Message**, paste a system prompt that:
     - Identifies it as a WAF testing assistant using WAFtester tools
     - Enforces the sequence: `detect_waf → discover → learn → scan → bypass → assess`
     - Mentions async tools return `task_id` and polling with `get_task_status`
     - Lists supported attack categories (as in the provided workflow)

4. **Add “OpenAI Chat Model”**
   - Node type: **LangChain → OpenAI Chat Model**
   - Create credentials:
     - n8n: **Settings → Credentials → New → OpenAI API**
     - Provide your OpenAI API key (and org/project details if required).
   - Select that credential in the node.
   - (Recommended) Select a strong model (the note suggests **GPT-4o**).

5. **Add “WAFtester MCP”**
   - Node type: **LangChain → MCP Client Tool**
   - Configure **SSE Endpoint**:
     - Use an expression like: `{{ $env.WAFTESTER_SSE_URL || 'http://waftester:8080/sse' }}`
   - Ensure your WAFtester MCP server is reachable at that URL.

6. **Wire the nodes**
   - Connect **Chat Trigger (main)** → **AI Agent (main)**
   - Connect **OpenAI Chat Model (ai_languageModel)** → **AI Agent (ai_languageModel)**
   - Connect **WAFtester MCP (ai_tool)** → **AI Agent (ai_tool)**

7. **Start WAFtester MCP server (external dependency)**
   - Example Docker command (as documented in the workflow note):
     - `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080`
   - If n8n runs in Docker separately:
     - Put both containers on the same Docker network and ensure hostname/port routing works, **or**
     - Set `WAFTESTER_SSE_URL` to a reachable host (e.g., `http://localhost:8080/sse` from n8n’s perspective, not your laptop’s).

8. **Activate and test**
   - Activate the workflow.
   - Open the n8n chat interface and try:
     - “Detect the WAF on https://example.com”
     - “Scan https://example.com for SQLi and XSS”
     - “Find WAF bypasses for https://example.com”

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Start WAFtester MCP server: `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080` | Required external service for MCP tools |
| Configure OpenAI credentials in n8n: **Settings → Credentials → New → OpenAI API** | Required for the OpenAI Chat Model node |
| If WAFtester runs on a different host, set `WAFTESTER_SSE_URL` | Used by the WAFtester MCP node: `$env.WAFTESTER_SSE_URL` fallback to `http://waftester:8080/sse` |
| Only test targets you have authorization to test | Stated in workflow description / implied security constraint |