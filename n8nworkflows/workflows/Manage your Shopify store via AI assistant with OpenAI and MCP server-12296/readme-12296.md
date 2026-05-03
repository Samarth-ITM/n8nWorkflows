Manage your Shopify store via AI assistant with OpenAI and MCP server

https://n8nworkflows.xyz/workflows/manage-your-shopify-store-via-ai-assistant-with-openai-and-mcp-server-12296


# Manage your Shopify store via AI assistant with OpenAI and MCP server

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow builds an AI assistant that can manage a Shopify store by calling Shopify “tools” through an **MCP server** (Model Context Protocol) and an **MCP client tool** inside an n8n LangChain Agent. It also supports multi-channel outbound messaging (Gmail, Telegram, Discord, and a custom “Rapiwa” node) and includes an error path.

**Primary use cases**
- Chat-based store operations: create/update/delete products and orders, search products, list orders, retrieve fulfilled orders.
- Centralized “AI agent” that interprets user chat messages and decides which Shopify tool to call.
- Send the AI’s final response to one or more messaging channels.

**Logical blocks**
1.1 **Shopify MCP Server (tool exposure layer)**  
Publishes multiple Shopify operations as callable tools to an MCP endpoint.

1.2 **Chat-driven AI Agent (reasoning + tool use)**  
Receives chat messages, uses an OpenAI chat model, maintains memory, and calls the Shopify MCP client tool when needed.

1.3 **Post-processing + Routing**  
Takes the model output and routes it through an IF node to either send messages (success path) or stop with an error (failure path).

1.4 **Outbound messaging channels**  
Sends the assistant’s response via Gmail / Telegram / Discord / Rapiwa.

---

## 2. Block-by-Block Analysis

### 2.1 Shopify MCP Server (tool exposure layer)

**Overview:**  
This block defines an MCP server trigger node that exposes a set of Shopify tool nodes (products and orders operations). External MCP clients (or internal ones via the MCP Client Tool) can invoke these tool functions.

**Nodes involved:**
- Shopify MCP Server
- Create a product in Shopify
- Get a product in Shopify
- Search products in Shopify by Title
- Get All products in Shopify
- Update a product in Shopify
- Delete a product in Shopify
- Update a product catagoirs
- Create an order in Shopify
- Get an order in Shopify
- Get all orders in Shopify
- Get fulfilled orders in Shopify
- Update an order in Shopify
- Delete an order in Shopify
- Sticky Note, Sticky Note1, Sticky Note2, Sticky Note5 (empty content, but still documented)

#### Node: **Shopify MCP Server**
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpTrigger` (v2) — MCP server entry point that receives MCP tool calls and dispatches them to connected “ai_tool” nodes.
- **Configuration (interpreted):** Uses an n8n webhook-style endpoint (webhookId present). Actual MCP endpoint URL is determined by n8n environment.
- **Connections:**
  - **Inputs:** This node is an entry point (trigger).
  - **Outputs:** Receives tool executions from connected Shopify tool nodes via `ai_tool` connections (tool nodes connect *into* it).
- **Version considerations:** Requires n8n with LangChain MCP nodes available and enabled.
- **Edge cases / failures:**
  - MCP connectivity issues (endpoint not reachable, wrong base URL, invalid auth if applied).
  - Tool schema mismatch (if MCP client sends parameters not matching what the tool expects).
  - Shopify API rate limits or auth failures surfaced through the tool nodes.

#### Nodes: **Shopify Tool nodes (Products + Orders)**
All of these are `n8n-nodes-base.shopifyTool` (v1) nodes, each connected via **`ai_tool`** to **Shopify MCP Server**.

Because the JSON has **empty `parameters` objects**, the specific operation fields (resource, operation, filters, IDs) are not visible here. However, the **node names indicate intended operations**, and the MCP server will expose them as tools with corresponding names/descriptions.

Below are the nodes and their intended roles:

- **Create a product in Shopify**
  - **Role:** Tool for creating a Shopify product.
  - **Common requirements:** Shopify credentials (Admin API access token/private app/custom app), product payload (title, variants, etc.).
  - **Failures:** validation errors, missing required fields, auth errors.

- **Get a product in Shopify**
  - **Role:** Fetch a product (typically by product ID/handle).
  - **Failures:** product not found, invalid ID, auth errors.

- **Search products in Shopify by Title**
  - **Role:** Search products using title query.
  - **Failures:** empty results, query syntax issues, rate limit.

- **Get All products in Shopify**
  - **Role:** List products (pagination likely).
  - **Failures:** pagination handling, large result sets, rate limit.

- **Update a product in Shopify**
  - **Role:** Update product fields.
  - **Failures:** invalid updates, concurrency conflicts, auth errors.

- **Delete a product in Shopify**
  - **Role:** Delete product by ID.
  - **Failures:** product not found, permissions.

- **Update a product catagoirs** (typo in name)
  - **Role:** Likely intended to update product categories/collections/taxonomy.
  - **Failures:** depends on what “categories” maps to in the configured Shopify operation; could fail if taxonomy endpoints differ by API version.

- **Create an order in Shopify**
  - **Role:** Create order (usually draft order or order creation depending on API).
  - **Failures:** payment/financial status constraints, customer/address validation.

- **Get an order in Shopify**
  - **Role:** Fetch order details by order ID.
  - **Failures:** order not found, permissions.

- **Get all orders in Shopify**
  - **Role:** List orders (pagination, filters).
  - **Failures:** large dataset, rate limit.

- **Get fulfilled orders in Shopify**
  - **Role:** List orders filtered by fulfillment status.
  - **Failures:** filtering mismatch depending on API params.

- **Update an order in Shopify**
  - **Role:** Update order fields (note Shopify limits what can be updated on orders).
  - **Failures:** Shopify restrictions on mutable fields, invalid status transitions.

- **Delete an order in Shopify**
  - **Role:** Delete/cancel order (Shopify has constraints; “delete” may be archival/cancel operation depending on tool config).
  - **Failures:** operation not permitted, order already fulfilled, permissions.

**Sticky notes in this block (all empty):**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note5  
Since their `content` is empty, they add no operational documentation.

---

### 2.2 Chat-driven AI Agent (reasoning + tool use)

**Overview:**  
This block receives a chat message, runs it through an Agent powered by OpenAI, uses conversation memory, and can call Shopify tools via an MCP client tool.

**Nodes involved:**
- When chat message received
- AI BOT
- OpenAI Model
- Memory
- Shopify MCP Client
- Sticky Note4 (empty)

#### Node: **When chat message received**
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` (v1.1) — entry point for chat-based interactions.
- **Configuration (interpreted):** Uses an n8n webhookId. In practice, it exposes a chat endpoint/UI integration depending on your n8n setup.
- **Connections:**
  - **Output (main):** to **AI BOT**
- **Edge cases / failures:**
  - No message payload (unexpected webhook calls).
  - Channel/user metadata missing if not provided by the chat source.

#### Node: **AI BOT**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (v2) — orchestrates reasoning, tool calling, and response generation.
- **Configuration (interpreted):**
  - Uses **OpenAI Model** via `ai_languageModel`.
  - Uses **Memory** via `ai_memory` (buffer window).
  - Uses **Shopify MCP Client** via `ai_tool` (so it can call MCP-exposed Shopify tools).
- **Connections:**
  - **Input (main):** from **When chat message received**
  - **Output (main):** to **Message a model** (acts as a post-agent step)
  - **Inputs (AI ports):**
    - `ai_languageModel` from **OpenAI Model**
    - `ai_memory` from **Memory**
    - `ai_tool` from **Shopify MCP Client**
- **Edge cases / failures:**
  - Tool call loops or unclear tool selection if the system prompt/instructions are not configured.
  - MCP tool errors bubbling up (Shopify auth/rate limits).
  - Token limits if memory window is large or Shopify responses are big.

#### Node: **OpenAI Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) — provides the chat LLM used by the agent.
- **Configuration (interpreted):** Credentials for OpenAI; model name/temperature not visible (parameters empty in JSON).
- **Connections:**
  - Output `ai_languageModel` → **AI BOT**
- **Edge cases / failures:**
  - Invalid OpenAI API key / org.
  - Model not available, rate limits, timeouts.
  - Response formatting issues if downstream expects a particular structure.

#### Node: **Memory**
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` (v1.3) — keeps last N messages as context.
- **Configuration (interpreted):** Window size not shown (empty parameters), defaults apply.
- **Connections:**
  - Output `ai_memory` → **AI BOT**
- **Edge cases / failures:**
  - Memory not persisted across executions depending on n8n settings (some memory nodes are execution-scoped unless configured otherwise).
  - Too small window causes loss of context; too large increases token usage.

#### Node: **Shopify MCP Client**
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpClientTool` (v1) — a tool the agent can use to call the **Shopify MCP Server**.
- **Configuration (interpreted):** Needs MCP server endpoint details; not visible in parameters.
- **Connections:**
  - Output `ai_tool` → **AI BOT**
- **Edge cases / failures:**
  - MCP server URL misconfigured.
  - Tool discovery issues (client can’t list tools, schema mismatch).
  - Authentication to MCP server if required (not shown here).

**Sticky note in this block (empty):**
- Sticky Note4 — no content.

---

### 2.3 Post-processing + Routing

**Overview:**  
After the agent produces a response, the workflow runs an extra OpenAI “Message a model” step, then uses an IF node to route either to outbound messaging nodes (success) or to Stop and Error (failure).

**Nodes involved:**
- Message a model
- If
- Stop and Error

#### Node: **Message a model**
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (v1.8) — a direct OpenAI messaging node (often used to format/refine output).
- **Configuration (interpreted):** Not shown (empty parameters). Typically contains prompt/instructions and selects model/credentials.
- **Connections:**
  - Input (main) from **AI BOT**
  - Output (main) to **If**
- **Edge cases / failures:**
  - If this node expects a field that the Agent doesn’t output (expression mapping issues).
  - OpenAI auth/rate limits/timeouts.

#### Node: **If**
- **Type / role:** `n8n-nodes-base.if` (v2.2) — conditional routing.
- **Configuration (interpreted):** Conditions not shown (empty parameters), so its logic cannot be precisely inferred. It likely checks whether the model output indicates success vs error/needs escalation.
- **Connections:**
  - Input (main) from **Message a model**
  - Output **true branch (index 0)** → Send a message2 (Gmail), Send a message (Discord), Send a text message (Telegram), Rapiwa
  - Output **false branch (index 1)** → Stop and Error
- **Edge cases / failures:**
  - Condition misconfigured leading to always-true or always-false behavior.
  - Type mismatch (string vs boolean) in evaluated expressions.

#### Node: **Stop and Error**
- **Type / role:** `n8n-nodes-base.stopAndError` (v1) — terminates execution with an error.
- **Configuration (interpreted):** Not shown (empty parameters). Often contains a static/dynamic error message.
- **Connections:**
  - Input (main) from **If** false branch
- **Edge cases / failures:**
  - If error message uses expressions referencing missing fields, it may throw expression errors.

---

### 2.4 Outbound messaging channels

**Overview:**  
On the IF “success” path, the workflow sends the assistant’s response to multiple channels in parallel: Gmail, Discord, Telegram, and a custom Rapiwa integration.

**Nodes involved:**
- Send a message2 (Gmail)
- Send a message (Discord)
- Send a text message (Telegram)
- Rapiwa
- Sticky Note3 (empty)

#### Node: **Send a message2**
- **Type / role:** `n8n-nodes-base.gmail` (v2.1) — sends an email.
- **Configuration (interpreted):** Not shown. Requires Gmail OAuth2 credentials; message fields (to/subject/body) likely mapped from AI output.
- **Connections:**
  - Input (main) from **If** true branch
- **Edge cases / failures:**
  - Gmail OAuth token expired/revoked.
  - Missing “To” address or invalid formatting.
  - Quota limits.

#### Node: **Send a message**
- **Type / role:** `n8n-nodes-base.discord` (v2) — sends a Discord message.
- **Configuration (interpreted):** Not shown. Typically uses a bot token and channel ID (or webhook).
- **Connections:**
  - Input (main) from **If** true branch
- **Edge cases / failures:**
  - Invalid token/channel permissions.
  - Message length limits.

#### Node: **Send a text message**
- **Type / role:** `n8n-nodes-base.telegram` (v1.2) — sends a Telegram message.
- **Configuration (interpreted):** Not shown. Requires Telegram bot token; chat ID.
- **Connections:**
  - Input (main) from **If** true branch
- **Edge cases / failures:**
  - Wrong chat ID, bot not started by user, blocked bot.
  - Message formatting issues (Markdown/HTML mode).

#### Node: **Rapiwa**
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` (v1) — custom/community node (unknown provider) used as an additional messaging/output channel.
- **Configuration (interpreted):** Not shown. Requires node-specific credentials/settings.
- **Connections:**
  - Input (main) from **If** true branch
- **Edge cases / failures:**
  - Node not installed on the n8n instance.
  - Breaking changes in community node versions.
  - Credential/config mismatches.

**Sticky note in this block (empty):**
- Sticky Note3 — no content.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Update a product catagoirs | n8n-nodes-base.shopifyTool | Shopify tool: update product categories (intended) | — | Shopify MCP Server (ai_tool) |  |
| Delete an order in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: delete/cancel order | — | Shopify MCP Server (ai_tool) |  |
| Update an order in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: update order | — | Shopify MCP Server (ai_tool) |  |
| Get all orders in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: list orders | — | Shopify MCP Server (ai_tool) |  |
| Get fulfilled orders in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: list fulfilled orders | — | Shopify MCP Server (ai_tool) |  |
| Get an order in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: get order details | — | Shopify MCP Server (ai_tool) |  |
| Create an order in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: create order | — | Shopify MCP Server (ai_tool) |  |
| Delete a product in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: delete product | — | Shopify MCP Server (ai_tool) |  |
| Update a product in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: update product | — | Shopify MCP Server (ai_tool) |  |
| Get All products in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: list products | — | Shopify MCP Server (ai_tool) |  |
| Search products in Shopify by Title | n8n-nodes-base.shopifyTool | Shopify tool: search products by title | — | Shopify MCP Server (ai_tool) |  |
| Get a product in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: get product | — | Shopify MCP Server (ai_tool) |  |
| Create a product in Shopify | n8n-nodes-base.shopifyTool | Shopify tool: create product | — | Shopify MCP Server (ai_tool) |  |
| Send a message | n8n-nodes-base.discord | Send response to Discord | If (true) | — |  |
| Send a text message | n8n-nodes-base.telegram | Send response to Telegram | If (true) | — |  |
| Rapiwa | n8n-nodes-rapiwa.rapiwa | Send response via custom Rapiwa node | If (true) | — |  |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Post-process/format AI response | AI BOT | If |  |
| If | n8n-nodes-base.if | Route success vs failure | Message a model | Send a message2; Send a message; Send a text message; Rapiwa; Stop and Error |  |
| Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation context memory | — | AI BOT (ai_memory) |  |
| AI BOT | @n8n/n8n-nodes-langchain.agent | AI agent + tool usage orchestration | When chat message received; OpenAI Model (ai); Memory (ai); Shopify MCP Client (ai) | Message a model |  |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | AI BOT |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| OpenAI Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | AI BOT (ai_languageModel) |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Send a message2 | n8n-nodes-base.gmail | Send response via Gmail | If (true) | — |  |
| Shopify MCP Server | @n8n/n8n-nodes-langchain.mcpTrigger | MCP server exposing Shopify tools | Tool nodes (ai_tool) | — |  |
| Shopify MCP Client | @n8n/n8n-nodes-langchain.mcpClientTool | MCP client tool callable by agent | — | AI BOT (ai_tool) |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Stop and Error | n8n-nodes-base.stopAndError | Fail-fast on IF false branch | If (false) | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Automate Shopify Store Management with AI Bot Using MCP Server”** (or your preferred name).
- Ensure n8n has LangChain nodes enabled and (if needed) install the **rapiwa** community node.

2) **Build the MCP Server tool exposure layer**
- Add node: **Shopify MCP Server** (`MCP Trigger`).
  - Configure its MCP/webhook endpoint as required by your environment (n8n will generate it).
- Add the following **Shopify Tool** nodes (`Shopify Tool`) and configure each for the intended Shopify Admin API operation (resource + operation):
  1. Create a product in Shopify
  2. Get a product in Shopify
  3. Search products in Shopify by Title
  4. Get All products in Shopify
  5. Update a product in Shopify
  6. Delete a product in Shopify
  7. Update a product catagoirs (rename to “Update product categories” if you want)
  8. Create an order in Shopify
  9. Get an order in Shopify
  10. Get all orders in Shopify
  11. Get fulfilled orders in Shopify
  12. Update an order in Shopify
  13. Delete an order in Shopify
- **Credentials:** create Shopify credentials (custom app access token + shop domain) and assign them to all Shopify tool nodes.
- **Connect each Shopify tool node to Shopify MCP Server** using the **AI Tool** connection type (`ai_tool`).

3) **Create the chat-driven AI agent block**
- Add node: **When chat message received** (`Chat Trigger`).
- Add node: **AI BOT** (`Agent`).
- Connect: **When chat message received → AI BOT** (main).
- Add node: **OpenAI Model** (`OpenAI Chat Model` / `lmChatOpenAi`).
  - Set OpenAI credentials (API key).
  - Select model (e.g., a GPT-4-class model) and temperature per your needs.
- Connect: **OpenAI Model → AI BOT** via **`ai_languageModel`**.
- Add node: **Memory** (`Memory Buffer Window`).
  - Set window size (e.g., 5–20 turns) depending on cost/context needs.
- Connect: **Memory → AI BOT** via **`ai_memory`**.
- Add node: **Shopify MCP Client** (`mcpClientTool`).
  - Configure it to point to the **Shopify MCP Server** endpoint (the MCP server URL).
  - Ensure it can list/resolve the exposed tools.
- Connect: **Shopify MCP Client → AI BOT** via **`ai_tool`**.
- In **AI BOT** configuration, add/adjust system instructions so the agent:
  - Understands it can manage Shopify via tools,
  - Asks for IDs/confirmation before destructive actions (delete),
  - Returns a clean final response text for messaging.

4) **Add post-processing and routing**
- Add node: **Message a model** (`OpenAI` LangChain node).
  - Configure it to transform the agent output into a final message format (for example: concise summary + action taken + any IDs).
  - Map input text from the AI BOT output into this node’s prompt (field mapping depends on your agent output schema).
- Connect: **AI BOT → Message a model**.
- Add node: **If**.
  - Configure condition to detect “success” vs “error”. Common patterns:
    - Check a boolean flag you set in the prior step,
    - Or check if output contains an “ERROR:” marker,
    - Or check whether tool execution returned an error field.
- Connect: **Message a model → If**.

5) **Add outbound messaging nodes (success path)**
- Add and configure:
  - **Send a message2** (`Gmail`): set To/Subject/Body (map body from Message a model output).
  - **Send a message** (`Discord`): set server/channel and message content.
  - **Send a text message** (`Telegram`): set chat ID and message content.
  - **Rapiwa**: configure credentials/endpoint per that node’s documentation.
- Connect all four nodes to the **If true branch** output.

6) **Add failure handling**
- Add node: **Stop and Error**.
  - Set a helpful error message, ideally including the model output or tool error details (careful with sensitive data).
- Connect: **If false branch → Stop and Error**.

7) **Credentials checklist**
- **Shopify:** Admin API access token + shop domain; ensure scopes cover products and orders read/write.
- **OpenAI:** API key with access to selected model.
- **Gmail:** OAuth2 in n8n; confirm consent and refresh token works.
- **Telegram:** bot token; chat ID reachable.
- **Discord:** bot token (or webhook) + permissions in target channel.
- **Rapiwa:** install node + configure required credentials.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow contains multiple Sticky Note nodes but their content is empty. | No additional embedded documentation was provided in sticky notes. |
| Shopify tool node parameters are not present in the JSON (`parameters: {}`), so you must configure each Shopify operation manually (resource/operation/fields). | Applies to all `shopifyTool` nodes. |
| “Update a product catagoirs” appears to be a misspelling; consider renaming for clarity. | Node naming/maintenance. |
| Rapiwa is a community/custom node; ensure it is installed on your n8n instance and version-compatible. | Node availability risk. |