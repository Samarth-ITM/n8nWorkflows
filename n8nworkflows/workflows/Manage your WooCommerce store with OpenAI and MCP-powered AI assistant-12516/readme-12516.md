Manage your WooCommerce store with OpenAI and MCP-powered AI assistant

https://n8nworkflows.xyz/workflows/manage-your-woocommerce-store-with-openai-and-mcp-powered-ai-assistant-12516


# Manage your WooCommerce store with OpenAI and MCP-powered AI assistant

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Manage your WooCommerce store with OpenAI and MCP-powered AI assistant  
**Internal workflow name:** AI-Powered WooCommerce Store Management using OpenAI & MCP  
**Status:** inactive (`active: false`)

This workflow implements an AI commerce assistant that receives chat messages, uses OpenAI (via n8n’s LangChain nodes) to interpret the request, and executes WooCommerce operations via an MCP (Model Context Protocol) tool server. When the assistant completes a “write” action (create/update/delete for products/customers/orders), it attempts to emit a strict status token (`successfully` or `notsuccessfully`). If the token equals `successfully`, the workflow sends multi-channel notifications (Gmail, Discord, Telegram, WhatsApp via Rapiwa).

### 1.1 Input Reception (Chat)
- Entry point: **Chat message received** trigger (public, file uploads allowed)
- Feeds the agent with the user’s message.

### 1.2 AI Agent Orchestration (Reasoning + Memory + Tools)
- **AI BOT** agent uses:
  - **OpenAI Model** (chat model) for reasoning
  - **Memory** for short conversational context
  - **WooCommerce MCP Client** (MCP tool access to WooCommerce automation server)
- Additionally, there is a separate **MCP Client** and a **Calculator** tool wired to a downstream “Message a model” node (not to the Agent), creating a *split tool architecture*.

### 1.3 Action Result Normalization (Strict “successfully” token)
- **Message a model** is instructed (system message) to output only `successfully` or `notsuccessfully` after tool actions.
- **IF (check status for notify)** checks if the model output equals `successfully`.

### 1.4 Notifications (Multi-channel)
- On success, sends “Your guide has been successfully updated.” via:
  - Gmail
  - Discord DM
  - Telegram
  - WhatsApp (Rapiwa)

### 1.5 MCP Server (Tool Provider)
- **WooCommerce MCP Server** is an MCP Trigger endpoint (`/mcp/my-personal-MCP`) that exposes WooCommerce tool nodes as MCP tools.
- Many WooCommerce tool nodes connect to this MCP Trigger using the `ai_tool` connection type.

---

## 2. Block-by-Block Analysis

### Block A — Chat Intake
**Overview:** Receives inbound user chat messages and starts the AI agent execution.  
**Nodes involved:** `Chat message received`

#### Node: Chat message received
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — workflow trigger for chat-based interactions.
- **Key configuration:**
  - `public: true` (anyone with the endpoint can access; ensure you intend this)
  - `allowFileUploads: true` (files may arrive; no downstream file handling is defined in this workflow)
- **Connections:**
  - **Output →** `AI BOT` (main)
- **Edge cases / failures:**
  - Public endpoint can be abused (spam, large files).
  - File uploads may produce payloads that the agent can’t use unless you add parsing/ingestion steps.

---

### Block B — Agent Core (Memory + LLM + MCP Tools)
**Overview:** The agent interprets natural language requests and uses MCP tools to manage WooCommerce with minimal user questions.  
**Nodes involved:** `AI BOT`, `OpenAI Model`, `Memory`, `WooCommerce MCP Client`

#### Node: AI BOT
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — conversational agent orchestrating LLM + tools.
- **Key configuration:**
  - Large **system message** defining behavior:
    - “Minimize questions” by fetching store data first
    - “Action-oriented” (create/update/search/report/coupons/settings)
    - “Retry/fuzzy search” guidance
    - Confirmation only for irreversible actions (e.g., refunds)
  - **Note:** The system message text mentions “Shopify” in a sticky note, but the workflow is WooCommerce-focused.
- **Connections:**
  - **Input:** from `Chat message received` (main)
  - **ai_languageModel:** from `OpenAI Model`
  - **ai_memory:** from `Memory`
  - **ai_tool:** from `WooCommerce MCP Client`
  - **Output →** `Message a model` (main)
- **Edge cases / failures:**
  - If MCP endpoint is unreachable, agent can’t perform store actions.
  - Tool naming/availability depends on MCP Server configuration (not explicitly filtered here).
  - If the user asks for unsupported operations (e.g., coupons/settings) but no corresponding WooCommerce tools are exposed, agent may fail or hallucinate capability unless guarded.

#### Node: OpenAI Model
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — chat LLM provider for the agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
- **Connections:**
  - **Output (ai_languageModel) →** `AI BOT`
- **Failure modes:**
  - OpenAI credential errors, quota limits, model availability changes.
  - Latency/timeouts under high load.

#### Node: Memory
- **Type / Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversation memory.
- **Key configuration:**
  - `contextWindowLength: 10` (keeps last 10 turns/items)
- **Connections:**
  - **Output (ai_memory) →** `AI BOT`
- **Edge cases:**
  - Memory may omit older but important details; consider longer window for complex store operations.

#### Node: WooCommerce MCP Client
- **Type / Role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — MCP tool client used by the agent.
- **Key configuration:**
  - `sseEndpoint: https://n8.spagreen.net/mcp/my-personal-MCP/sse`
- **Connections:**
  - **Output (ai_tool) →** `AI BOT`
- **Failure modes:**
  - SSE endpoint unreachable, TLS/cert issues, auth requirements (if any) not configured.
  - MCP server path mismatch (must align with MCP Trigger path).

---

### Block C — Post-Agent Status Enforcement + Success Gate
**Overview:** A separate OpenAI call is used to force a strict “successfully/notsuccessfully” output, then an IF node gates notification sending.  
**Nodes involved:** `Message a model`, `IF (check status for notify)`, `Calculator`, `MCP Client`

#### Node: Message a model
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — direct OpenAI message call (separate from the Agent’s LLM node).
- **Key configuration:**
  - Model ID: “add your ID” (must be replaced; uses ID mode)
  - System message strictly instructs:
    - After any tool action among: product upload/create/update, order create/update/fulfillment/delete → return only `successfully`
    - On failure → return only `notsuccessfully`
    - No additional text
- **Connections:**
  - **Input:** from `AI BOT` (main)
  - **Tool inputs:** from `Calculator` (ai_tool), from `MCP Client` (ai_tool)
  - **Output →** `IF (check status for notify)`
- **Important behavior note:**
  - This node is the *only place* where the strict status token is checked downstream.
  - It is not guaranteed that this node actually “knows” whether WooCommerce tool actions succeeded unless the preceding content/tool execution results are correctly passed into it.
- **Failure modes:**
  - Model ID not set → runtime failure.
  - If the prompt or tool outputs include extra text, the model may violate the strict output requirement.
  - If the “tool action” occurs in the agent step but the status model doesn’t see structured results, it may output incorrectly.

#### Node: IF (check status for notify)
- **Type / Role:** `n8n-nodes-base.if` — conditional gate.
- **Key configuration:**
  - Condition: `$json.output` **equals** `"successfully"` (strict, case-sensitive).
- **Connections:**
  - **True branch →** Gmail + Discord + Telegram + Rapiwa notification nodes (all in parallel)
  - No false-branch actions configured.
- **Edge cases:**
  - If the OpenAI node output field is not `output` (depends on node version/output schema), the condition fails silently and notifications never fire.
  - If model returns `Successfully` (capital S) or includes whitespace/newlines → condition fails.

#### Node: Calculator
- **Type / Role:** `@n8n/n8n-nodes-langchain.toolCalculator` — math tool for the model.
- **Connections:** **ai_tool →** `Message a model`
- **Note:** Not connected to the Agent; only available to the status-enforcing model call.
- **Edge cases:** Typically safe; rarely needed in this workflow unless the model uses it.

#### Node: MCP Client
- **Type / Role:** `@n8n/n8n-nodes-langchain.mcpClientTool` — second MCP client tool.
- **Key configuration:**
  - `endpointUrl: https://n8.spagreen.net/mcp/my-personal-MCP/sse`
- **Connections:** **ai_tool →** `Message a model`
- **Note:** Duplicates MCP access but not used by the Agent (Agent uses “WooCommerce MCP Client”). This can cause confusion and inconsistent tool availability between agent vs. status model.

---

### Block D — Notifications on Success
**Overview:** When the IF node confirms success, send a confirmation message through multiple channels.  
**Nodes involved:** `Send a message2 (Gmail)`, `Send a message (Discord)`, `Send a text message (Telegram)`, `Rapiwa (WhatsApp)`

#### Node: Send a message2
- **Type / Role:** `n8n-nodes-base.gmail` — send email.
- **Key configuration:**
  - To: `your_mail_address`
  - Subject: `Shopify Update (BOT)` (mismatch with WooCommerce branding)
  - Body: `Your guide has been successfully updated.`
- **Connections:** from `IF (check status for notify)` true branch.
- **Failure modes:** OAuth token expiry, missing Gmail scopes, rate limits.

#### Node: Send a message (Discord)
- **Type / Role:** `n8n-nodes-base.discord` — send a Discord message (DM to user).
- **Key configuration:**
  - SendTo: `user`
  - `userId: your_user_id`
  - `guildId: your_server_id`
  - Content: `Your guide has been successfully updated.`
- **Failure modes:** Bot not in guild, missing permissions, wrong user ID, Discord API errors.

#### Node: Send a text message (Telegram)
- **Type / Role:** `n8n-nodes-base.telegram` — send Telegram message.
- **Key configuration:**
  - `chatId: your_telegram_id`
  - Text: `Your guide has been successfully updated.`
- **Failure modes:** invalid chat ID, bot not started by user, blocked bot, API limits.

#### Node: Rapiwa
- **Type / Role:** `n8n-nodes-rapiwa.rapiwa` — WhatsApp message via Rapiwa.
- **Key configuration:**
  - Number: `your_whatsapp_number`
  - Message: `Your guide has been successfully updated.`
- **Failure modes:** provider downtime, invalid number formatting, API key issues, template/session constraints depending on WhatsApp rules/provider.

---

### Block E — MCP Server: WooCommerce Tool Exposure
**Overview:** Exposes WooCommerce REST operations as MCP tools via an MCP Trigger endpoint. The agent and/or the status model can call these tools through MCP clients.  
**Nodes involved:** `WooCommerce MCP Server`, all `* in WooCommerce` tool nodes (customers/products/orders)

#### Node: WooCommerce MCP Server
- **Type / Role:** `@n8n/n8n-nodes-langchain.mcpTrigger` — MCP server trigger that publishes tools.
- **Key configuration:**
  - `path: my-personal-MCP` (your MCP endpoint path)
- **Connections:**
  - Receives `ai_tool` connections from WooCommerce tool nodes (they register/expose to MCP trigger).
- **Failure modes / edge cases:**
  - If you deploy behind a reverse proxy, ensure SSE works and the `/sse` endpoint is reachable.
  - Tool exposure depends on correct wiring using `ai_tool` connections.

#### WooCommerce tool nodes (exposed as MCP tools)
All nodes below are `n8n-nodes-base.wooCommerceTool` and connect via `ai_tool → WooCommerce MCP Server`.

##### Customer tools
1) **Create a customer in WooCommerce**
- Operation: `customer.create`
- Uses many `$fromAI('Field', '', 'type')` expressions for email, name, addresses, meta_data fields, etc.
- Has a hardcoded placeholder password: `YOUR_CREDENTIAL_HERE` (should be replaced or removed; consider generating securely).
- Failure modes: duplicate email, invalid billing/shipping fields, auth errors.

2) **Get many customers in WooCommerce**
- Operation: `customer.getAll` with `returnAll: true`
- Failure: large stores may be slow/heavy; consider pagination.

3) **Get a customer in WooCommerce By ID**
- Operation: `customer.get` with AI-provided `Customer_ID`
- Failure: invalid ID, not found.

4) **Get customers in WooCommerce By Email**
- Operation: `customer.getAll` filtered by email
- Failure: email not found → empty array handling needed (agent should handle).

5) **Update a customer in WooCommerce**
- Operation: `customer.update` with AI-provided `Customer_ID`, updating billing/shipping, names
- Failure: invalid ID, partial updates overwriting fields (ensure AI only sets intended fields).

6) **Delete a customer in WooCommerce**
- Operation: `customer.delete` with AI-provided `Customer_ID`
- Failure: deletion constraints, wrong ID (irreversible).

##### Product tools
7) **Create a product in WooCommerce**
- Operation: `product.create`
- Sets `status: draft`, `catalogVisibility: visible`, `downloadable: true`, `reviewsAllowed: true`
- Takes categories/tags/sku/slug/prices/description from AI.
- Failure: invalid price formats, category parsing (string vs IDs), downloadable mismatch.

8) **Get many products in WooCommerce**
- Operation: `product.getAll` `returnAll: true`
- Failure: heavy on large catalogs.

9) **Get a product in WooCommerce By Id**
- Operation: `product.get` with AI `Product_ID`

10) **Search many products in WooCommerce By Category Name**
- Operation: `product.getAll` with `options.search = $fromAI('Search')`
- Parameter naming suggests category name, but it’s actually generic search.
- `returnAll` is AI-provided boolean; risky (AI may request huge pulls).

11) **Update a product in WooCommerce**
- Operation: `product.update`
- Updates many fields; forces `type: simple`, `status: publish`, `stockStatus: instock`, `taxStatus: taxable`
- Uses `dimensionsUi` and `metadataUi` with AI-provided values.
- Failure: unintended overwrites (publishing drafts, forcing instock), invalid numeric conversions (`menuOrder`, `stockQuantity`).

12) **Delete a product in WooCommerce**
- Operation: `product.delete`
- Failure: irreversible deletion, wrong product ID.

##### Order tools
13) **Create an order in WooCommerce**
- Operation: `order.create`
- Uses line items UI (one item in template) with AI fields for product/variation IDs, totals, quantities
- `setPaid`, currency, customerId, note are AI-provided.
- Failure: mismatched totals/subtotals, invalid productId/variationId, tax/shipping requirements.

14) **Get many orders in WooCommerce**
- Operation: `order.getAll` returnAll true

15) **Get an order in WooCommerce**
- Operation: `order.get` by AI `Order_ID`

16) **Get many orders in WooCommerce By Customer**
- Operation: `order.getAll` with `options.customer` AI-provided

17) **Update an order in WooCommerce**
- Operation: `order.update` with extensive billing/shipping/line items/fees/coupons/shipping lines
- `updateFields.status` hardcoded to `pending`
- Failure: overwriting addresses/items unintentionally; “method ID” field appears as `method ID` (with a space) which may not map correctly.

18) **Delete an order in WooCommerce**
- Operation: `order.delete` by AI `Order_ID`
- Failure: irreversible; should require confirmation.

**Version-specific note on `$fromAI(...)`:** These expressions are n8n’s “auto-generated-fromAI-override” placeholders typically used when AI is expected to supply parameters. They require the calling AI/tool layer to provide those fields. If the MCP/agent call doesn’t supply them, nodes may send empty strings or fail validation depending on WooCommerce API requirements.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Chat message received | LangChain Chat Trigger | Public chat entry point |  | AI BOT | ## Overview… (full overview + links from Sticky Note5) |
| AI BOT | LangChain Agent | Main reasoning + tool orchestration | Chat message received; Memory; OpenAI Model; WooCommerce MCP Client | Message a model | **AI BOT**: Understands requests, maintains context, and automates Shopify. / **Shopify MCP Client**: Handles products, orders, searches, and discounts. |
| OpenAI Model | LangChain ChatOpenAI | LLM for agent |  | AI BOT | **AI BOT**… / **Shopify MCP Client**… |
| Memory | Buffer Window Memory | Conversation context |  | AI BOT | ## Overview… (full overview + links from Sticky Note5) |
| WooCommerce MCP Client | MCP Client Tool | MCP tools for agent |  | AI BOT | ## MCP Server Setup (3 steps) |
| Message a model | LangChain OpenAI | Enforce strict status output | AI BOT; Calculator; MCP Client | IF (check status for notify) | # Sends notifications when actions are completed. |
| IF (check status for notify) | IF | Gate notifications on “successfully” | Message a model | Send a message2; Send a message; Send a text message; Rapiwa | # Sends notifications when actions are completed. |
| Send a message2 | Gmail | Email notification | IF (check status for notify) |  | ## For notification after work is completed |
| Send a message | Discord | Discord DM notification | IF (check status for notify) |  | ## For notification after work is completed |
| Send a text message | Telegram | Telegram notification | IF (check status for notify) |  | ## For notification after work is completed |
| Rapiwa | Rapiwa WhatsApp | WhatsApp notification | IF (check status for notify) |  | ## For notification after work is completed |
| MCP Client | MCP Client Tool | MCP tool access for status model |  | Message a model | # Sends notifications when actions are completed. |
| Calculator | Calculator Tool | Optional computation tool |  | Message a model | # Sends notifications when actions are completed. |
| WooCommerce MCP Server | MCP Trigger | MCP server exposing WooCommerce tools | (ai_tool registrations from WooCommerce tools) |  | ## WooCommerce automation workflow that manages e-commerce operations (customers, products, orders) |
| Create a customer in WooCommerce | WooCommerce Tool | MCP tool: create customer |  | WooCommerce MCP Server | ## Customer Information Related Tools |
| Get many customers in WooCommerce | WooCommerce Tool | MCP tool: list customers |  | WooCommerce MCP Server | ## Customer Information Related Tools |
| Get a customer in WooCommerce By ID | WooCommerce Tool | MCP tool: get customer by ID |  | WooCommerce MCP Server | ## Customer Information Related Tools |
| Get customers in WooCommerce By Email | WooCommerce Tool | MCP tool: search customers by email |  | WooCommerce MCP Server | ## Customer Information Related Tools |
| Update a customer in WooCommerce | WooCommerce Tool | MCP tool: update customer |  | WooCommerce MCP Server | ## Customer Information Related Tools |
| Delete a customer in WooCommerce | WooCommerce Tool | MCP tool: delete customer |  | WooCommerce MCP Server | ## Customer Information Related Tools |
| Create a product in WooCommerce | WooCommerce Tool | MCP tool: create product |  | WooCommerce MCP Server | ## Product Information Update Tools |
| Get many products in WooCommerce | WooCommerce Tool | MCP tool: list products |  | WooCommerce MCP Server | ## Product Information Update Tools |
| Get a product in WooCommerce By Id | WooCommerce Tool | MCP tool: get product by ID |  | WooCommerce MCP Server | ## Product Information Update Tools |
| Search many products in WooCommerce By Category Name | WooCommerce Tool | MCP tool: search products |  | WooCommerce MCP Server | ## Product Information Update Tools |
| Update a product in WooCommerce | WooCommerce Tool | MCP tool: update product |  | WooCommerce MCP Server | ## Product Information Update Tools |
| Delete a product in WooCommerce | WooCommerce Tool | MCP tool: delete product |  | WooCommerce MCP Server | ## Product Information Update Tools |
| Create an order in WooCommerce | WooCommerce Tool | MCP tool: create order |  | WooCommerce MCP Server | ## Order Information Related Tools *(note: sticky is disabled in workflow)* |
| Get many orders in WooCommerce | WooCommerce Tool | MCP tool: list orders |  | WooCommerce MCP Server | ## Order Information Related Tools *(disabled sticky note)* |
| Get an order in WooCommerce | WooCommerce Tool | MCP tool: get order by ID |  | WooCommerce MCP Server | ## Order Information Related Tools *(disabled sticky note)* |
| Get many orders in WooCommerce By Customer | WooCommerce Tool | MCP tool: list orders by customer |  | WooCommerce MCP Server | ## Order Information Related Tools *(disabled sticky note)* |
| Update an order in WooCommerce | WooCommerce Tool | MCP tool: update order |  | WooCommerce MCP Server | ## Order Information Related Tools *(disabled sticky note)* |
| Delete an order in WooCommerce | WooCommerce Tool | MCP tool: delete order |  | WooCommerce MCP Server | ## Order Information Related Tools *(disabled sticky note)* |
| Sticky Note | Sticky Note (disabled) | Comment |  |  | ## Order Information Related Tools |
| Sticky Note1 | Sticky Note | Comment |  |  | ## Product Information Update Tools |
| Sticky Note2 | Sticky Note | Comment |  |  | ## Customer Information Related Tools |
| Sticky Note3 | Sticky Note | Comment |  |  | ## For notification after work is completed |
| Sticky Note4 | Sticky Note | Comment |  |  | **AI BOT**… **Shopify MCP Client**… |
| Sticky Note5 | Sticky Note | Comment |  |  | ## Overview … + Useful Links (WooCommerce, WP REST, OpenAI, MCP, Discord, Telegram, Gmail, Rapiwa) |
| Sticky Note6 | Sticky Note | Comment |  |  | ## WooCommerce automation workflow that manages e-commerce operations (customers, products, orders) |
| Sticky Note7 | Sticky Note | Comment |  |  | # Sends notifications when actions are completed. |
| Sticky Note8 | Sticky Note | Comment |  |  | ## MCP Server Setup (3 steps) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: *AI-Powered WooCommerce Store Management using OpenAI & MCP* (or your preferred name).
   - Ensure n8n version supports LangChain + MCP nodes.

2) **Add the Chat Trigger**
   - Add node: **Chat message received** (`LangChain Chat Trigger`)
   - Set:
     - **Public:** enabled (only if intended)
     - **Allow file uploads:** enabled/disabled as you prefer
   - This becomes your entry node.

3) **Add Memory**
   - Add node: **Memory** (`Buffer Window Memory`)
   - Set **Context Window Length = 10**
   - Connect **Memory (ai_memory)** → **AI BOT (ai_memory)** later.

4) **Add the Agent**
   - Add node: **AI BOT** (`LangChain Agent`)
   - Paste the provided “AI Commerce Assistant” system message (customize branding: WooCommerce vs Shopify).
   - Connect:
     - **Chat message received (main)** → **AI BOT (main)**

5) **Add the LLM for the Agent**
   - Add node: **OpenAI Model** (`ChatOpenAI`)
   - Choose model: `gpt-4.1-mini` (or any supported chat model)
   - Configure **OpenAI credentials** (API key or OpenAI account credential in n8n)
   - Connect **OpenAI Model (ai_languageModel)** → **AI BOT (ai_languageModel)**

6) **Add MCP Client Tool for the Agent**
   - Add node: **WooCommerce MCP Client** (`MCP Client Tool`)
   - Set **SSE Endpoint** to your MCP server SSE URL, e.g.:
     - `https://<your-domain>/mcp/my-personal-MCP/sse`
   - Connect **WooCommerce MCP Client (ai_tool)** → **AI BOT (ai_tool)**

7) **Add the “status normalization” OpenAI call**
   - Add node: **Message a model** (`OpenAI` LangChain node)
   - Set:
     - **Model ID**: pick a valid model identifier (replace “add your ID”)
     - Add the strict system message that forces output to only `successfully` or `notsuccessfully`
   - Connect **AI BOT (main)** → **Message a model (main)**

8) **(Optional but as per workflow) Add tool nodes for Message a model**
   - Add node: **Calculator** (`Tool Calculator`) and connect **Calculator (ai_tool)** → **Message a model (ai_tool)**
   - Add node: **MCP Client** (`MCP Client Tool`) and set endpoint URL to the same SSE URL
   - Connect **MCP Client (ai_tool)** → **Message a model (ai_tool)**
   - Note: this is redundant if the agent already executes tools; keep only if you truly want the second model to call tools.

9) **Add the IF gate**
   - Add node: **IF (check status for notify)** (`IF`)
   - Condition:
     - Left: expression `{{$json.output}}`
     - Operator: **equals**
     - Right: `successfully`
     - Keep case-sensitive strictness
   - Connect **Message a model (main)** → **IF (check status for notify) (main)**

10) **Add notification nodes (success branch)**
   - Create the following nodes and connect them from **IF true output** in parallel:
     1. **Gmail** “Send a message2”
        - To: your email
        - Subject: update to “WooCommerce Update (BOT)” to avoid confusion
        - Message: customize
        - Configure **Gmail OAuth2 credential**
     2. **Discord** “Send a message”
        - Resource: message, send to user
        - Provide `userId`, `guildId`
        - Configure **Discord Bot credential**
     3. **Telegram** “Send a text message”
        - Provide `chatId`
        - Configure **Telegram API credential**
     4. **Rapiwa** “Rapiwa”
        - Provide WhatsApp number + message
        - Configure **Rapiwa API credential**

11) **Create the MCP Server workflow portion (tool exposure)**
   - Add node: **WooCommerce MCP Server** (`MCP Trigger`)
   - Set **Path** = `my-personal-MCP` (or your desired path)
   - Ensure your deployment supports SSE and external access to `/sse`.

12) **Add WooCommerce tool nodes and attach them to MCP Trigger**
   - For each WooCommerce operation you want to expose, add a **WooCommerce Tool** node and connect:
     - **WooCommerce Tool (ai_tool)** → **WooCommerce MCP Server (ai_tool)**
   - Configure **WooCommerce credentials** (consumer key/secret, store URL, REST API permissions).
   - Implement (as in the provided workflow):
     - Customer: create, getAll, get by ID, getAll by email, update, delete
     - Product: create, getAll, get by ID, search, update, delete
     - Order: create, getAll, get by ID, getAll by customer, update, delete
   - For AI-filled parameters, use `$fromAI('FieldName', '', 'type')` patterns consistently.

13) **Test end-to-end**
   - Send a chat message like: “Create a draft product named Test Product for $19.99”.
   - Verify:
     - Agent calls MCP tools successfully
     - “Message a model” returns exactly `successfully`
     - IF condition passes
     - Notifications fire

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| WooCommerce API | https://developer.woocommerce.com/ |
| WordPress REST API Handbook | https://developer.wordpress.org/rest-api/ |
| OpenAI API docs | https://platform.openai.com/docs/api-reference |
| Model Context Protocol specification | https://modelcontextprotocol.io/ |
| Discord Developers docs | https://discord.com/developers/docs/intro |
| Telegram Bots docs | https://core.telegram.org/bots |
| Gmail API docs | https://developers.google.com/gmail/api |
| Rapiwa documentation | https://rapiwa.com/ |
| Branding mismatch: some notes/subjects reference “Shopify” though tools are WooCommerce | Update Sticky Note4 text and Gmail subject to WooCommerce for clarity |
| Reliability note: IF checks `$json.output` equals `successfully` | Confirm the OpenAI node output field name is `output` in your node version; otherwise adjust expression accordingly |

