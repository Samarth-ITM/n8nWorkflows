Manage WooCommerce store operations via AI Telegram bot with OpenRouter

https://n8nworkflows.xyz/workflows/manage-woocommerce-store-operations-via-ai-telegram-bot-with-openrouter-13503


# Manage WooCommerce store operations via AI Telegram bot with OpenRouter

## 1. Workflow Overview

**Workflow name:** *eCommerce Ai Agent ( Telegram Bot )*  
**Purpose:** Operate a WooCommerce digital-product (ebooks) store from Telegram using an AI agent powered by **OpenRouter**, with **conversation memory**, and the ability to call “tools” that perform real actions: **read orders**, **read/update products**, **log to Google Sheets**, and **send customer emails via Gmail**.

**Target use cases**
- Sales monitoring and order lookup from chat
- Product listing and product updates (price/description/stock fields)
- Customer support actions (emailing customers)
- Lightweight reporting by logging orders to Google Sheets

### 1.1 Input Reception (Telegram)
Receives Telegram messages and forwards the user text to the AI agent.

### 1.2 AI Orchestration (Agent + Model + Memory)
The **AI Agent** interprets the user message, uses the OpenRouter chat model, keeps chat context via memory, and decides which tool node(s) to execute.

### 1.3 Store Operations (WooCommerce Tools)
Tool nodes for: **Get orders**, **Get products**, **Update product**. These are invoked by the agent when needed.

### 1.4 Logging & Notifications (Google Sheets + Gmail Tools)
Tool nodes for: writing/updating rows in **Google Sheets** as a “database” and sending emails through **Gmail**.

### 1.5 Output Delivery (Telegram)
Sends the agent’s final response back to the Telegram user.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Intake
**Overview:** Listens for incoming Telegram messages and starts the workflow.  
**Nodes involved:** `Telegram Trigger`

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point; webhook-based Telegram update listener.
- **Key configuration:**
  - **Updates:** `message` (only reacts to standard message events)
  - Uses Telegram credentials: **Telegram account**
- **Inputs / outputs:**
  - **Output →** `AI Agent` (main)
- **Key data produced (typical):**
  - `$json.message.text` (user text)
  - `$json.message.chat.id` (chat identifier; used later)
- **Edge cases / failures:**
  - Telegram credential invalid/revoked
  - Bot not started by user (some chats won’t deliver messages until user initiates)
  - Non-text messages (stickers, photos) may not have `message.text`, causing expressions downstream to evaluate to `null`

---

### Block 2 — Core AI Brain (Model + Memory + Agent)
**Overview:** The AI agent reads the Telegram message, consults memory, uses OpenRouter LLM, and optionally calls tools to fulfill store-management tasks.  
**Nodes involved:** `AI Agent`, `OpenRouter Chat Model`, `Simple Memory`

#### Node: OpenRouter Chat Model
- **Type / role:** `lmChatOpenRouter` — LLM provider for the agent via OpenRouter.
- **Key configuration:**
  - Uses OpenRouter credentials: **OpenRouter account**
  - No special options configured (defaults apply)
- **Connections:**
  - **Output (ai_languageModel) →** `AI Agent`
- **Edge cases / failures:**
  - OpenRouter API key missing/invalid
  - Model/provider rate limits, timeouts
  - If OpenRouter account requires specifying a model and none is set (depends on node defaults / n8n version), the call can fail

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow` — conversation memory for the agent.
- **Key configuration:**
  - **Session key:** `={{ $json.message.chat.id }}` (per Telegram chat)
  - **Context window length:** `20` (keeps last 20 interaction turns/items, depending on implementation)
  - **Session ID type:** custom key
- **Connections:**
  - **Output (ai_memory) →** `AI Agent`
- **Edge cases / failures:**
  - If `message.chat.id` missing (unusual), memory session breaks
  - Memory growth is controlled by window length, but multi-user concurrency depends on correct session keys

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrator; interprets user intent and executes tools.
- **Key configuration:**
  - **User text:** `={{ $json.message.text }}`
  - **System message (important behaviors):**
    - Persona: “AI Store Manager for a WooCommerce Digital Product Store that sells ebooks”
    - Must greet when `/start` or “Hello”:  
      **“Salam Alkiom Mohamed, how can i help you today”**
    - Declares available tools and preferred usage:
      - “Get Orders” for orders info
      - “Database” (Google Sheets) to store orders info
      - Gmail to send emails when asked
    - Language support rules:
      - English, Arabic (MSA), Darija (chat only, not emails), French on request
    - Injects runtime context: `Current date and time: {{ $now }}`
- **Connections:**
  - **Input (main) ←** `Telegram Trigger`
  - **Input (ai_languageModel) ←** `OpenRouter Chat Model`
  - **Input (ai_memory) ←** `Simple Memory`
  - **Tool access (ai_tool) ←** `Get orders`, `Get products`, `Update product`, `database`, `Send email`
  - **Output (main) →** `Send a text message`
- **Outputs:**
  - Produces a final response in `$json.output` (used by Telegram send node)
- **Edge cases / failures:**
  - If `$json.message.text` is empty (non-text update), agent may respond incorrectly or error depending on node behavior
  - Tool calls rely on `$fromAI()` parameters in tool nodes; if the agent does not supply required fields (e.g., Product_ID), tool execution may fail
  - Prompt instruction mismatch: system message references “Get Orders” and “Database”; node names are `Get orders` and `database`. Usually tool binding is by node/tool registration rather than literal string, but ambiguity can reduce tool-use reliability.

---

### Block 3 — WooCommerce Store Tools
**Overview:** Provides WooCommerce read/update operations as callable tools for the agent.  
**Nodes involved:** `Get orders`, `Get products`, `Update product`

#### Node: Get orders
- **Type / role:** `wooCommerceTool` — agent tool to list orders.
- **Key configuration:**
  - **Resource:** `order`
  - **Operation:** `getAll`
  - **Return all:** `={{ $fromAI('Return_All', '', 'boolean') }}`
    - The agent decides whether to retrieve all results.
- **Connections:**
  - **Tool output (ai_tool) →** `AI Agent`
- **Edge cases / failures:**
  - WooCommerce credential invalid, REST API disabled, or wrong site URL
  - Pagination/large datasets: `Return_All=true` can be slow or hit limits
  - If agent doesn’t provide `Return_All`, expression may resolve to `false`/empty depending on `$fromAI` behavior; could unexpectedly return only first page

#### Node: Get products
- **Type / role:** `wooCommerceTool` — agent tool to list products.
- **Key configuration:**
  - **Operation:** `getAll` (resource defaults to product in this node’s config; operation indicates product listing)
  - **Return all:** `={{ $fromAI('Return_All', '', 'boolean') }}`
- **Connections:**
  - **Tool output (ai_tool) →** `AI Agent`
- **Edge cases / failures:**
  - Same as orders: auth, rate limits, pagination; returning all products can be heavy

#### Node: Update product
- **Type / role:** `wooCommerceTool` — agent tool to update a specific product.
- **Key configuration:**
  - **Resource:** `product`
  - **Operation:** `update`
  - **Product ID:** `={{ $fromAI('Product_ID', '', 'string') }}`
  - **Update fields** (all AI-supplied):
    - `name`, `description`, `shortDescription`
    - `regularPrice`, `salePrice` (strings; WooCommerce expects string numeric)
    - `manageStock` (boolean)
    - `stockQuantity` (number)
  - **Description type:** auto
- **Connections:**
  - **Tool output (ai_tool) →** `AI Agent`
- **Edge cases / failures:**
  - Missing/invalid Product ID → update fails
  - Type issues: `stockQuantity` must be numeric; price fields should be formatted as string numbers
  - Updating stock fields on digital products may be irrelevant; if `manageStock=false` but `stockQuantity` provided, WooCommerce behavior may vary
  - Permission errors if API key lacks write scope

---

### Block 4 — Data Logging + Email Tools
**Overview:** Provides Google Sheets as a lightweight database and Gmail for customer communication—both callable by the agent.  
**Nodes involved:** `database`, `Send email`

#### Node: database (Google Sheets)
- **Type / role:** `googleSheetsTool` — agent tool to append or update rows in a Google Sheet.
- **Key configuration:**
  - **Operation:** `appendOrUpdate`
  - **Document:** `woocoomerce database` (Spreadsheet ID `1_zPBSN4sD4Tz-jlaY-B2yMI83_6AzRx-NCHrSHrgV5I`)
  - **Sheet:** `Sheet1` (`gid=0`)
  - **Matching column:** `Order ID` (used to decide update vs append)
  - **Mapping mode:** defineBelow; explicit schema and column mappings
  - **Columns written (AI-supplied via `$fromAI`)**:
    - Status, Country, Order ID, Quantity, Date Paid, Last Name, First Name,
      Items Name, Order Date, Customer Note, Customer Email, Payment Method
- **Connections:**
  - **Tool output (ai_tool) →** `AI Agent`
- **Edge cases / failures:**
  - Google OAuth expired/revoked, missing spreadsheet permission
  - If `Order ID` is empty, matching fails → may append duplicates or error depending on node behavior
  - Data normalization: all values are strings (Quantity is also stored as string), which can complicate later numeric analysis
  - Column header mismatch in the actual sheet (must match the schema names)

#### Node: Send email (Gmail)
- **Type / role:** `gmailTool` — agent tool to send emails.
- **Key configuration:**
  - **To:** `={{ $fromAI('To', '', 'string') }}`
  - **Subject:** `={{ $fromAI('Subject', '', 'string') }}`
  - **Message:** `={{ $fromAI('Message', '', 'string') }}`
  - **Sender name:** `Morocco vibe`
  - **Append attribution:** false
- **Connections:**
  - **Tool output (ai_tool) →** `AI Agent`
- **Edge cases / failures:**
  - Gmail OAuth not authorized for sending
  - Invalid recipient email
  - Prompt rule: Darija allowed for chat replies but **not** emails; agent must comply—otherwise content policy/brand issues (and deliverability) may arise
  - If the agent does not provide To/Subject/Message, send will fail

---

### Block 5 — Telegram Output
**Overview:** Sends the agent’s final response back to the originating Telegram chat.  
**Nodes involved:** `Send a text message`

#### Node: Send a text message
- **Type / role:** `telegram` — send message action.
- **Key configuration:**
  - **Chat ID:** `={{ $('Telegram Trigger').item.json.message.chat.id }}`
  - **Text:** `={{ $json.output }}`
  - **Append attribution:** false
- **Connections:**
  - **Input ←** `AI Agent`
- **Edge cases / failures:**
  - If AI Agent output doesn’t include `output`, message will be blank or fail
  - If Telegram Trigger item reference fails (e.g., different execution paths, missing item), chatId expression can error
  - Telegram rate limits if many responses are sent quickly

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receives Telegram messages | — | AI Agent | ## Telegram bot trigger.  \nReceives user messages and sends them to the AI agent. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates LLM + memory + tool calls; produces final reply | Telegram Trigger; OpenRouter Chat Model (ai); Simple Memory (ai); Tools (ai_tool) | Send a text message | ## Core AI brain.  \nUnderstands user intent and decides which tool to execute.  \nPowered by OpenRouter chat model with memory. |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides LLM responses via OpenRouter | — | AI Agent | ## Core AI brain.  \nUnderstands user intent and decides which tool to execute.  \nPowered by OpenRouter chat model with memory. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores per-chat context window for the agent | — | AI Agent | ## Core AI brain.  \nUnderstands user intent and decides which tool to execute.  \nPowered by OpenRouter chat model with memory. |
| Get orders | wooCommerceTool | Tool: fetch WooCommerce orders | — | AI Agent (ai_tool) | ## Woocommerce  \nStore management tools.  \nIncludes:  \n• Get orders  \n• Get products  \n• Update products |
| Get products | wooCommerceTool | Tool: fetch WooCommerce products | — | AI Agent (ai_tool) | ## Woocommerce  \nStore management tools.  \nIncludes:  \n• Get orders  \n• Get products  \n• Update products |
| Update product | wooCommerceTool | Tool: update a WooCommerce product | — | AI Agent (ai_tool) | ## Woocommerce  \nStore management tools.  \nIncludes:  \n• Get orders  \n• Get products  \n• Update products |
| database | googleSheetsTool | Tool: append/update order rows in Google Sheets | — | AI Agent (ai_tool) | ## Google Sheets  \nDatabase logging.  \nStores retrieved or updated store data.  \n\n## Gmail  \n\nEmail notifications.  \nUsed for sending reports or alerts. |
| Send email | gmailTool | Tool: send emails via Gmail | — | AI Agent (ai_tool) | ## Google Sheets  \nDatabase logging.  \nStores retrieved or updated store data.  \n\n## Gmail  \n\nEmail notifications.  \nUsed for sending reports or alerts. |
| Send a text message | telegram | Sends the AI response back to the Telegram user | AI Agent | — | ## Telegram bot Action .  \nsends Ai Agent messages to user. |
| Sticky Note | stickyNote | Canvas note (documentation) | — | — | Telegram AI eCommerce Agent for WooCommerce  \n\n@[youtube](cO5SazP1eNI)  \n\nThis workflow allows you to manage your WooCommerce store directly from Telegram using an AI assistant.  \n\nFeatures:  \n• Get orders  \n• Retrieve products  \n• Update product information  \n• Store data in Google Sheets  \n• Send emails  \n\nSetup Steps:  \n1. Connect Telegram credentials  \n2. Add WooCommerce API keys  \n3. Connect Google Sheets  \n4. Connect Gmail  \n5. Add OpenRouter/OpenAI API key  \n\nUse cases:  \n• Sales monitoring  \n• Product updates  \n• Customer support  \n• Store reporting  \n\nSend commands via Telegram to interact with your store conversationally. |
| Sticky Note1 | stickyNote | Canvas note (documentation) | — | — | ## Telegram bot trigger.  \n\nReceives user messages and sends them to the AI agent. |
| Sticky Note2 | stickyNote | Canvas note (documentation) | — | — | ## Telegram bot Action .  \n\nsends Ai Agent messages to user. |
| Sticky Note3 | stickyNote | Canvas note (documentation) | — | — | ## Core AI brain.  \nUnderstands user intent and decides which tool to execute.  \nPowered by OpenRouter chat model with memory. |
| Sticky Note4 | stickyNote | Canvas note (documentation) | — | — | ## Woocommerce  \n\nStore management tools.  \n\nIncludes:  \n• Get orders  \n• Get products  \n• Update products |
| Sticky Note5 | stickyNote | Canvas note (documentation) | — | — | ## Google Sheets  \n\nDatabase logging.  \nStores retrieved or updated store data.  \n\n## Gmail  \n\nEmail notifications.  \nUsed for sending reports or alerts. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: `eCommerce Ai Agent ( Telegram Bot )`.

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Updates: **message**
   - Credentials: connect your **Telegram Bot API** credential (create a bot with BotFather, paste token into n8n credential).

3. **Add OpenRouter Chat Model**
   - Node: **OpenRouter Chat Model** (OpenRouter chat)
   - Credentials: **OpenRouter API key**
   - Keep default options unless you want to force a specific model (recommended if your environment requires it).

4. **Add Simple Memory**
   - Node: **Simple Memory**
   - Session ID type: **Custom key**
   - Session key expression: `{{ $json.message.chat.id }}`
   - Context window length: `20`

5. **Add AI Agent**
   - Node: **AI Agent** (LangChain agent)
   - Input text expression: `{{ $json.message.text }}`
   - Prompt type: **Define**
   - System message: paste the provided system message (persona + greeting rule + tools + languages + `{{ $now }}` context).
   - Connect:
     - `Telegram Trigger` → `AI Agent` (main)
     - `OpenRouter Chat Model` → `AI Agent` (ai_languageModel)
     - `Simple Memory` → `AI Agent` (ai_memory)

6. **Add WooCommerce tool nodes**
   - Create **Get orders**:
     - Node: WooCommerce Tool
     - Resource: **Order**
     - Operation: **Get All**
     - Return All expression: `{{ $fromAI('Return_All', '', 'boolean') }}`
     - Credentials: WooCommerce REST API (consumer key/secret; ensure read permissions)
     - Connect node to agent as a **tool** (ai_tool).
   - Create **Get products**:
     - Operation: **Get All**
     - Return All expression: `{{ $fromAI('Return_All', '', 'boolean') }}`
     - Credentials: same WooCommerce credential
     - Connect as **ai_tool** to the agent.
   - Create **Update product**:
     - Resource: **Product**
     - Operation: **Update**
     - Product ID: `{{ $fromAI('Product_ID', '', 'string') }}`
     - Update fields (all from AI):
       - Name: `{{ $fromAI('Name', '', 'string') }}`
       - Regular price: `{{ $fromAI('Regular_Price', '', 'string') }}`
       - Sale price: `{{ $fromAI('Sale_Price', '', 'string') }}`
       - Description: `{{ $fromAI('Description', '', 'string') }}`
       - Short description: `{{ $fromAI('Short_Description', '', 'string') }}`
       - Manage stock: `{{ $fromAI('Manage_Stock', '', 'boolean') }}`
       - Stock quantity: `{{ $fromAI('Stock_Quantity', '', 'number') }}`
     - Connect as **ai_tool** to the agent.

7. **Add Google Sheets “database” tool**
   - Node: **Google Sheets Tool**
   - Operation: **Append or Update**
   - Select spreadsheet: your target sheet (create one if needed)
   - Select worksheet/tab (e.g., `Sheet1`)
   - Set **Matching column**: `Order ID`
   - Define columns and map each field using `$fromAI(...)`:
     - Status, Country, Order ID, Quantity, Date Paid, Last Name, First Name, Items Name, Order Date, Customer Note, Customer Email, Payment Method
   - Credentials: Google Sheets OAuth2 (account with access to the spreadsheet)
   - Connect as **ai_tool** to the agent.

8. **Add Gmail send tool**
   - Node: **Gmail Tool** (send)
   - To: `{{ $fromAI('To', '', 'string') }}`
   - Subject: `{{ $fromAI('Subject', '', 'string') }}`
   - Message: `{{ $fromAI('Message', '', 'string') }}`
   - Options:
     - Sender name: `Morocco vibe`
     - Append attribution: off
   - Credentials: Gmail OAuth2 (authorized for sending)
   - Connect as **ai_tool** to the agent.

9. **Add Telegram Send Message**
   - Node: **Telegram** → “Send a text message”
   - Chat ID expression: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Text expression: `{{ $json.output }}`
   - Append attribution: off
   - Connect: `AI Agent` → `Send a text message` (main)

10. **Activate the workflow**
   - Send `/start` or “Hello” in Telegram to validate greeting behavior.
   - Test tool calls with commands like:
     - “Show me the latest orders”
     - “List products”
     - “Update product 123 regular price to 19.99”
     - “Email customer X about their order”

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Telegram AI eCommerce Agent for WooCommerce | Sticky note overview text in workflow canvas |
| YouTube link: `@[youtube](cO5SazP1eNI)` | Included in sticky note (embedded YouTube reference) |
| Setup Steps (credentials): Telegram, WooCommerce API keys, Google Sheets, Gmail, OpenRouter/OpenAI | Included in sticky note |
| Use cases: Sales monitoring, Product updates, Customer support, Store reporting | Included in sticky note |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.