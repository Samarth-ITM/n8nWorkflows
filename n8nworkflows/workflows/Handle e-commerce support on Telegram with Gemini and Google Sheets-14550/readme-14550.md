Handle e-commerce support on Telegram with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/handle-e-commerce-support-on-telegram-with-gemini-and-google-sheets-14550


# Handle e-commerce support on Telegram with Gemini and Google Sheets

## 1. Workflow Overview

This workflow implements a Telegram-based e-commerce support bot powered by Google Gemini and Google Sheets. It receives Telegram messages or button clicks, normalizes the user input, routes `/start` requests to a menu, and sends all other requests to an AI agent that can read orders, cancel eligible orders, and create support tickets.

Typical use cases:
- Order tracking by order ID or email
- Order cancellation with status checks and confirmation
- Escalation to human support through ticket creation

### 1.1 Inbound Reception and Message Normalization
The workflow starts from a Telegram webhook trigger and converts either plain messages or inline keyboard callback events into a unified structure containing `chatId`, `text`, `firstName`, and `sessionKey`.

### 1.2 Start Command Routing
The normalized input is checked for the `/start` command. If matched, the workflow sends a predefined Telegram welcome message with inline menu buttons. Otherwise, it passes the request to the AI branch.

### 1.3 AI Support Processing with Tools and Memory
A Gemini-backed AI Agent handles user intent using short-term conversation memory and three Google Sheets tools:
- read orders
- update order status
- append support tickets

The agent prompt defines the business logic for order tracking, cancellation, and support escalation.

### 1.4 Reply Preparation and Telegram Delivery
The AI agent’s textual output is converted into a Telegram-compatible payload. If the agent returns no output, a fallback message is generated. The final text is then sent back to the Telegram user in Markdown mode.

---

## 2. Block-by-Block Analysis

## 2.1 Inbound Reception and Message Normalization

**Overview:**  
This block receives Telegram updates and transforms different Telegram event formats into a single, clean payload used by the rest of the workflow. It also establishes the session key used for AI memory.

**Nodes Involved:**  
- Telegram Trigger
- Extract Message

### Node Details

#### Telegram Trigger
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Entry point that listens for Telegram bot updates through a webhook.
- **Configuration choices:**  
  - Subscribed update types: `message`, `callback_query`
  - This allows the workflow to react both to typed user messages and inline keyboard button clicks.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input; this is a trigger node
  - Output → `Extract Message`
- **Version-specific requirements:**  
  - Uses node type version `1.2`
  - Requires Telegram Bot credentials configured in n8n
- **Edge cases or potential failure types:**  
  - Invalid or missing Telegram credentials
  - Webhook registration issues if the workflow is not active
  - Telegram bot not authorized or token revoked
  - Unsupported update structures that do not contain `message` or `callback_query`
- **Sub-workflow reference:**  
  None

#### Extract Message
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript normalization node that extracts common fields from Telegram events.
- **Configuration choices:**  
  The code checks:
  - `callback_query` → uses callback data and callback sender/message chat data
  - `message` → uses text message and sender/chat data
  - otherwise returns no items
- **Key expressions or variables used:**  
  Output fields:
  - `chatId`
  - `text`
  - `firstName`
  - `sessionKey` = `chatId`
  
  Logic summary:
  - callback event → `text = callback_query.data`
  - message event → `text = message.text || ''`
  - trims text before output
- **Input and output connections:**  
  - Input ← `Telegram Trigger`
  - Output → `Is /start?`
- **Version-specific requirements:**  
  - Uses code node version `2`
- **Edge cases or potential failure types:**  
  - Non-text messages will produce empty string text
  - Telegram updates that contain neither `message` nor `callback_query` are dropped
  - Missing nested Telegram fields could cause unexpected undefined values, although fallback defaults are included for `firstName`
- **Sub-workflow reference:**  
  None

---

## 2.2 Start Command Routing

**Overview:**  
This block decides whether the user is entering the bot through `/start` or requesting operational help. `/start` opens a fixed menu, while all other messages continue to the AI agent.

**Nodes Involved:**  
- Is /start?
- Send Welcome Menu

### Node Details

#### Is /start?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router used to split the flow based on the normalized text value.
- **Configuration choices:**  
  - Compares `{{$json.text}}` to the exact string `/start`
  - Strict and case-sensitive string comparison
- **Key expressions or variables used:**  
  - Left value: `={{ $json.text }}`
  - Right value: `/start`
- **Input and output connections:**  
  - Input ← `Extract Message`
  - True output → `Send Welcome Menu`
  - False output → `Customer Support AI Agent`
- **Version-specific requirements:**  
  - Uses node version `2`
- **Edge cases or potential failure types:**  
  - `/Start` or `/start ` will not match because comparison is exact and case-sensitive
  - Callback buttons never hit this branch unless their callback data literally equals `/start`
- **Sub-workflow reference:**  
  None

#### Send Welcome Menu
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the initial Telegram menu with inline keyboard buttons.
- **Configuration choices:**  
  - Dynamic text greeting with user first name
  - Markdown enabled
  - Inline keyboard with three buttons:
    - `📦 Track My Order` → callback data `Track my order`
    - `❌ Cancel My Order` → callback data `I want to cancel my order`
    - `🎫 Talk to Support` → callback data `I need to talk to support`
- **Key expressions or variables used:**  
  - Text: `={{ '👋 Hello *' + $json.firstName + '*! Welcome to our Support Bot.\n\nHow can I help you today?' }}`
  - Chat ID: `={{ $json.chatId }}`
- **Input and output connections:**  
  - Input ← `Is /start?` true branch
  - No downstream connection
- **Version-specific requirements:**  
  - Uses Telegram node version `1.2`
  - Requires Telegram Bot credentials
- **Edge cases or potential failure types:**  
  - Telegram API errors due to invalid bot credentials
  - Markdown parsing issues if the first name contains Telegram Markdown-reserved characters
  - Bot may be unable to message users if chat permissions are limited
- **Sub-workflow reference:**  
  None

---

## 2.3 AI Support Processing with Tools and Memory

**Overview:**  
This is the core logic block. A Gemini-based AI agent receives the user request, uses short-term memory keyed by Telegram chat, and can call three Google Sheets tools to retrieve orders, update order status, or create a support ticket.

**Nodes Involved:**  
- Customer Support AI Agent
- Google Gemini Chat Model
- Simple Memory
- Tool: Read Orders Sheet
- Tool: Update Order Status
- Tool: Create Support Ticket

### Node Details

#### Customer Support AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates LLM reasoning, memory access, and tool usage.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Main user input text: `={{ $json.text }}`
  - Uses a detailed system message that:
    - injects customer `firstName` and Telegram `chatId`
    - explains available tools
    - defines procedural handling for order tracking, cancellation, and support tickets
    - enforces mandatory reply behavior after tool usage
    - requests concise, friendly Markdown answers
- **Key expressions or variables used:**  
  - User text: `={{ $json.text }}`
  - In system prompt:
    - `{{ $('Extract Message').first().json.firstName }}`
    - `{{ $('Extract Message').first().json.chatId }}`
- **Input and output connections:**  
  - Main input ← `Is /start?` false branch
  - Language model input ← `Google Gemini Chat Model`
  - Memory input ← `Simple Memory`
  - Tool inputs ← all three Google Sheets tool nodes
  - Main output → `Prepare Reply`
- **Version-specific requirements:**  
  - Uses version `2`
  - Requires compatible LangChain/AI features available in the n8n instance
- **Edge cases or potential failure types:**  
  - LLM may misunderstand user intent or fail to call a required tool
  - Prompt/tool mismatch if sheet schemas change
  - Memory may preserve context in ways that affect later turns unexpectedly
  - Empty or ambiguous user text may still produce a general response rather than a deterministic branch
  - Model or tool execution timeouts
- **Sub-workflow reference:**  
  None

#### Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Provides the large language model used by the AI agent.
- **Configuration choices:**  
  - Minimal explicit options configured
  - Uses Google Gemini credentials defined in n8n
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output as AI language model → `Customer Support AI Agent`
- **Version-specific requirements:**  
  - Uses version `1`
  - Requires Google Gemini API credentials
- **Edge cases or potential failure types:**  
  - Invalid API key or insufficient quota
  - Regional/API availability restrictions
  - Safety filtering or model refusal for malformed requests
  - Latency or rate limits
- **Sub-workflow reference:**  
  None

#### Simple Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Maintains conversation memory per Telegram chat/session.
- **Configuration choices:**  
  - Session ID type: custom key
  - Session key expression uses the normalized `chatId`
- **Key expressions or variables used:**  
  - `={{ $('Extract Message').first().json.sessionKey }}`
- **Input and output connections:**  
  - Output as AI memory → `Customer Support AI Agent`
- **Version-specific requirements:**  
  - Uses version `1.3`
- **Edge cases or potential failure types:**  
  - If `sessionKey` is missing, context continuity breaks
  - Memory accumulation may influence follow-up requests unexpectedly
  - Window-based memory can omit older context in longer support conversations
- **Sub-workflow reference:**  
  None

#### Tool: Read Orders Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`  
  AI tool that reads all order rows from a Google Sheet.
- **Configuration choices:**  
  - Operation effectively reads rows from:
    - Google Sheet document ID: `1Yw7vq8iZ3geCCEyst-JpAwfICX1zxUtF0et71muqiT0`
    - Sheet tab: `gid=0`, cached as `Sheet1`
  - Tool description clearly tells the agent to use it for order lookup by `order_id` or email
- **Key expressions or variables used:**  
  None in field mapping; it is a read tool.
- **Input and output connections:**  
  - Output as AI tool → `Customer Support AI Agent`
- **Version-specific requirements:**  
  - Uses Google Sheets tool version `4.5`
  - Requires Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**  
  - Spreadsheet access denied
  - Wrong document ID or worksheet selection
  - Large sheet size may increase latency because the tool reads all rows
  - Column name changes may break the agent’s assumptions (`order_id`, `customer_name`, `email`, `product`, `Status`, `date`)
- **Sub-workflow reference:**  
  None

#### Tool: Update Order Status
- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`  
  AI tool that updates the `Status` field of an order row matched by `order_id`.
- **Configuration choices:**  
  - Operation: `update`
  - Matching column: `order_id`
  - Writable columns:
    - `order_id`
    - `Status`
  - Tool description instructs the agent to use `Cancelled` when appropriate
- **Key expressions or variables used:**  
  Uses AI-generated field extraction via `$fromAI(...)`:
  - `Status = $fromAI('Status', ...)`
  - `order_id = $fromAI('order_id', ...)`
- **Input and output connections:**  
  - Output as AI tool → `Customer Support AI Agent`
- **Version-specific requirements:**  
  - Uses Google Sheets tool version `4.5`
- **Edge cases or potential failure types:**  
  - No matching `order_id` found
  - Multiple duplicate order IDs could update unintended rows depending on sheet behavior
  - Spreadsheet permissions or schema mismatch
  - AI may pass an incorrect `order_id` or unsupported status value
- **Sub-workflow reference:**  
  None

#### Tool: Create Support Ticket
- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`  
  AI tool that appends a new support ticket row into a separate Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Writes to spreadsheet:
    - Document ID: `1RNXRi_WaJbyYnIFOM8YJePUbN-hfEXis0dpbWXGfLmw`
    - Sheet tab: `gid=0`
  - Static/default values:
    - `status = Pending`
    - `created_at = new Date().toISOString()`
    - `telegram_id = Extract Message chatId`
  - AI-populated fields:
    - `ticket_id`
    - `name`
    - `order_id`
    - `query`
    - `summary`
    - `category`
- **Key expressions or variables used:**  
  AI extraction:
  - `name = $fromAI('name', ...)`
  - `query = $fromAI('query', ...)`
  - `summary = $fromAI('summary', ...)`
  - `category = $fromAI('category', ...)`
  - `order_id = $fromAI('order_id', ...)`
  - `ticket_id = $fromAI('ticket_id', ...)`
  
  System-generated:
  - `created_at = {{ new Date().toISOString() }}`
  - `telegram_id = {{ $('Extract Message').first().json.chatId }}`
- **Input and output connections:**  
  - Output as AI tool → `Customer Support AI Agent`
- **Version-specific requirements:**  
  - Uses Google Sheets tool version `4.5`
- **Edge cases or potential failure types:**  
  - AI may generate non-unique `ticket_id` values because uniqueness is instructed but not programmatically enforced
  - Spreadsheet access failures
  - Missing required columns in the support sheet
  - Category drift if the model produces a value outside the expected set
  - Timezone considerations for ISO timestamps
- **Sub-workflow reference:**  
  None

---

## 2.4 Reply Preparation and Telegram Delivery

**Overview:**  
This block converts the AI agent’s result into a clean Telegram message payload and sends it to the user. It also protects against empty AI outputs with a default fallback reply.

**Nodes Involved:**  
- Prepare Reply
- Send Telegram Reply

### Node Details

#### Prepare Reply
- **Type and technical role:** `n8n-nodes-base.code`  
  Post-processing node that extracts the AI output and builds a final `{ chatId, text }` object for Telegram.
- **Configuration choices:**  
  - Reads `output` from the AI agent result
  - If empty or blank, sends a fallback message:
    - `✅ Done! Is there anything else I can help you with? Type /start to see the main menu.`
- **Key expressions or variables used:**  
  - `const output = $input.first().json.output || ''`
  - `const chatId = $('Extract Message').first().json.chatId`
- **Input and output connections:**  
  - Input ← `Customer Support AI Agent`
  - Output → `Send Telegram Reply`
- **Version-specific requirements:**  
  - Uses code node version `2`
- **Edge cases or potential failure types:**  
  - If agent output schema changes and `output` is not present, fallback message will always be used
  - If upstream execution is empty, `.first()` assumptions could fail in abnormal scenarios
- **Sub-workflow reference:**  
  None

#### Send Telegram Reply
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Final outbound Telegram message sender.
- **Configuration choices:**  
  - Sends dynamic `text` to dynamic `chatId`
  - Markdown parse mode enabled
  - `appendAttribution` disabled
- **Key expressions or variables used:**  
  - Text: `={{ $json.text }}`
  - Chat ID: `={{ $json.chatId }}`
- **Input and output connections:**  
  - Input ← `Prepare Reply`
  - No downstream connection
- **Version-specific requirements:**  
  - Uses Telegram node version `1.2`
  - Requires Telegram Bot credentials
- **Edge cases or potential failure types:**  
  - Markdown formatting issues if AI returns malformed Markdown
  - Telegram message length limits for unusually large outputs
  - API errors due to invalid chat ID or blocked bot
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives Telegram webhook updates for messages and callback queries |  | Extract Message | ### ⚙️ Setup Required<br>1. Add your **Telegram Bot** credential<br>2. Add **Google Gemini API** credential<br>3. Add **Google Sheets OAuth2** credential<br>4. Update Sheet IDs if needed<br>5. Activate workflow and register webhook<br>###  🤖 Telegram AI Support Bot<br><br>This workflow manages customer interactions via Telegram, using an AI Agent to handle order tracking, cancellations, and support tickets automatically. |
| Extract Message | n8n-nodes-base.code | Normalizes Telegram message/callback payload into shared fields | Telegram Trigger | Is /start? | ###  Step-1: Inbound & Extraction<br><br><br>Receives messages or button clicks and cleans the data (Chat ID, Name, Text) for the rest of the flow. |
| Is /start? | n8n-nodes-base.if | Routes `/start` requests to menu and other requests to AI | Extract Message | Send Welcome Menu; Customer Support AI Agent | ###  Steo-2: Logic Router<br><br>Checks if the user typed **/start**.<br><br>- **True:** Shows the main menu buttons.<br>- **False:** Passes the query to the AI Agent. |
| Send Welcome Menu | n8n-nodes-base.telegram | Sends Telegram welcome message with inline support options | Is /start? |  |  |
| Customer Support AI Agent | @n8n/n8n-nodes-langchain.agent | Handles support conversations using Gemini, memory, and Google Sheets tools | Is /start?; Google Gemini Chat Model; Simple Memory; Tool: Read Orders Sheet; Tool: Update Order Status; Tool: Create Support Ticket | Prepare Reply | ###  Step-3: AI Agent & Google Sheets Tools<br><br>The **Gemini-powered Agent** uses context and memory to decide which tool to use:<br>- **Read Orders:** To track status.<br>- **Update Orders:** To handle cancellations.<br>- **Create Ticket:** To log issues in the support sheet." |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides LLM backend for the AI agent |  | Customer Support AI Agent | ###  Step-3: AI Agent & Google Sheets Tools<br><br>The **Gemini-powered Agent** uses context and memory to decide which tool to use:<br>- **Read Orders:** To track status.<br>- **Update Orders:** To handle cancellations.<br>- **Create Ticket:** To log issues in the support sheet." |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores conversation context per Telegram chat |  | Customer Support AI Agent | ###  Step-3: AI Agent & Google Sheets Tools<br><br>The **Gemini-powered Agent** uses context and memory to decide which tool to use:<br>- **Read Orders:** To track status.<br>- **Update Orders:** To handle cancellations.<br>- **Create Ticket:** To log issues in the support sheet." |
| Tool: Read Orders Sheet | n8n-nodes-base.googleSheetsTool | AI tool to read all order rows |  | Customer Support AI Agent | ###  Step-3: AI Agent & Google Sheets Tools<br><br>The **Gemini-powered Agent** uses context and memory to decide which tool to use:<br>- **Read Orders:** To track status.<br>- **Update Orders:** To handle cancellations.<br>- **Create Ticket:** To log issues in the support sheet." |
| Tool: Update Order Status | n8n-nodes-base.googleSheetsTool | AI tool to update order status by order ID |  | Customer Support AI Agent | ###  Step-3: AI Agent & Google Sheets Tools<br><br>The **Gemini-powered Agent** uses context and memory to decide which tool to use:<br>- **Read Orders:** To track status.<br>- **Update Orders:** To handle cancellations.<br>- **Create Ticket:** To log issues in the support sheet." |
| Tool: Create Support Ticket | n8n-nodes-base.googleSheetsTool | AI tool to append support tickets to Google Sheets |  | Customer Support AI Agent | ###  Step-3: AI Agent & Google Sheets Tools<br><br>The **Gemini-powered Agent** uses context and memory to decide which tool to use:<br>- **Read Orders:** To track status.<br>- **Update Orders:** To handle cancellations.<br>- **Create Ticket:** To log issues in the support sheet." |
| Prepare Reply | n8n-nodes-base.code | Converts agent output into final Telegram payload with fallback | Customer Support AI Agent | Send Telegram Reply | ###  Step-4: Response Delivery<br><br><br>Formats the AI output into a user-friendly message and sends it back to Telegram. |
| Send Telegram Reply | n8n-nodes-base.telegram | Sends final AI-generated response to Telegram user | Prepare Reply |  | ###  Step-4: Response Delivery<br><br><br>Formats the AI output into a user-friendly message and sends it back to Telegram. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for setup requirements |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for workflow purpose |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for inbound extraction block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for routing block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for AI and Sheets tools block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation for response block |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Telegram Trigger node**
   - Node type: `Telegram Trigger`
   - Configure Telegram Bot credentials.
   - Set update types to:
     - `message`
     - `callback_query`
   - Save the node.
   - Later, activate the workflow so n8n registers the Telegram webhook.

3. **Add a Code node named `Extract Message`**
   - Node type: `Code`
   - Paste this logic:
     - If update contains `callback_query`, extract:
       - `chatId` from `callback_query.message.chat.id`
       - `text` from `callback_query.data`
       - `firstName` from `callback_query.from.first_name`
     - Else if update contains `message`, extract:
       - `chatId` from `message.chat.id`
       - `text` from `message.text`
       - `firstName` from `message.from.first_name`
     - Return:
       - `chatId` as string
       - `text` trimmed
       - `firstName`
       - `sessionKey` equal to `chatId`
     - If neither structure exists, return no items.
   - Connect `Telegram Trigger` → `Extract Message`.

4. **Add an IF node named `Is /start?`**
   - Node type: `If`
   - Configure one string condition:
     - Left value: `{{ $json.text }}`
     - Operation: `equals`
     - Right value: `/start`
   - Keep comparison strict/case-sensitive.
   - Connect `Extract Message` → `Is /start?`.

5. **Add a Telegram node named `Send Welcome Menu`**
   - Node type: `Telegram`
   - Use the same Telegram Bot credentials as the trigger.
   - Operation: send message
   - Chat ID: `{{ $json.chatId }}`
   - Text:
     `{{ '👋 Hello *' + $json.firstName + '*! Welcome to our Support Bot.\n\nHow can I help you today?' }}`
   - Enable `parse_mode = Markdown`
   - Set reply markup to `inlineKeyboard`
   - Add three rows of buttons:
     1. Text: `📦 Track My Order`, callback data: `Track my order`
     2. Text: `❌ Cancel My Order`, callback data: `I want to cancel my order`
     3. Text: `🎫 Talk to Support`, callback data: `I need to talk to support`
   - Connect the **true** output of `Is /start?` → `Send Welcome Menu`.

6. **Add an AI Agent node named `Customer Support AI Agent`**
   - Node type: `AI Agent` / `LangChain Agent`
   - Set input text to: `{{ $json.text }}`
   - Choose manual/defined prompt mode.
   - Paste a system message equivalent to the original logic:
     - Identify the bot as a friendly e-commerce support bot
     - Inject:
       - customer name from `Extract Message.first().json.firstName`
       - Telegram chat ID from `Extract Message.first().json.chatId`
     - Describe the 3 tools:
       - read orders
       - update order status
       - create support ticket
     - Add behavior rules:
       - For tracking: ask for order ID/email, read all rows, filter by ID/email, reply with order details or not found
       - For cancellation: ask for order ID/email, read the order, block cancellation if shipped, note if already cancelled, otherwise request YES/NO confirmation, then update status to `Cancelled`
       - For support: ask for name and issue, optionally order ID, create ticket, and always include ticket ID in final reply
       - Always send a user-facing text after every tool call
       - Use concise and friendly Markdown
       - If unclear input, present the 3 available options
   - Connect the **false** output of `Is /start?` → `Customer Support AI Agent`.

7. **Add a Google Gemini Chat Model node**
   - Node type: `Google Gemini Chat Model`
   - Configure Google Gemini API credentials.
   - Default options are sufficient unless you want to set model-specific parameters.
   - Connect this node to the AI Agent via the `ai_languageModel` connection.

8. **Add a Simple Memory node**
   - Node type: `Simple Memory` / `Buffer Window Memory`
   - Session ID type: `customKey`
   - Session key: `{{ $('Extract Message').first().json.sessionKey }}`
   - Connect this node to the AI Agent via the `ai_memory` connection.

9. **Prepare the orders spreadsheet**
   - In Google Sheets, create or verify an orders sheet with columns expected by the agent:
     - `order_id`
     - `customer_name`
     - `email`
     - `product`
     - `Status`
     - `date`
   - Share the spreadsheet appropriately with the OAuth-connected Google account if needed.

10. **Add a Google Sheets Tool node named `Tool: Read Orders Sheet`**
    - Node type: `Google Sheets Tool`
    - Configure Google Sheets OAuth2 credentials.
    - Select the orders spreadsheet document.
    - Select the sheet/tab, corresponding to `gid=0` in the original workflow.
    - Configure it as a read/list tool that returns all rows.
    - Tool description should explicitly say it reads all order rows and is used to find orders by `order_id` or email.
    - Connect it to the AI Agent via `ai_tool`.

11. **Add a Google Sheets Tool node named `Tool: Update Order Status`**
    - Node type: `Google Sheets Tool`
    - Same orders spreadsheet as above.
    - Operation: `update`
    - Match rows by column:
      - `order_id`
    - Map fields:
      - `order_id` from AI input
      - `Status` from AI input
    - In n8n AI tool mapping, use AI-extracted values for both fields.
    - Tool description should say this tool updates an order’s `Status`, typically to `Cancelled`.
    - Connect it to the AI Agent via `ai_tool`.

12. **Prepare the support spreadsheet**
    - In Google Sheets, create or verify a support sheet with columns:
      - `ticket_id`
      - `name`
      - `order_id`
      - `query`
      - `summary`
      - `category`
      - `status`
      - `created_at`
      - `telegram_id`
    - Share it properly with the OAuth-connected account if needed.

13. **Add a Google Sheets Tool node named `Tool: Create Support Ticket`**
    - Node type: `Google Sheets Tool`
    - Use the support spreadsheet.
    - Operation: `append`
    - Map fields:
      - `ticket_id` from AI
      - `name` from AI
      - `order_id` from AI
      - `query` from AI
      - `summary` from AI
      - `category` from AI
      - `status` fixed to `Pending`
      - `created_at` set to `{{ new Date().toISOString() }}`
      - `telegram_id` set to `{{ $('Extract Message').first().json.chatId }}`
    - In the tool description, instruct the agent to:
      - generate ticket IDs like `TKT-XXXX`
      - classify categories as `Payment`, `Shipping`, `Product Issue`, or `Other`
      - set status to `Pending`
    - Connect it to the AI Agent via `ai_tool`.

14. **Add a Code node named `Prepare Reply`**
    - Node type: `Code`
    - Add logic equivalent to:
      - Read `output` from the AI Agent result
      - Read `chatId` from `Extract Message`
      - If output is empty, return fallback text:
        `✅ Done! Is there anything else I can help you with? Type /start to see the main menu.`
      - Otherwise return `{ chatId, text: output }`
    - Connect `Customer Support AI Agent` → `Prepare Reply`.

15. **Add a Telegram node named `Send Telegram Reply`**
    - Node type: `Telegram`
    - Use the same Telegram credential.
    - Operation: send message
    - Chat ID: `{{ $json.chatId }}`
    - Text: `{{ $json.text }}`
    - Enable:
      - `parse_mode = Markdown`
      - `appendAttribution = false`
    - Connect `Prepare Reply` → `Send Telegram Reply`.

16. **Optionally add sticky notes for workspace clarity**
    - Add notes similar to:
      - setup requirements
      - inbound extraction
      - logic routing
      - AI tools and memory
      - response delivery

17. **Configure credentials**
    - **Telegram Bot credential**
      - Create a bot via BotFather
      - Copy the bot token into n8n
    - **Google Gemini API credential**
      - Create or obtain a Google AI/Gemini API key
      - Add it to n8n
    - **Google Sheets OAuth2 credential**
      - Configure OAuth client details in Google Cloud if needed
      - Authenticate the Google account that can access both spreadsheets

18. **Test the workflow**
    - Send `/start` in Telegram:
      - Verify welcome text and inline keyboard appear
    - Click each button:
      - Verify callback query is processed and reaches AI
    - Test sample prompts:
      - order tracking by known order ID
      - cancellation of non-shipped order
      - support issue creation
    - Confirm updates/rows appear in Google Sheets.

19. **Activate the workflow**
    - Activation is required for Telegram webhook registration.
    - If the trigger does not work after credential changes, deactivate and reactivate the workflow.

20. **Recommended hardening after rebuild**
    - Escape Telegram Markdown special characters in dynamic text
    - Add validation for empty user messages
    - Add uniqueness validation for `ticket_id`
    - Add better matching/normalization for `/start` variants and email/order format checks
    - Consider structured confirmation state tracking for cancellations instead of relying only on chat memory

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add your **Telegram Bot** credential | Initial setup requirement |
| Add **Google Gemini API** credential | Initial setup requirement |
| Add **Google Sheets OAuth2** credential | Initial setup requirement |
| Update Sheet IDs if needed | Spreadsheet configuration |
| Activate workflow and register webhook | Required for Telegram trigger to work |
| This workflow manages customer interactions via Telegram, using an AI Agent to handle order tracking, cancellations, and support tickets automatically. | Project purpose |

