Send AI-parsed restaurant orders and status updates via Telegram and Google Sheets

https://n8nworkflows.xyz/workflows/send-ai-parsed-restaurant-orders-and-status-updates-via-telegram-and-google-sheets-14136


# Send AI-parsed restaurant orders and status updates via Telegram and Google Sheets

# 1. Workflow Overview

This workflow implements a restaurant order bot on Telegram, backed by Google Sheets and assisted by an AI parsing step. It serves two main purposes:

1. Let customers place and manage food orders through Telegram.
2. Automatically notify customers when restaurant staff updates order status in Google Sheets.

The workflow is logically split into two independent entry-point flows.

## 1.1 Customer Message Intake and Routing

This block listens to all incoming Telegram customer messages, extracts normalized fields such as chat ID and text, detects commands, and routes each message to the appropriate handling path.

## 1.2 Static Customer Responses

This block handles simple command-driven responses that do not require database or AI processing, such as welcome text, help, menu display, and blocking staff-only commands.

## 1.3 AI-Based Order Parsing and Order Creation

This block treats ordinary free-text messages as potential food orders, uses Anthropic Claude Haiku through the LangChain Agent node to parse them, generates queue and wait data, saves valid orders to Google Sheets, and confirms the order to the customer.

## 1.4 Order Cancellation Flow

This block handles `CANCEL <queue>` requests. It looks up the order in Google Sheets, validates ownership and current status, updates the row when cancellation is allowed, and replies to the customer with the result.

## 1.5 Order Status Lookup Flow

This block handles `STATUS <queue>` requests by retrieving the matching order row from Google Sheets and formatting a human-readable Telegram response.

## 1.6 Customer Order History Flow

This block supports `/myorders` by reading all rows from Google Sheets, filtering by Telegram chat ID, and returning the latest five orders for that customer.

## 1.7 Scheduled Staff Status Notification Flow

This separate scheduled flow runs every minute, reads all rows from the sheet, detects rows whose `Status` differs from `Last Status Sent`, sends Telegram notifications for those status changes, and then writes the new status into column H to prevent duplicate notifications.

---

# 2. Block-by-Block Analysis

## 2.1 Customer Message Intake and Routing

### Overview
This is the main customer-facing entry block. It receives Telegram messages, normalizes the payload into a simpler structure, classifies the message intent, and dispatches it to one of eight downstream branches.

### Nodes Involved
- Customer Bot Listener
- Route Message
- Message Switch

### Node Details

#### Customer Bot Listener
- **Type and role:** `n8n-nodes-base.telegramTrigger`; Telegram webhook trigger for incoming customer messages.
- **Configuration choices:** Configured to listen only for `message` updates.
- **Key expressions or variables used:** None in-node; raw message payload is passed downstream.
- **Input and output connections:** Entry point node; outputs to `Route Message`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases / failures:**
  - Telegram credential or webhook registration issues.
  - Bot token misconfiguration.
  - Bot privacy mode or blocked bot interactions could reduce delivered events.
- **Sub-workflow reference:** None.

#### Route Message
- **Type and role:** `n8n-nodes-base.code`; transforms Telegram payload into normalized routing metadata.
- **Configuration choices:** Custom JavaScript extracts:
  - `text`
  - `chat_id`
  - `first_name`
  - uppercase version `upper`
  - `route`
  - `queue_match`
- **Key expressions or variables used:**
  - `msg.message?.text || msg.text`
  - `msg.message?.chat?.id || msg.chat?.id`
  - Regex matches:
    - `^STATUS\s+(\d+)`
    - `^CANCEL\s+(\d+)`
    - `^/START`
    - `^/HELP|^HELP`
    - `^/MENU|^MENU`
    - `^/MYORDERS|^MYORDERS`
    - `^READY\s+\d+` => blocked
- **Routing behavior:**
  - default route = `order`
  - command-specific overrides for `start`, `help`, `menu`, `myorders`, `status`, `cancel`
  - customer attempts to send `READY <id>` are blocked
- **Input and output connections:** Input from `Customer Bot Listener`; output to `Message Switch`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Empty text becomes `''`, which still routes to `order`.
  - Non-text Telegram messages are effectively treated as empty order messages.
  - Only numeric queue IDs are recognized in `STATUS` and `CANCEL`.
- **Sub-workflow reference:** None.

#### Message Switch
- **Type and role:** `n8n-nodes-base.switch`; branches execution by `route`.
- **Configuration choices:** Eight explicit route conditions:
  - `start`
  - `help`
  - `order`
  - `cancel`
  - `status`
  - `menu`
  - `myorders`
  - `blocked`
- **Key expressions or variables used:** `={{ $json.route }}`
- **Input and output connections:** Input from `Route Message`; outputs to all customer response / processing branches.
- **Version-specific requirements:** Type version `3.2`, conditions configured with version 2 rules.
- **Edge cases / failures:**
  - Any unrecognized route would not match a branch, but current code always produces one of the defined values.
- **Sub-workflow reference:** None.

---

## 2.2 Static Customer Responses

### Overview
This block covers direct-response commands that do not query the sheet or call AI. It sends prewritten Telegram messages for onboarding, help, menu display, and staff-command blocking.

### Nodes Involved
- Send Welcome
- Send Help
- Send Menu
- Send Blocked Message

### Node Details

#### Send Welcome
- **Type and role:** `n8n-nodes-base.telegram`; sends a welcome/onboarding message.
- **Configuration choices:** Uses Markdown formatting and references customer first name and chat ID from `Route Message`.
- **Key expressions or variables used:**
  - `{{ $('Route Message').first().json.first_name }}`
  - `{{ $('Route Message').first().json.chat_id }}`
- **Input and output connections:** Input from `Message Switch` start branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram Markdown parse issues if the name contains special characters.
  - Telegram auth or chat delivery failures.
- **Sub-workflow reference:** None.

#### Send Help
- **Type and role:** `n8n-nodes-base.telegram`; sends usage instructions.
- **Configuration choices:** Markdown-formatted help text.
- **Key expressions or variables used:** chat ID from `Route Message`.
- **Input and output connections:** Input from `Message Switch` help branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Telegram Markdown escaping must remain valid.
- **Sub-workflow reference:** None.

#### Send Menu
- **Type and role:** `n8n-nodes-base.telegram`; sends static menu text.
- **Configuration choices:** Hard-coded menu items and prices in Markdown.
- **Key expressions or variables used:** chat ID from `Route Message`.
- **Input and output connections:** Input from `Message Switch` menu branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Menu becomes stale unless manually updated.
- **Sub-workflow reference:** None.

#### Send Blocked Message
- **Type and role:** `n8n-nodes-base.telegram`; warns customers that staff commands are not available in this bot.
- **Configuration choices:** Triggered when the message matches a staff-like `READY <id>` command.
- **Key expressions or variables used:** chat ID from `Route Message`.
- **Input and output connections:** Input from `Message Switch` blocked branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Only `READY <number>` is explicitly blocked; other staff command variants are not handled.
- **Sub-workflow reference:** None.

---

## 2.3 AI-Based Order Parsing and Order Creation

### Overview
This block interprets non-command text as a possible restaurant order. It uses Claude Haiku to determine whether the text is a valid order and, if so, creates order metadata, saves the row to Google Sheets, and confirms the order to the customer.

### Nodes Involved
- AI Parse Order
- Claude Haiku
- Build Order Response
- Save Order
- Send Confirmation

### Node Details

#### AI Parse Order
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; AI agent wrapper used to parse natural-language food orders.
- **Configuration choices:**
  - Prompt asks for raw single-line JSON only.
  - Expected output format: `{"items":"clean readable order summary","valid":true}`
  - Uses a system message enforcing JSON-only output.
- **Key expressions or variables used:**
  - Message text injected from `{{ $('Route Message').first().json.text }}`
- **Input and output connections:** Input from `Message Switch` order branch; receives model connection from `Claude Haiku`; outputs to `Build Order Response`.
- **Version-specific requirements:** Type version `3.1`; requires the LangChain-compatible n8n AI nodes package.
- **Edge cases / failures:**
  - Model may still return extra text, malformed JSON, or markdown despite prompt constraints.
  - Anthropic credential/model access required.
  - Latency or API quota issues can delay responses.
- **Sub-workflow reference:** None.

#### Claude Haiku
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; Anthropic chat model provider for the agent node.
- **Configuration choices:** Uses model `claude-haiku-4-5-20251001`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to `AI Parse Order`.
- **Version-specific requirements:** Type version `1.3`; requires Anthropic credentials and access to the specified model.
- **Edge cases / failures:**
  - Model name availability may differ by account or release state.
  - Authentication, billing, or model deprecation errors.
- **Sub-workflow reference:** None.

#### Build Order Response
- **Type and role:** `n8n-nodes-base.code`; parses AI output, handles invalid or malformed responses, and prepares the order confirmation payload.
- **Configuration choices:**
  - Tries `JSON.parse` directly from `output`.
  - Falls back to extracting the first `{...}` block with regex.
  - If parsing still fails, defaults to `{ items: original_text, valid: true }`.
  - If `valid` is false, builds a polite correction message.
  - If valid:
    - Generates queue number with `Math.floor(Date.now() / 1000) % 9000 + 1000`
    - Generates estimated wait time between 10 and 19 minutes
    - Sets status to `Pending`
    - Adds ISO order time and localized `en-IN` date
- **Key expressions or variables used:**
  - `$('Route Message').first().json`
  - `aiRaw = $input.first().json.output || ''`
- **Input and output connections:** Input from `AI Parse Order`; output to `Save Order`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Queue generation is not guaranteed unique.
  - Fallback behavior may incorrectly treat random text as a valid order if AI output is malformed.
  - `toLocaleDateString('en-IN')` and time formatting depend on runtime locale support.
- **Sub-workflow reference:** None.

#### Save Order
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a new customer order row to Google Sheets.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `Restaurant Orders`
  - Sheet: `Sheet1`
  - Mapped columns:
    - `Name`
    - `Order`
    - `Status` = `Pending`
    - `Chat ID`
    - `Order Date`
    - `Order Time`
    - `Queue Number`
- **Key expressions or variables used:**
  - `={{ $json.first_name }}`
  - `={{ $json.parsed_order }}`
  - `={{ $json.chat_id }}`
  - `={{ $json.order_date }}`
  - `={{ $json.order_time }}`
  - `={{ $json.queue_number }}`
- **Input and output connections:** Input from `Build Order Response`; output to `Send Confirmation`.
- **Version-specific requirements:** Type version `4.5`; requires Google Sheets OAuth2 credentials.
- **Edge cases / failures:**
  - Sheet schema must contain exactly the referenced columns.
  - Missing permissions or spreadsheet relocation can break the node.
  - If `Build Order Response` produced an invalid-order reply with no sheet columns, this node may fail because the flow does not branch on `is_order`.
- **Sub-workflow reference:** None.

#### Send Confirmation
- **Type and role:** `n8n-nodes-base.telegram`; sends the order confirmation message to the customer.
- **Configuration choices:** Sends the `reply` built in `Build Order Response`.
- **Key expressions or variables used:**
  - `={{ $('Build Order Response').first().json.reply }}`
  - `={{ $('Build Order Response').first().json.chat_id }}`
- **Input and output connections:** Input from `Save Order`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Because it depends on `Save Order`, a Google Sheets failure prevents confirmation delivery.
- **Sub-workflow reference:** None.

---

## 2.4 Order Cancellation Flow

### Overview
This block validates customer cancellation requests against the spreadsheet. It ensures the order exists, belongs to the same Telegram user, and is still cancellable before updating the sheet.

### Nodes Involved
- Find Order to Cancel
- Validate Cancel
- Can Cancel?
- Mark as Cancelled
- Send Cancel Reply

### Node Details

#### Find Order to Cancel
- **Type and role:** `n8n-nodes-base.googleSheets`; fetches the first row matching the requested queue number.
- **Configuration choices:**
  - Lookup filter on `Queue Number`
  - `returnFirstMatch: true`
- **Key expressions or variables used:** `={{ $json.queue_match }}`
- **Input and output connections:** Input from `Message Switch` cancel branch; output to `Validate Cancel`.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - Duplicate queue numbers would return only the first match.
  - If no row is found, downstream code handles it.
- **Sub-workflow reference:** None.

#### Validate Cancel
- **Type and role:** `n8n-nodes-base.code`; business-rule validator for cancellation requests.
- **Configuration choices:**
  - Reads current order row and route metadata.
  - Validates:
    - order exists
    - order belongs to same `chat_id`
    - status is not `Completed`
    - status is not `Cancelled`
    - status is not `Ready`
    - status is not `Preparing`
  - Only `Pending` effectively remains cancellable.
- **Key expressions or variables used:**
  - `$('Route Message').first().json`
  - row fields like `['Queue Number']`, `['Order']`, `['Status']`, `['Chat ID']`
- **Input and output connections:** Input from `Find Order to Cancel`; output to `Can Cancel?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Assumes exact status spelling/casing.
  - If sheet data contains unexpected statuses, they may pass through as cancellable unless explicitly blocked.
- **Sub-workflow reference:** None.

#### Can Cancel?
- **Type and role:** `n8n-nodes-base.if`; branches on boolean `can_cancel`.
- **Configuration choices:** True branch when `={{ $json.can_cancel }}` equals boolean `true`.
- **Key expressions or variables used:** `can_cancel`
- **Input and output connections:**
  - Input from `Validate Cancel`
  - True output to `Mark as Cancelled`
  - False output to `Send Cancel Reply`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases / failures:**
  - Expression type mismatch if upstream returns a non-boolean value.
- **Sub-workflow reference:** None.

#### Mark as Cancelled
- **Type and role:** `n8n-nodes-base.googleSheets`; updates the matched order row status to `Cancelled`.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `Queue Number`
  - Sets `Status = Cancelled`
- **Key expressions or variables used:** `={{ $('Validate Cancel').first().json.queue_number }}`
- **Input and output connections:** Input from `Can Cancel?` true branch; output to `Send Cancel Reply`.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - Duplicate queue numbers can update multiple unintended rows depending on Google Sheets node behavior.
  - Missing match column or changed schema breaks the update.
- **Sub-workflow reference:** None.

#### Send Cancel Reply
- **Type and role:** `n8n-nodes-base.telegram`; sends the outcome of the cancellation attempt.
- **Configuration choices:** Markdown reply from `Validate Cancel`.
- **Key expressions or variables used:**
  - `={{ $('Validate Cancel').first().json.reply }}`
  - `={{ $('Validate Cancel').first().json.chat_id }}`
- **Input and output connections:**
  - Input from `Can Cancel?` false branch
  - Also input from `Mark as Cancelled`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Markdown formatting issues if order text contains Telegram special characters.
- **Sub-workflow reference:** None.

---

## 2.5 Order Status Lookup Flow

### Overview
This block supports customer self-service status checks for a specific queue number. It reads the order from Google Sheets and formats a status message with emoji and optional time information.

### Nodes Involved
- Find Order Status
- Build Status Reply
- Send Status

### Node Details

#### Find Order Status
- **Type and role:** `n8n-nodes-base.googleSheets`; retrieves the matching row for a requested queue number.
- **Configuration choices:**
  - Filter on `Queue Number`
  - `returnFirstMatch: true`
- **Key expressions or variables used:** `={{ $json.queue_match }}`
- **Input and output connections:** Input from `Message Switch` status branch; output to `Build Status Reply`.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - Duplicate queue numbers create ambiguity.
- **Sub-workflow reference:** None.

#### Build Status Reply
- **Type and role:** `n8n-nodes-base.code`; formats a customer-facing status message.
- **Configuration choices:**
  - If no row found, returns not-found response.
  - Maps statuses to emoji:
    - Completed ✅
    - Cancelled ❌
    - Pending ⏳
    - Ready 🍕
    - Preparing 👨‍🍳
  - Tries to parse `Order Time` and display local time in `en-IN` format.
- **Key expressions or variables used:**
  - `$('Route Message').first().json`
  - row fields like `['Status']`, `['Order']`, `['Name']`, `['Order Time']`
- **Input and output connections:** Input from `Find Order Status`; output to `Send Status`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - It does not validate ownership, so any user knowing a queue number can view that order’s status and customer name.
  - Invalid `Order Time` values are silently ignored.
- **Sub-workflow reference:** None.

#### Send Status
- **Type and role:** `n8n-nodes-base.telegram`; delivers the formatted order status message.
- **Configuration choices:** Markdown output.
- **Key expressions or variables used:**
  - `={{ $json.reply }}`
  - `={{ $json.chat_id }}`
- **Input and output connections:** Input from `Build Status Reply`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Markdown escaping issues if order text contains unsupported special characters.
- **Sub-workflow reference:** None.

---

## 2.6 Customer Order History Flow

### Overview
This block returns the last five orders associated with the current Telegram chat ID. It reads all spreadsheet rows, filters them in code, and sends a compact order history message.

### Nodes Involved
- Read All Orders
- Build My Orders
- Send My Orders

### Node Details

#### Read All Orders
- **Type and role:** `n8n-nodes-base.googleSheets`; reads all order rows from the sheet.
- **Configuration choices:** Simple sheet read from `Restaurant Orders / Sheet1`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Message Switch` myorders branch; output to `Build My Orders`.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - Can become slower as sheet size grows.
- **Sub-workflow reference:** None.

#### Build My Orders
- **Type and role:** `n8n-nodes-base.code`; filters all rows to the current chat ID and formats up to five recent orders.
- **Configuration choices:**
  - `allRows.filter(...)`
  - `slice(-5).reverse()` for last five, newest first
  - includes queue number, order text, status, and date
- **Key expressions or variables used:**
  - `$('Route Message').first().json.chat_id`
  - row field `['Chat ID']`
- **Input and output connections:** Input from `Read All Orders`; output to `Send My Orders`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Assumes sheet row order reflects chronology.
  - If rows are sorted manually in Sheets, "recent" may no longer be accurate.
- **Sub-workflow reference:** None.

#### Send My Orders
- **Type and role:** `n8n-nodes-base.telegram`; sends recent-order history to the customer.
- **Configuration choices:** Markdown message output.
- **Key expressions or variables used:**
  - `={{ $json.reply }}`
  - `={{ $json.chat_id }}`
- **Input and output connections:** Input from `Build My Orders`; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Markdown formatting may break if order content contains special characters.
- **Sub-workflow reference:** None.

---

## 2.7 Scheduled Staff Status Notification Flow

### Overview
This is the second workflow path, triggered by time rather than Telegram input. It scans the orders sheet for rows whose current status has changed since the last customer notification, sends the appropriate Telegram status update, then records that notification in column H.

### Nodes Involved
- Every 1 Minute
- Read All Rows
- Detect Changed Row
- Build Message
- Send to Customer
- Prep Sheet Update
- Save Last Status Sent

### Node Details

#### Every 1 Minute
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; periodic trigger for the status notifier flow.
- **Configuration choices:** Runs every 1 minute.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point to `Read All Rows`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Missed schedules if workflow is inactive or server unavailable.
- **Sub-workflow reference:** None.

#### Read All Rows
- **Type and role:** `n8n-nodes-base.googleSheets`; loads all order rows for change detection.
- **Configuration choices:** Reads the same `Restaurant Orders / Sheet1`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Every 1 Minute`; output to `Detect Changed Row`.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases / failures:**
  - Sheet performance may degrade as the dataset grows.
- **Sub-workflow reference:** None.

#### Detect Changed Row
- **Type and role:** `n8n-nodes-base.code`; per-row change detector.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Reads:
    - `row_number`
    - `Queue Number`
    - `Chat ID`
    - `Status`
    - `Last Status Sent`
    - `Order`
    - `Name`
  - Skips rows when:
    - queue/chat missing
    - `chatId === '0'`
    - status not in `Preparing`, `Ready`, `Completed`, `Cancelled`
    - status equals `Last Status Sent`
  - Emits rows needing notification
- **Key expressions or variables used:** row field names from Sheets.
- **Input and output connections:** Input from `Read All Rows`; output to `Build Message`.
- **Version-specific requirements:** Type version `2`, mode `runOnceForEachItem`.
- **Edge cases / failures:**
  - Requires row metadata field `row_number` to be present from the Sheets node.
  - Status names must exactly match expected strings.
  - Pending notifications are intentionally ignored.
- **Sub-workflow reference:** None.

#### Build Message
- **Type and role:** `n8n-nodes-base.code`; creates status-specific Telegram text for each changed row.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Templates:
    - Preparing
    - Ready
    - Completed
    - Cancelled
- **Key expressions or variables used:** `$input.item.json.status`, `order`, `queue_number`
- **Input and output connections:** Input from `Detect Changed Row`; output to `Send to Customer`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - If an unexpected status gets through, `msg` may be undefined.
- **Sub-workflow reference:** None.

#### Send to Customer
- **Type and role:** `n8n-nodes-base.telegram`; sends the status-change notification.
- **Configuration choices:** Markdown parse mode, chat ID from row data.
- **Key expressions or variables used:**
  - `={{ $json.msg }}`
  - `={{ $json.chat_id }}`
- **Input and output connections:** Input from `Build Message`; output to `Prep Sheet Update`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - If Telegram send fails, the workflow will not mark `Last Status Sent`, which is actually desirable because it allows retry on the next run.
- **Sub-workflow reference:** None.

#### Prep Sheet Update
- **Type and role:** `n8n-nodes-base.code`; prepares a direct Google Sheets API update request for column H.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Builds:
    - A target range `Sheet1!H{row_number}`
    - Google Sheets API URL
    - Body with `values:[[status]]`
- **Key expressions or variables used:**
  - `$('Build Message').item.json`
  - hard-coded spreadsheet ID
  - hard-coded sheet name `Sheet1`
- **Input and output connections:** Input from `Send to Customer`; output to `Save Last Status Sent`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Assumes column H is `Last Status Sent`.
  - Breaks if sheet name changes.
  - Uses a hard-coded spreadsheet ID rather than inheriting from node settings.
- **Sub-workflow reference:** None.

#### Save Last Status Sent
- **Type and role:** `n8n-nodes-base.httpRequest`; updates Google Sheets directly via REST API.
- **Configuration choices:**
  - Method: `PUT`
  - Authentication: predefined credential type `googleSheetsOAuth2Api`
  - Sends JSON body prepared upstream
- **Key expressions or variables used:**
  - `={{ $json._url }}`
  - `={{ JSON.stringify($json._body) }}`
- **Input and output connections:** Input from `Prep Sheet Update`; no downstream node.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Requires OAuth scope sufficient for direct Sheets API write access.
  - If Google API returns a 403 or 404, duplicate notifications will continue on future schedule runs.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Customer Bot Listener | n8n-nodes-base.telegramTrigger | Telegram entry point for customer messages |  | Route Message | ## 🍕 WORKFLOW 1 — Customer Bot<br>**Telegram bot for customers to place and manage orders**<br><br>**Bot:** @new_nirav_restaurant_bot<br>**Sheet:** Restaurant Orders (1_XaI7z844...)<br><br>**How it works:**<br>Customer texts the bot → Route Message detects what they want → routes to correct handler<br><br>## 📥 Entry Point<br>Listens to ALL messages from the customer bot.<br><br>Extracts: chat_id, text, first_name<br>Detects commands: /start, /help, /menu, /myorders, STATUS, CANCEL<br>Default route = **order** (food order) |
| Route Message | n8n-nodes-base.code | Normalize Telegram payload and classify intent | Customer Bot Listener | Message Switch | ## 🍕 WORKFLOW 1 — Customer Bot<br>**Telegram bot for customers to place and manage orders**<br><br>**Bot:** @new_nirav_restaurant_bot<br>**Sheet:** Restaurant Orders (1_XaI7z844...)<br><br>**How it works:**<br>Customer texts the bot → Route Message detects what they want → routes to correct handler<br><br>## 📥 Entry Point<br>Listens to ALL messages from the customer bot.<br><br>Extracts: chat_id, text, first_name<br>Detects commands: /start, /help, /menu, /myorders, STATUS, CANCEL<br>Default route = **order** (food order) |
| Message Switch | n8n-nodes-base.switch | Route customer message to the correct flow | Route Message | Send Welcome; Send Help; AI Parse Order; Find Order to Cancel; Find Order Status; Send Menu; Read All Orders; Send Blocked Message | ## 🔀 Message Router<br>Routes to one of 8 paths:<br>0. /start → Welcome message<br>1. /help → Help message<br>2. order → AI Parse Order<br>3. cancel → Cancel flow<br>4. status → Status check<br>5. /menu → Menu<br>6. /myorders → Order history<br>7. blocked → Staff cmd blocked |
| Send Welcome | n8n-nodes-base.telegram | Send onboarding message | Message Switch |  |  |
| Send Help | n8n-nodes-base.telegram | Send usage help | Message Switch |  |  |
| AI Parse Order | @n8n/n8n-nodes-langchain.agent | Parse free-text food order with AI | Message Switch; Claude Haiku | Build Order Response | ## 🤖 AI Order Parsing<br>Uses **Claude Haiku** to parse natural language orders.<br><br>Input: "2 pizza + 1 coke"<br>Output:<br>`{<br>"items": "2 Margherita Pizza + 1 Coke",<br>"valid": true<br>}`<br><br>If not a food order → returns valid:<br>false → sends error message to customer |
| Claude Haiku | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic model provider for order parsing |  | AI Parse Order | ## 🤖 AI Order Parsing<br>Uses **Claude Haiku** to parse natural language orders.<br><br>Input: "2 pizza + 1 coke"<br>Output:<br>`{<br>"items": "2 Margherita Pizza + 1 Coke",<br>"valid": true<br>}`<br><br>If not a food order → returns valid:<br>false → sends error message to customer |
| Build Order Response | n8n-nodes-base.code | Parse AI output and prepare order metadata/reply | AI Parse Order | Save Order | ## 🤖 AI Order Parsing<br>Uses **Claude Haiku** to parse natural language orders.<br><br>Input: "2 pizza + 1 coke"<br>Output:<br>`{<br>"items": "2 Margherita Pizza + 1 Coke",<br>"valid": true<br>}`<br><br>If not a food order → returns valid:<br>false → sends error message to customer |
| Save Order | n8n-nodes-base.googleSheets | Append new order row to Google Sheets | Build Order Response | Send Confirmation | ## 💾 Save Order<br>Appends new order to Google Sheet with:<br>- Queue Number (auto-generated)<br>- Chat ID (for later notifications)<br>- Name, Order, Status=Pending<br>- Order Time, Order Date<br><br>⚠️ Column H (Last Status Sent) stays empty — filled by W2 |
| Send Confirmation | n8n-nodes-base.telegram | Send order confirmation to customer | Save Order |  | ## 💾 Save Order<br>Appends new order to Google Sheet with:<br>- Queue Number (auto-generated)<br>- Chat ID (for later notifications)<br>- Name, Order, Status=Pending<br>- Order Time, Order Date<br><br>⚠️ Column H (Last Status Sent) stays empty — filled by W2 |
| Find Order to Cancel | n8n-nodes-base.googleSheets | Find row for cancellation request | Message Switch | Validate Cancel |  |
| Validate Cancel | n8n-nodes-base.code | Enforce cancellation business rules | Find Order to Cancel | Can Cancel? |  |
| Can Cancel? | n8n-nodes-base.if | Branch based on whether cancellation is allowed | Validate Cancel | Mark as Cancelled; Send Cancel Reply |  |
| Mark as Cancelled | n8n-nodes-base.googleSheets | Update sheet status to Cancelled | Can Cancel? | Send Cancel Reply | ## ❌ Cancel Flow<br>Validation rules:<br>✅ Pending → can cancel<br>🚫 Preparing → cannot cancel<br>🚫 Ready → cannot cancel<br>🚫 Completed → cannot cancel<br>🚫 Cancelled → already cancelled<br>🚫 Other person's order → blocked<br><br>If valid → updates sheet Status to Cancelled |
| Send Cancel Reply | n8n-nodes-base.telegram | Send cancellation outcome | Can Cancel?; Mark as Cancelled |  | ## ❌ Cancel Flow<br>Validation rules:<br>✅ Pending → can cancel<br>🚫 Preparing → cannot cancel<br>🚫 Ready → cannot cancel<br>🚫 Completed → cannot cancel<br>🚫 Cancelled → already cancelled<br>🚫 Other person's order → blocked<br><br>If valid → updates sheet Status to Cancelled |
| Find Order Status | n8n-nodes-base.googleSheets | Look up order row by queue number | Message Switch | Build Status Reply | ## 📊 Order Status Check<br>Customer types: STATUS 6771<br>→ Looks up Queue Number in sheet<br>→ Returns current status with emoji:<br>⏳ Pending  👨‍🍳 Preparing<br>🍕 Ready   ✅ Completed  ❌ Cancelled |
| Build Status Reply | n8n-nodes-base.code | Format order status response | Find Order Status | Send Status | ## 📊 Order Status Check<br>Customer types: STATUS 6771<br>→ Looks up Queue Number in sheet<br>→ Returns current status with emoji:<br>⏳ Pending  👨‍🍳 Preparing<br>🍕 Ready   ✅ Completed  ❌ Cancelled |
| Send Status | n8n-nodes-base.telegram | Send formatted status message | Build Status Reply |  | ## 📊 Order Status Check<br>Customer types: STATUS 6771<br>→ Looks up Queue Number in sheet<br>→ Returns current status with emoji:<br>⏳ Pending  👨‍🍳 Preparing<br>🍕 Ready   ✅ Completed  ❌ Cancelled |
| Send Menu | n8n-nodes-base.telegram | Send static restaurant menu | Message Switch |  |  |
| Read All Orders | n8n-nodes-base.googleSheets | Read all orders for customer history lookup | Message Switch | Build My Orders | ## 📋 My Orders<br>Shows last 5 orders for this customer.<br>Filters all sheet rows by Chat ID.<br>Displays: queue number, order, status, date |
| Build My Orders | n8n-nodes-base.code | Build last-5-orders response | Read All Orders | Send My Orders | ## 📋 My Orders<br>Shows last 5 orders for this customer.<br>Filters all sheet rows by Chat ID.<br>Displays: queue number, order, status, date |
| Send My Orders | n8n-nodes-base.telegram | Send recent order history to customer | Build My Orders |  | ## 📋 My Orders<br>Shows last 5 orders for this customer.<br>Filters all sheet rows by Chat ID.<br>Displays: queue number, order, status, date |
| Send Blocked Message | n8n-nodes-base.telegram | Block customer use of staff-style commands | Message Switch |  |  |
| Every 1 Minute | n8n-nodes-base.scheduleTrigger | Scheduled entry point for staff status notifications |  | Read All Rows | ## 👨‍🍳 WORKFLOW 2 — Staff Status Notifier<br>**Auto-notifies customers when staff changes order status in Google Sheet**<br><br>**How it works:**<br>Runs every minute → reads all rows → compares Status vs Last Status Sent (col H)<br><br> → sends Telegram message for changed rows only → updates col H<br><br>## ⏱️ Schedule Trigger<br>Runs **every 1 minute**.<br><br>Cannot use Google Sheets trigger<br><br>because it cannot detect which specific row changed — it always returns row 1. |
| Read All Rows | n8n-nodes-base.googleSheets | Read all rows for status change detection | Every 1 Minute | Detect Changed Row |  |
| Detect Changed Row | n8n-nodes-base.code | Filter rows needing notification | Read All Rows | Build Message | ## 🔍 Detect Changed Row<br>**runOnceForEachItem** — processes every row independently.<br><br>Logic per row:<br>1. Skip if no Queue Number or Chat ID<br>2. Skip if Status = Pending (not notifiable)<br>3. Skip if Status = Last Status Sent (already notified)<br>4. ✅ Pass through if Status changed<br><br>Only changed rows flow forward. |
| Build Message | n8n-nodes-base.code | Build Telegram text for status changes | Detect Changed Row | Send to Customer | ## 💬 Message Templates<br>Builds message based on new status:<br>👨‍🍳 Preparing → "Being prepared!"<br>🍕 Ready → "READY for collection!"<br>✅ Completed → "Thank you!"<br>❌ Cancelled → "Order cancelled" |
| Send to Customer | n8n-nodes-base.telegram | Notify customer of status change | Build Message | Prep Sheet Update | ## 💬 Message Templates<br>Builds message based on new status:<br>👨‍🍳 Preparing → "Being prepared!"<br>🍕 Ready → "READY for collection!"<br>✅ Completed → "Thank you!"<br>❌ Cancelled → "Order cancelled" |
| Prep Sheet Update | n8n-nodes-base.code | Build direct Google Sheets API request to write column H | Send to Customer | Save Last Status Sent | ## 📝 Update Last Status Sent<br>Writes current Status into **column H** (Last Status Sent) of the exact row using Google Sheets HTTP API.<br><br>This prevents duplicate notifications — next minute this row will be skipped because Status = Last Status Sent.<br><br>⚠️ Column H must exist in your sheet! |
| Save Last Status Sent | n8n-nodes-base.httpRequest | Persist last-sent status marker to Google Sheets | Prep Sheet Update |  | ## 📝 Update Last Status Sent<br>Writes current Status into **column H** (Last Status Sent) of the exact row using Google Sheets HTTP API.<br><br>This prevents duplicate notifications — next minute this row will be skipped because Status = Last Status Sent.<br><br>⚠️ Column H must exist in your sheet! |
| W1 Title | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 🍕 WORKFLOW 1 — Customer Bot<br>**Telegram bot for customers to place and manage orders**<br><br>**Bot:** @new_nirav_restaurant_bot<br>**Sheet:** Restaurant Orders (1_XaI7z844...)<br><br>**How it works:**<br>Customer texts the bot → Route Message detects what they want → routes to correct handler |
| Entry Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 📥 Entry Point<br>Listens to ALL messages from the customer bot.<br><br>Extracts: chat_id, text, first_name<br>Detects commands: /start, /help, /menu, /myorders, STATUS, CANCEL<br>Default route = **order** (food order) |
| Switch Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 🔀 Message Router<br>Routes to one of 8 paths:<br>0. /start → Welcome message<br>1. /help → Help message<br>2. order → AI Parse Order<br>3. cancel → Cancel flow<br>4. status → Status check<br>5. /menu → Menu<br>6. /myorders → Order history<br>7. blocked → Staff cmd blocked |
| AI Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 🤖 AI Order Parsing<br>Uses **Claude Haiku** to parse natural language orders.<br><br>Input: "2 pizza + 1 coke"<br>Output:<br>`{<br>"items": "2 Margherita Pizza + 1 Coke",<br>"valid": true<br>}`<br><br>If not a food order → returns valid:<br>false → sends error message to customer |
| Save Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 💾 Save Order<br>Appends new order to Google Sheet with:<br>- Queue Number (auto-generated)<br>- Chat ID (for later notifications)<br>- Name, Order, Status=Pending<br>- Order Time, Order Date<br><br>⚠️ Column H (Last Status Sent) stays empty — filled by W2 |
| Cancel Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## ❌ Cancel Flow<br>Validation rules:<br>✅ Pending → can cancel<br>🚫 Preparing → cannot cancel<br>🚫 Ready → cannot cancel<br>🚫 Completed → cannot cancel<br>🚫 Cancelled → already cancelled<br>🚫 Other person's order → blocked<br><br>If valid → updates sheet Status to Cancelled |
| Status Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 📊 Order Status Check<br>Customer types: STATUS 6771<br>→ Looks up Queue Number in sheet<br>→ Returns current status with emoji:<br>⏳ Pending  👨‍🍳 Preparing<br>🍕 Ready   ✅ Completed  ❌ Cancelled |
| MyOrders Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 📋 My Orders<br>Shows last 5 orders for this customer.<br>Filters all sheet rows by Chat ID.<br>Displays: queue number, order, status, date |
| W2 Title | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 👨‍🍳 WORKFLOW 2 — Staff Status Notifier<br>**Auto-notifies customers when staff changes order status in Google Sheet**<br><br>**How it works:**<br>Runs every minute → reads all rows → compares Status vs Last Status Sent (col H)<br><br> → sends Telegram message for changed rows only → updates col H |
| Schedule Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## ⏱️ Schedule Trigger<br>Runs **every 1 minute**.<br><br>Cannot use Google Sheets trigger<br><br>because it cannot detect which specific row changed — it always returns row 1. |
| Detect Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 🔍 Detect Changed Row<br>**runOnceForEachItem** — processes every row independently.<br><br>Logic per row:<br>1. Skip if no Queue Number or Chat ID<br>2. Skip if Status = Pending (not notifiable)<br>3. Skip if Status = Last Status Sent (already notified)<br>4. ✅ Pass through if Status changed<br><br>Only changed rows flow forward. |
| MsgBuild Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 💬 Message Templates<br>Builds message based on new status:<br>👨‍🍳 Preparing → "Being prepared!"<br>🍕 Ready → "READY for collection!"<br>✅ Completed → "Thank you!"<br>❌ Cancelled → "Order cancelled" |
| UpdateH Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## 📝 Update Last Status Sent<br>Writes current Status into **column H** (Last Status Sent) of the exact row using Google Sheets HTTP API.<br><br>This prevents duplicate notifications — next minute this row will be skipped because Status = Last Status Sent.<br><br>⚠️ Column H must exist in your sheet! |

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites

1. Create a Telegram bot with BotFather.
2. Prepare Google Sheets OAuth2 credentials in n8n.
3. Prepare Anthropic credentials in n8n.
4. Create a Google Sheet named something like `Restaurant Orders`.
5. In `Sheet1`, create these columns in this order or at minimum with these exact names:
   - `Queue Number`
   - `Chat ID`
   - `Name`
   - `Order`
   - `Status`
   - `Order Time`
   - `Order Date`
   - `Last Status Sent`

> Important: the scheduled notification flow assumes `Last Status Sent` is in column H.

---

## Build Workflow 1: Customer Bot

### 1. Add the Telegram trigger
1. Create a **Telegram Trigger** node named **Customer Bot Listener**.
2. Select your Telegram bot credentials.
3. Set **Updates** to `message`.

### 2. Add the message-normalization code node
4. Create a **Code** node named **Route Message**.
5. Connect **Customer Bot Listener → Route Message**.
6. Paste logic that:
   - extracts `text`, `chat_id`, `first_name`
   - uppercases text
   - detects:
     - `/start`
     - `/help` or `help`
     - `/menu` or `menu`
     - `/myorders` or `myorders`
     - `STATUS <number>`
     - `CANCEL <number>`
     - `READY <number>` as blocked
   - defaults to `route = 'order'`
   - outputs:
     - `chat_id`
     - `text`
     - `first_name`
     - `upper`
     - `route`
     - `queue_match`

### 3. Add the switch router
7. Create a **Switch** node named **Message Switch**.
8. Connect **Route Message → Message Switch**.
9. Configure eight rules on `{{$json.route}}`:
   - equals `start`
   - equals `help`
   - equals `order`
   - equals `cancel`
   - equals `status`
   - equals `menu`
   - equals `myorders`
   - equals `blocked`

---

## Build static response branches

### 4. Welcome branch
10. Create a **Telegram** node named **Send Welcome**.
11. Connect output 0 of **Message Switch** to it.
12. Set **Chat ID** to `{{ $('Route Message').first().json.chat_id }}`.
13. Enable Markdown parse mode.
14. Add a welcome message introducing the order bot and supported commands.

### 5. Help branch
15. Create a **Telegram** node named **Send Help**.
16. Connect output 1 of **Message Switch**.
17. Set **Chat ID** to `{{ $('Route Message').first().json.chat_id }}`.
18. Enable Markdown parse mode.
19. Add help text for placing orders, checking status, canceling, and viewing menu/orders.

### 6. Menu branch
20. Create a **Telegram** node named **Send Menu**.
21. Connect output 5 of **Message Switch**.
22. Set **Chat ID** to `{{ $('Route Message').first().json.chat_id }}`.
23. Enable Markdown parse mode.
24. Add a static menu text.

### 7. Blocked-command branch
25. Create a **Telegram** node named **Send Blocked Message**.
26. Connect output 7 of **Message Switch**.
27. Set **Chat ID** to `{{ $('Route Message').first().json.chat_id }}`.
28. Enable Markdown parse mode.
29. Add a message explaining that staff commands are not available to customers.

---

## Build AI order branch

### 8. Add the Anthropic model node
30. Create an **Anthropic Chat Model** node named **Claude Haiku**.
31. Select your Anthropic credentials.
32. Choose model `claude-haiku-4-5-20251001` or the closest currently available Claude Haiku model in your account.

### 9. Add the AI agent/parser node
33. Create an **AI Agent** node named **AI Parse Order**.
34. Connect output 2 of **Message Switch** to it.
35. Connect **Claude Haiku** to the AI Agent’s language model input.
36. Configure the prompt to instruct the model:
   - parse restaurant orders
   - return only raw JSON
   - single line only
   - format like `{"items":"clean readable order summary","valid":true}`
   - return `valid:false` for non-order text
37. Inject the customer message with `{{ $('Route Message').first().json.text }}`.
38. Add a system message reinforcing “JSON only, no markdown”.

### 10. Build order response payload
39. Create a **Code** node named **Build Order Response**.
40. Connect **AI Parse Order → Build Order Response**.
41. Implement logic to:
   - read `output` from AI result
   - try to parse JSON directly
   - fallback to regex extraction of JSON object
   - if parsing fails entirely, fallback to original text as valid order
   - if `valid` is false, build a rejection/help reply
   - if `valid` is true:
     - generate a queue number
     - generate wait minutes
     - create a reply message
     - set:
       - `chat_id`
       - `first_name`
       - `parsed_order`
       - `queue_number`
       - `wait_mins`
       - `status = Pending`
       - `order_time` as ISO string
       - `order_date` using locale formatting

### 11. Save the order to Google Sheets
42. Create a **Google Sheets** node named **Save Order**.
43. Connect **Build Order Response → Save Order**.
44. Use your Google Sheets credentials.
45. Choose the spreadsheet and `Sheet1`.
46. Set **Operation** to `Append`.
47. Use manual field mapping:
   - `Name` = `{{ $json.first_name }}`
   - `Order` = `{{ $json.parsed_order }}`
   - `Status` = `Pending`
   - `Chat ID` = `{{ $json.chat_id }}`
   - `Order Date` = `{{ $json.order_date }}`
   - `Order Time` = `{{ $json.order_time }}`
   - `Queue Number` = `{{ $json.queue_number }}`

### 12. Send confirmation
48. Create a **Telegram** node named **Send Confirmation**.
49. Connect **Save Order → Send Confirmation**.
50. Set **Chat ID** to `{{ $('Build Order Response').first().json.chat_id }}`.
51. Set **Text** to `{{ $('Build Order Response').first().json.reply }}`.

> Recommended improvement: add an IF node before `Save Order` to avoid attempting to append invalid orders. The provided workflow does not do this, which can cause failures when AI returns `valid:false`.

---

## Build cancellation branch

### 13. Find order by queue number
52. Create a **Google Sheets** node named **Find Order to Cancel**.
53. Connect output 3 of **Message Switch**.
54. Configure it to search `Sheet1` where `Queue Number = {{ $json.queue_match }}`.
55. Enable **Return First Match**.

### 14. Validate cancel eligibility
56. Create a **Code** node named **Validate Cancel**.
57. Connect **Find Order to Cancel → Validate Cancel**.
58. In code:
   - read route data from `Route Message`
   - confirm a row exists
   - compare row `Chat ID` with current `chat_id`
   - reject if status is:
     - `Completed`
     - `Cancelled`
     - `Ready`
     - `Preparing`
   - allow if still effectively `Pending`
   - output:
     - `chat_id`
     - `reply`
     - `can_cancel`
     - `queue_number` when allowed

### 15. Branch on cancel eligibility
59. Create an **If** node named **Can Cancel?**.
60. Connect **Validate Cancel → Can Cancel?**.
61. Configure condition:
   - `{{ $json.can_cancel }}` equals boolean `true`.

### 16. Update sheet status for valid cancellation
62. Create a **Google Sheets** node named **Mark as Cancelled**.
63. Connect the **true** output of **Can Cancel?**.
64. Set **Operation** to `Update`.
65. Match on `Queue Number`.
66. Map:
   - `Queue Number` = `{{ $('Validate Cancel').first().json.queue_number }}`
   - `Status` = `Cancelled`

### 17. Send cancel response
67. Create a **Telegram** node named **Send Cancel Reply**.
68. Connect both:
   - **false** output of **Can Cancel?**
   - **Mark as Cancelled**
69. Set **Chat ID** to `{{ $('Validate Cancel').first().json.chat_id }}`.
70. Set **Text** to `{{ $('Validate Cancel').first().json.reply }}`.
71. Enable Markdown parse mode.

---

## Build status lookup branch

### 18. Find order status
72. Create a **Google Sheets** node named **Find Order Status**.
73. Connect output 4 of **Message Switch**.
74. Search `Queue Number = {{ $json.queue_match }}`.
75. Enable **Return First Match**.

### 19. Build status reply
76. Create a **Code** node named **Build Status Reply**.
77. Connect **Find Order Status → Build Status Reply**.
78. Implement logic to:
   - return not-found if no row exists
   - read `Status`, `Order`, `Name`, `Order Time`
   - map emoji by status
   - format optional order time
   - return `chat_id` and `reply`

### 20. Send status
79. Create a **Telegram** node named **Send Status**.
80. Connect **Build Status Reply → Send Status**.
81. Set **Chat ID** to `{{ $json.chat_id }}`.
82. Set **Text** to `{{ $json.reply }}`.
83. Enable Markdown parse mode.

---

## Build my-orders branch

### 21. Read all orders
84. Create a **Google Sheets** node named **Read All Orders**.
85. Connect output 6 of **Message Switch**.
86. Read all rows from the same sheet.

### 22. Filter current customer’s orders
87. Create a **Code** node named **Build My Orders**.
88. Connect **Read All Orders → Build My Orders**.
89. In code:
   - get current `chat_id` from `Route Message`
   - filter rows where `Chat ID` matches
   - keep the last five rows
   - reverse them so newest comes first
   - build a Markdown reply

### 23. Send order history
90. Create a **Telegram** node named **Send My Orders**.
91. Connect **Build My Orders → Send My Orders**.
92. Set **Chat ID** to `{{ $json.chat_id }}`.
93. Set **Text** to `{{ $json.reply }}`.
94. Enable Markdown parse mode.

---

## Build Workflow 2: Scheduled staff status notifier

### 24. Add scheduled trigger
95. Create a **Schedule Trigger** node named **Every 1 Minute**.
96. Configure it to run every 1 minute.

### 25. Read all rows
97. Create a **Google Sheets** node named **Read All Rows**.
98. Connect **Every 1 Minute → Read All Rows**.
99. Read all rows from the same spreadsheet and sheet.

### 26. Detect changed rows
100. Create a **Code** node named **Detect Changed Row**.
101. Connect **Read All Rows → Detect Changed Row**.
102. Set mode to **Run Once for Each Item**.
103. In code, for each row:
   - read `row_number`
   - read `Queue Number`, `Chat ID`, `Status`, `Last Status Sent`, `Order`, `Name`
   - skip if queue/chat missing
   - skip if chat ID is `0`
   - skip if status is not one of:
     - `Preparing`
     - `Ready`
     - `Completed`
     - `Cancelled`
   - skip if `Status === Last Status Sent`
   - otherwise output row data for notification

### 27. Build status-change messages
104. Create a **Code** node named **Build Message**.
105. Connect **Detect Changed Row → Build Message**.
106. Set mode to **Run Once for Each Item**.
107. Build a message template per status:
   - Preparing: being prepared
   - Ready: ready for collection
   - Completed: thank-you/completed
   - Cancelled: cancellation message

### 28. Send Telegram notification
108. Create a **Telegram** node named **Send to Customer**.
109. Connect **Build Message → Send to Customer**.
110. Set **Chat ID** to `{{ $json.chat_id }}`.
111. Set **Text** to `{{ $json.msg }}`.
112. Enable Markdown parse mode.

### 29. Prepare direct Google Sheets API update
113. Create a **Code** node named **Prep Sheet Update**.
114. Connect **Send to Customer → Prep Sheet Update**.
115. Set mode to **Run Once for Each Item**.
116. In code:
   - get `row_number` and `status`
   - build range `Sheet1!H{row_number}`
   - build URL:
     `https://sheets.googleapis.com/v4/spreadsheets/<SPREADSHEET_ID>/values/<ENCODED_RANGE>?valueInputOption=RAW`
   - build body:
     - `range`
     - `majorDimension: 'ROWS'`
     - `values: [[status]]`
   - output `_url` and `_body`

### 30. Write Last Status Sent
117. Create an **HTTP Request** node named **Save Last Status Sent**.
118. Connect **Prep Sheet Update → Save Last Status Sent**.
119. Set:
   - **Method:** `PUT`
   - **URL:** `{{ $json._url }}`
   - **Authentication:** predefined credential type
   - **Credential type:** `googleSheetsOAuth2Api`
   - **Send Body:** true
   - **Body type:** JSON
   - **JSON Body:** `{{ JSON.stringify($json._body) }}`

---

## Credential configuration summary

### Telegram
- Use the same Telegram bot credentials for:
  - Customer Bot Listener
  - Send Welcome
  - Send Help
  - Send Confirmation
  - Send Cancel Reply
  - Send Status
  - Send Menu
  - Send My Orders
  - Send Blocked Message
  - Send to Customer

### Google Sheets
- Use Google Sheets OAuth2 credentials for:
  - Save Order
  - Find Order to Cancel
  - Mark as Cancelled
  - Find Order Status
  - Read All Orders
  - Read All Rows
  - Save Last Status Sent HTTP Request

### Anthropic
- Use Anthropic API credentials for:
  - Claude Haiku

---

## Important implementation constraints

1. **Spreadsheet columns must match exactly**:
   - Queue Number
   - Chat ID
   - Name
   - Order
   - Status
   - Order Time
   - Order Date
   - Last Status Sent

2. **Column H must be `Last Status Sent`** or the scheduled notifier will update the wrong column.

3. **Queue number generation is weakly unique** in the original logic. For production, use UUIDs, a sheet counter, or a timestamp-plus-random approach.

4. **Status flow assumes exact status values**:
   - Pending
   - Preparing
   - Ready
   - Completed
   - Cancelled

5. **The status lookup flow does not verify ownership.**
   Any user who knows a queue number can query that order’s status.

6. **The AI branch should ideally add a guard IF node** between `Build Order Response` and `Save Order`, checking `is_order === true`.

7. **Row ordering matters** for `/myorders`.
   If staff manually sort the sheet, the “last 5” logic may no longer represent the latest chronological orders.

8. **The scheduled flow uses direct Sheets API writes** with a hard-coded spreadsheet ID and sheet name. If you duplicate the workflow, update these values in `Prep Sheet Update`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Bot used in the workflow canvas notes: `@new_nirav_restaurant_bot` | Telegram bot reference |
| Google Sheet referenced in node configuration: `Restaurant Orders` | Spreadsheet ID: `1_XaI7z844arJWhzMN2MbD6kFzHoK6yUvsY1OyE10ebM` |
| Sheet tab used throughout the workflow | `Sheet1` |
| Scheduled notifier rationale | Google Sheets trigger is avoided because the note says it always returns row 1 rather than the specific changed row |
| Staff-status deduplication mechanism | Compare `Status` against `Last Status Sent`, then write the current status back to column H after notification |