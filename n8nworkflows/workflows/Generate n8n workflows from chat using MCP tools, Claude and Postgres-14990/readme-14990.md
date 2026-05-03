Generate n8n workflows from chat using MCP tools, Claude and Postgres

https://n8nworkflows.xyz/workflows/generate-n8n-workflows-from-chat-using-mcp-tools--claude-and-postgres-14990


# Generate n8n workflows from chat using MCP tools, Claude and Postgres

Let me now produce the comprehensive document.# Generate n8n Workflows from Chat Using MCP Tools, Claude and Postgres

---

## 1. Workflow Overview

This workflow implements an **AI-powered n8n workflow architect agent** that receives natural-language chat requests, uses an MCP (Model Context Protocol) tool server to discover, validate, and deploy n8n workflows, and returns results back to the user via chat. It persists conversation history in a PostgreSQL database so that multi-turn interactions (refinements, modifications) maintain context.

The logic is organized into three functional blocks:

| Block | Name | Purpose |
|-------|------|---------|
| **1** | Input Reception | Receives incoming chat messages via a webhook-based chat trigger and routes them to the AI Agent. |
| **2** | AI Processing & Memory | The AI Agent orchestrates a multi-phase workflow generation pipeline (discovery → configuration → validation → build → deploy) using an OpenRouter LLM, an MCP Client for n8n tool calls, and PostgreSQL-backed chat memory for session continuity. |
| **3** | Output Delivery | Formats the agent's response and sends it back to the chat client. |

---

## 2. Block-by-Block Analysis

---

### Block 1 — Input Reception

**Overview:**  
This block captures user chat messages through a webhook endpoint and passes them directly to the AI Agent for processing. It uses n8n's built-in LangChain chat trigger, which handles session management and provides a `sessionId` that downstream nodes use for memory scoping.

**Nodes Involved:**  
- Chat Trigger

**Node Details:**

#### Chat Trigger
| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.chatTrigger` |
| **Technical Role** | Webhook-based entry point; listens for incoming chat messages and emits them as workflow items. |
| **Configuration** | Response mode is set to **"responseNodes"**, meaning the workflow will use a dedicated "Respond To Chat" node to send replies rather than automatically echoing input. Webhook ID is auto-generated (`72b1e899-fcf5-4b48-9672-0fd496d88165`). |
| **Key Expressions / Variables** | Emits `json.sessionId` — used by Postgres Chat Memory to scope conversation history per session. Also emits `json.message` (the user's text) consumed by the AI Agent. |
| **Input Connections** | None (entry node). |
| **Output Connections** | Main output → AI Agent. |
| **Version** | 1.4 |
| **Edge Cases / Failures** | - Webhook URL must be reachable from the chat client (network/firewall issues). <br> - If the chat client disconnects before the response is sent, the response will be lost (no retry mechanism). <br> - The `responseNodes` mode requires that the "Respond To Chat" node always executes; if the workflow errors before reaching it, the user receives no reply. |

---

### Block 2 — AI Processing & Memory

**Overview:**  
This is the core intelligence block. The **AI Agent** node runs a detailed 7-phase system prompt that enforces strict tool-driven workflow generation (no guessing, full validation before deployment). It is powered by three sub-nodes: an **OpenRouter Chat Model** for LLM inference, an **MCP Client** that exposes n8n-specific tools (search nodes, validate, create workflows, etc.), and **Postgres Chat Memory** for per-session conversation history.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- MCP Client  
- Postgres Chat Memory

**Node Details:**

#### AI Agent
| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.agent` |
| **Technical Role** | Central orchestrator that receives the chat message, invokes tools and the LLM, manages the multi-step reasoning loop, and returns the final output. |
| **Configuration** | **Max Iterations:** 25 — allows up to 25 tool-calling rounds per request, critical for the multi-phase pipeline (discovery, config, validation, build, deploy). **Streaming:** Disabled — responses are returned as a single payload to the Respond To Chat node. **System Message:** A comprehensive 7-phase prompt (detailed below). |
| **System Message Phases** | 1. **Always Start Here** — calls `tools_documentation()` first, asks one clarifying question if needed. <br> 2. **Discovery** — uses `search_nodes`, `list_nodes`, `list_ai_tools`, `get_node_for_task` to find correct node types; parallel calls encouraged. <br> 3. **Configuration** — uses `get_node_essentials`, `search_node_properties`, `get_node_documentation` to determine exact parameters; never invents parameters. <br> 4. **Pre-Validation** — uses `validate_node_minimal` and `validate_node_operation` on each node config before assembly. <br> 5. **Build** — assembles workflow JSON with strict rules (unique IDs, correct typeVersion, 220px spacing, left-to-right layout starting at [240,300], no hardcoded credentials). <br> 6. **Post-Build Validation** — runs three separate checks: `validate_workflow`, `validate_workflow_connections`, `validate_workflow_expressions`; fixes errors before proceeding. <br> 7. **Deploy** — uses `n8n_create_workflow`, then `n8n_validate_workflow` on the deployed ID; patches with `n8n_update_partial_workflow` if needed. |
| **Output Rules (from system message)** | After successful deployment, replies with: (1) workflow URL, (2) bullet list of credentials the user must add, (3) one confirmation sentence. Never outputs raw JSON, code blocks, or internal technical details. |
| **Modification Rules** | For existing workflow changes: asks for workflow ID, uses `n8n_update_partial_workflow` with diffs, re-validates after every update. |
| **Absolute Rules** | Never guess node types; never skip `tools_documentation`; never deploy without all 3 post-build validations; never use Code node if a standard node suffices; never show raw JSON; never hardcode credentials. |
| **Key Expressions / Variables** | Consumes the output of all three sub-nodes. Produces `json.output` consumed by Respond To Chat. |
| **Input Connections** | Main input ← Chat Trigger; ai_languageModel ← OpenRouter Chat Model; ai_tool ← MCP Client; ai_memory ← Postgres Chat Memory. |
| **Output Connections** | Main output → Respond To Chat. |
| **Version** | 3.1 |
| **Edge Cases / Failures** | - If the MCP server is unreachable, all tool calls fail and the agent cannot discover/validate/deploy workflows. <br> - Hitting the 25-iteration limit before completing all phases will result in a truncated response. <br> - If the LLM hallucinates a tool call or node type despite the system prompt, the MCP server will return an error; the agent should recover within remaining iterations. <br> - Expression syntax errors in generated workflows are caught by `validate_workflow_expressions` in Phase 6. |

#### OpenRouter Chat Model
| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` |
| **Technical Role** | Provides LLM inference via the OpenRouter API, serving as the language model for the AI Agent's reasoning and tool-calling loop. |
| **Configuration** | **Model:** `openai/gpt-4.1-nano` — a lightweight, fast model chosen for cost efficiency and speed in tool-heavy loops. No additional model options configured. |
| **Credentials** | OpenRouter API credential (named "OpenRouter account 2"). Requires a valid OpenRouter API key with sufficient credits/quota. |
| **Key Expressions / Variables** | None — static configuration. |
| **Input Connections** | None (sub-node, connected via `ai_languageModel` to AI Agent). |
| **Output Connections** | ai_languageModel → AI Agent. |
| **Version** | 1 |
| **Edge Cases / Failures** | - API key expiration or quota exhaustion causes all LLM calls to fail. <br> - Model `openai/gpt-4.1-nano` may be deprecated or renamed on OpenRouter; the workflow will fail if the model identifier becomes invalid. <br> - Rate limiting on OpenRouter may cause intermittent failures during high-iteration runs. <br> - No fallback model is configured; a single point of failure. |

#### MCP Client
| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.mcpClientTool` |
| **Technical Role** | Acts as a tool provider for the AI Agent by connecting to a remote MCP (Model Context Protocol) server that exposes n8n-specific tools for node discovery, validation, workflow creation, and deployment. |
| **Configuration** | **Endpoint URL:** `http://n8n-mcp:3000/mcp` — assumes the MCP server is running as a Docker service or sidecar accessible at hostname `n8n-mcp` on port 3000. **Authentication:** Bearer Auth — uses an HTTP Bearer Auth credential. **Output Log Routing:** `ai_tool` — tool call logs are routed back to the AI Agent for observability. No additional options configured. |
| **Credentials** | HTTP Bearer Auth credential (named "N8n MCP Bearer"). The bearer token must match the one expected by the MCP server. |
| **Key Expressions / Variables** | None — static configuration. |
| **Input Connections** | None (sub-node, connected via `ai_tool` to AI Agent). |
| **Output Connections** | ai_tool → AI Agent. |
| **Version** | 1.2 |
| **Edge Cases / Failures** | - The MCP server must be running and accessible at `http://n8n-mcp:3000/mcp`; DNS resolution failure or service downtime blocks all tool calls. <br> - Bearer token mismatch results in 401 Unauthorized errors. <br> - If the MCP server exposes tools that differ from what the system prompt expects (e.g., renamed or removed tools), the agent will encounter tool-not-found errors. <br> - Network latency to the MCP server may slow down iterations significantly, especially with 25 max iterations. |

#### Postgres Chat Memory
| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.memoryPostgresChat` |
| **Technical Role** | Persists the conversation history per session in a PostgreSQL database, enabling multi-turn context for follow-up requests and modifications. |
| **Configuration** | **Session ID Type:** `customKey` — uses a custom key rather than a generated or fixed ID. **Session Key Expression:** `={{ $('Chat Trigger').item.json.sessionId }}` — scopes memory to the chat session ID provided by the Chat Trigger. **Context Window Length:** 10 — retains the last 10 message exchanges in the conversation context sent to the LLM. |
| **Credentials** | PostgreSQL credential (named "Postgres account"). Requires a running PostgreSQL instance with appropriate tables created (n8n's LangChain memory node auto-creates the required schema on first run). |
| **Key Expressions / Variables** | `$('Chat Trigger').item.json.sessionId` — references the session ID from the triggering chat message. |
| **Input Connections** | None (sub-node, connected via `ai_memory` to AI Agent). |
| **Output Connections** | ai_memory → AI Agent. |
| **Version** | 1.3 |
| **Edge Cases / Failures** | - If the PostgreSQL database is unreachable, memory operations fail and the agent loses conversation context (it will still function but without history). <br> - The `sessionId` expression depends on the Chat Trigger always providing a `sessionId`; if this field is missing, memory will fail to initialize or may mix sessions. <br> - With a context window of 10, very long multi-turn conversations will lose early context; the agent may ask the user to repeat information. <br> - Database migration/schema creation requires the Postgres user to have CREATE TABLE permissions on first run. |

---

### Block 3 — Output Delivery

**Overview:**  
This block takes the final output from the AI Agent and delivers it back to the chat client. It uses a fallback expression to ensure the user always receives a meaningful response, even if the agent's output is unexpectedly empty.

**Nodes Involved:**  
- Respond To Chat

**Node Details:**

#### Respond To Chat
| Attribute | Detail |
|-----------|--------|
| **Type** | `@n8n/n8n-nodes-langchain.chat` |
| **Technical Role** | Sends the AI Agent's response text back to the chat client that originated the request. This is the designated response node required by the Chat Trigger's `responseNodes` mode. |
| **Configuration** | **Message Expression:** `={{ $json.output ?? "Something went wrong. Please try rephrasing your request." }}` — uses the agent's `output` field if present, otherwise falls back to a generic error message. No additional options configured. Webhook ID is auto-generated (`3704ed64-56da-44d9-a85d-3289b54646bb`). |
| **Key Expressions / Variables** | `$json.output` — the text output from the AI Agent; `??` nullish coalescing provides the fallback string. |
| **Input Connections** | Main input ← AI Agent. |
| **Output Connections** | None (terminal node). |
| **Version** | 1.3 |
| **Edge Cases / Failures** | - If the AI Agent produces no `output` field (e.g., due to an error or iteration limit), the fallback message is sent instead. <br> - If this node fails to execute (e.g., webhook delivery failure), the user receives no response despite the Chat Trigger being in `responseNodes` mode. <br> - The webhook ID must match the session context managed by n8n's chat infrastructure. |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|-----------------|-------------|
| Chat Trigger | `@n8n/n8n-nodes-langchain.chatTrigger` | Webhook entry point for incoming chat messages; emits sessionId for memory scoping | — | AI Agent | **Sticky Note1:** "Initiate chat processing — Starts the workflow in response to chat messages and passes it to the AI Agent for processing." |
| AI Agent | `@n8n/n8n-nodes-langchain.agent` | Central orchestrator executing a 7-phase workflow generation pipeline using tools, LLM, and memory | Chat Trigger (main), OpenRouter Chat Model (ai_languageModel), MCP Client (ai_tool), Postgres Chat Memory (ai_memory) | Respond To Chat | **Sticky Note1:** "Initiate chat processing — Starts the workflow in response to chat messages and passes it to the AI Agent for processing." |
| OpenRouter Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | LLM inference provider via OpenRouter (model: openai/gpt-4.1-nano) | — | AI Agent (ai_languageModel) | **Sticky Note2:** "Handle AI and memory tasks — Processes the chat message using AI services and manages chat memory." |
| MCP Client | `@n8n/n8n-nodes-langchain.mcpClientTool` | Tool provider connecting to MCP server for n8n node discovery, validation, and workflow deployment | — | AI Agent (ai_tool) | **Sticky Note2:** "Handle AI and memory tasks — Processes the chat message using AI services and manages chat memory." |
| Postgres Chat Memory | `@n8n/n8n-nodes-langchain.memoryPostgresChat` | Persists per-session conversation history in PostgreSQL (context window: 10 exchanges) | — | AI Agent (ai_memory) | **Sticky Note2:** "Handle AI and memory tasks — Processes the chat message using AI services and manages chat memory." |
| Respond To Chat | `@n8n/n8n-nodes-langchain.chat` | Delivers the AI Agent's response back to the chat client with fallback error message | AI Agent | — | **Sticky Note3:** "Output chat results — Outputs the final processed chat result to the chat client." |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation: workflow overview, how it works, setup steps, customization | — | — | "n8n Workflow Generator Agent with MCP + Claude - Auto-Build & Deploy — How it works: 1. The Chat Trigger node initiates the workflow in response to incoming chat messages. 2. The workflow passes the message to the AI Agent for processing. 3. The AI Agent uses sub-nodes to interact with external services and models. 4. The MCP Client and OpenRouter Chat Model sub-nodes handle the AI processing tasks. 5. Postgres Chat Memory records conversation history. 6. The Chat node finalizes the interaction and outputs the result. — Setup steps: [ ] Configure AI Agent with appropriate AI model credentials. [ ] Set up database access for Postgres Chat Memory. [ ] Ensure that the Chat Trigger is connected to a chat client. — Customization: Customize the AI model and connection settings in the AI Agent node to change the AI's behavior or add new functionalities." |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation: describes input reception block | — | — | "Initiate chat processing — Starts the workflow in response to chat messages and passes it to the AI Agent for processing." |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation: describes AI and memory processing block | — | — | "Handle AI and memory tasks — Processes the chat message using AI services and manages chat memory." |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation: describes output delivery block | — | — | "Output chat results — Outputs the final processed chat result to the chat client." |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Documentation: video setup guide link | — | — | "📺 Watch the Setup Guide — @[youtube](HMXBInLaUB8)" |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Generate n8n workflows from chat using MCP tools, Claude and Postgres".

2. **Add the Chat Trigger node:**
   - Node type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Position: approximately `[-80, 0]`
   - Configuration: Set **Options → Response Mode** to `responseNodes`.
   - No credentials required. A webhook ID will be auto-generated upon saving.

3. **Add the AI Agent node:**
   - Node type: `@n8n/n8n-nodes-langchain.agent`
   - Position: approximately `[208, 0]`
   - Configuration:
     - **Max Iterations:** `25`
     - **Streaming:** Disabled
     - **System Message:** Paste the full 7-phase system message (see Block 2 — AI Agent details above for the complete text covering Phase 1 through Phase 7, Output Rules, Modification Rules, and Absolute Rules).
   - No credentials required at this node level.

4. **Connect Chat Trigger → AI Agent:**
   - Draw a **main** connection from Chat Trigger's output to AI Agent's main input.

5. **Add the OpenRouter Chat Model sub-node:**
   - Node type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
   - Position: approximately `[80, 448]`
   - Configuration:
     - **Model:** `openai/gpt-4.1-nano`
     - **Options:** Leave defaults (no additional settings).
   - Credentials: Create an **OpenRouter API** credential with a valid API key. Name it for easy identification (e.g., "OpenRouter account").
   - Connect this node's **ai_languageModel** output to the AI Agent's **ai_languageModel** input.

6. **Add the MCP Client sub-node:**
   - Node type: `@n8n/n8n-nodes-langchain.mcpClientTool`
   - Position: approximately `[400, 464]`
   - Configuration:
     - **Endpoint URL:** `http://n8n-mcp:3000/mcp`
     - **Authentication:** `bearerAuth`
     - **Options:** Leave defaults.
     - **Output Log Routing (rewireOutputLogTo):** `ai_tool`
   - Credentials: Create an **HTTP Bearer Auth** credential with the bearer token expected by your MCP server. Name it (e.g., "N8n MCP Bearer").
   - Connect this node's **ai_tool** output to the AI Agent's **ai_tool** input.

7. **Add the Postgres Chat Memory sub-node:**
   - Node type: `@n8n/n8n-nodes-langchain.memoryPostgresChat`
   - Position: approximately `[224, 448]`
   - Configuration:
     - **Session ID Type:** `customKey`
     - **Session Key:** `={{ $('Chat Trigger').item.json.sessionId }}`
     - **Context Window Length:** `10`
   - Credentials: Create a **PostgreSQL** credential pointing to your Postgres instance with connection details (host, port, database, user, password). The database user needs CREATE TABLE permissions for initial schema creation. Name it (e.g., "Postgres account").
   - Connect this node's **ai_memory** output to the AI Agent's **ai_memory** input.

8. **Add the Respond To Chat node:**
   - Node type: `@n8n/n8n-nodes-langchain.chat`
   - Position: approximately `[704, 0]`
   - Configuration:
     - **Message:** `={{ $json.output ?? "Something went wrong. Please try rephrasing your request." }}`
     - **Options:** Leave defaults.
   - No credentials required. A webhook ID will be auto-generated.

9. **Connect AI Agent → Respond To Chat:**
   - Draw a **main** connection from AI Agent's output to Respond To Chat's main input.

10. **Add documentation sticky notes (optional but recommended):**
    - **Sticky Note** at `[-624, -208]`, size 480×512: Copy the full overview text including "How it works", "Setup steps", and "Customization" sections.
    - **Sticky Note1** at `[-112, -208]`, size 720×512, color 7: "Initiate chat processing — Starts the workflow in response to chat messages and passes it to the AI Agent for processing."
    - **Sticky Note2** at `[32, 320]`, size 480×304, color 7: "Handle AI and memory tasks — Processes the chat message using AI services and manages chat memory."
    - **Sticky Note3** at `[640, -144]`, height 320, color 7: "Output chat results — Outputs the final processed chat result to the chat client."
    - **Sticky Note6** at `[976, -144]`, size 496×320: "📺 Watch the Setup Guide — @[youtube](HMXBInLaUB8)"

11. **Verify all connections:**
    - Chat Trigger → AI Agent (main)
    - AI Agent → Respond To Chat (main)
    - OpenRouter Chat Model → AI Agent (ai_languageModel)
    - MCP Client → AI Agent (ai_tool)
    - Postgres Chat Memory → AI Agent (ai_memory)

12. **Set up the MCP server externally:**
    - Deploy the n8n MCP server as a Docker container or sidecar service accessible at hostname `n8n-mcp` on port `3000`.
    - Ensure the bearer token configured in the MCP Client credential matches the server's expected token.
    - Verify that the MCP server exposes the tools referenced in the system prompt: `tools_documentation`, `search_nodes`, `list_nodes`, `list_ai_tools`, `get_node_for_task`, `get_node_essentials`, `search_node_properties`, `get_node_documentation`, `validate_node_minimal`, `validate_node_operation`, `validate_workflow`, `validate_workflow_connections`, `validate_workflow_expressions`, `n8n_create_workflow`, `n8n_validate_workflow`, `n8n_update_partial_workflow`.

13. **Test the workflow:**
    - Save and activate the workflow.
    - Open the Chat Trigger's chat URL in a browser or connect a chat client.
    - Send a test message (e.g., "Create a simple webhook workflow").
    - Verify the agent calls `tools_documentation()` first, then proceeds through the discovery and build phases, and finally returns the workflow URL and credential list.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The MCP server must be deployed and accessible before the workflow can function. It is not bundled with n8n and must be set up as a separate service (Docker container recommended) at `http://n8n-mcp:3000/mcp`. | Infrastructure dependency |
| The OpenRouter model `openai/gpt-4.1-nano` is chosen for cost and speed. For more capable reasoning on complex workflows, consider switching to a more powerful model (e.g., `anthropic/claude-sonnet-4`), but be aware of higher cost and potential iteration timeout issues. | Model selection guidance |
| The Postgres Chat Memory node auto-creates its schema on first execution. The database user must have CREATE TABLE permissions initially. After creation, SELECT/INSERT/UPDATE permissions suffice. | Database setup |
| The context window of 10 exchanges means long refinement sessions may lose early context. Increase `contextWindowLength` if multi-turn modifications are common. | Memory tuning |
| The AI Agent's max iterations of 25 is critical for the 7-phase pipeline. Complex workflows (many nodes) may require more iterations. Increase cautiously as each iteration consumes LLM tokens. | Performance consideration |
| Setup guide video available on YouTube | [https://www.youtube.com/watch?v=HMXBInLaUB8](https://www.youtube.com/watch?v=HMXBInLaUB8) |
| Workflow branding: "n8n Workflow Generator Agent with MCP + Claude - Auto-Build & Deploy" | Sticky Note title |
| The system message strictly prohibits outputting raw JSON or code blocks to the user. This is a UX design choice to keep responses clean and actionable. | Agent behavior rule |
| When modifying existing workflows, the agent prefers `n8n_update_partial_workflow` with diff operations over recreating the full workflow, minimizing disruption. | Modification best practice |