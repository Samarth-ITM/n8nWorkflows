Handle e-commerce support, orders and inventory with Claude, Shopify and Slack

https://n8nworkflows.xyz/workflows/handle-e-commerce-support--orders-and-inventory-with-claude--shopify-and-slack-13594


# Handle e-commerce support, orders and inventory with Claude, Shopify and Slack

## 1. Workflow Overview

**Workflow name:** Smart E-Commerce AI Agent (Inventory + Orders + Support)  
**User-provided title:** Handle e-commerce support, orders and inventory with Claude, Shopify and Slack

This workflow exposes a webhook endpoint that receives customer support requests (chat/email/API), fetches live Shopify context (recent orders + inventory levels), classifies the customer’s intent, takes an appropriate action (refund, low-stock alert, or standard support handling), logs the interaction to PostgreSQL, notifies Slack, emails the customer, and returns an immediate webhook acknowledgement.

### 1.1 Trigger & Input Reception
Receives inbound support messages via a POST webhook and fans out to parallel Shopify API calls.

### 1.2 Fetch Live Store Data (Parallel)
Queries Shopify Orders API (by customer email) and Shopify Inventory Levels API, then merges the results into one context object.

### 1.3 Intent Classification & Decisioning
A Code node performs deterministic intent classification (keyword-based), builds an order summary, detects low stock, determines refund eligibility, and prepares a draft response plus action flags.

### 1.4 Routing & Actions
A Switch node routes to one of three tracks:
- Refund Request → attempt Shopify refund + escalation notification path (as currently wired)
- Low Stock Alert → post Slack low-stock alert
- Order/General/Pricing → proceed without special actions

### 1.5 Response Composition, Logging, Notifications
Builds an HTML email + Slack summary, sends an immediate webhook acknowledgement, emails the customer, posts a Slack summary, and records success stats in workflow static data.

---

## 2. Block-by-Block Analysis

### Block 0 — Documentation / Canvas Notes
**Overview:** Sticky notes describe the intended architecture, setup steps, and logical sections. They do not affect execution.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Large overview of purpose, “How it works”, and setup steps.
- **Connections:** None
- **Failure modes:** None

#### Node: Sticky Note1 / Sticky Note2 / Sticky Note3 / Sticky Note4
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Each note labels a major stage (Trigger, Fetch data, AI reasoning, Act/log/notify).
- **Connections:** None
- **Failure modes:** None

---

### Block 1 — Trigger & Receive Request
**Overview:** Accepts inbound customer support requests via webhook and starts two parallel data fetches from Shopify.  
**Nodes involved:** `Receive customer support request`

#### Node: Receive customer support request
- **Type / role:** Webhook trigger; entry point for the workflow.
- **Configuration choices:**
  - **Method:** POST
  - **Path:** `ecommerce-support`
  - **Response mode:** “Respond via Respond to Webhook node” (response is not returned automatically here).
- **Key fields expected in payload:** The later logic expects fields like:
  - `body.message` (or `message`)
  - `body.customer_email` (or `customer_email`)
  - `body.customer_name` (or `customer_name`)
  - `body.session_id` (optional)
- **Outputs / connections:**
  - Output → `Fetch order details from Shopify`
  - Output → `Fetch inventory levels from Shopify` (parallel)
- **Edge cases / failures:**
  - Missing `customer_email` leads to Shopify Orders query returning no matches or a malformed URL.
  - If you never reach `Send immediate webhook acknowledgement`, the webhook caller may time out.

---

### Block 2 — Fetch Live Store Data (Shopify)
**Overview:** Retrieves recent orders for the customer and current inventory levels, then merges them into a single item for analysis.  
**Nodes involved:** `Fetch order details from Shopify`, `Fetch inventory levels from Shopify`, `Merge order and inventory data`

#### Node: Fetch order details from Shopify
- **Type / role:** HTTP Request; Shopify Admin API call to list orders by email.
- **Configuration choices:**
  - **URL:** Uses the customer email from the webhook payload and URL-encodes it:  
    `.../orders.json?email={{ encodeURIComponent($json.body.customer_email) }}&status=any&limit=5`
  - **Headers:** Sets `X-Shopify-Access-Token` and `Content-Type: application/json`
- **Inputs:** Webhook payload item from `Receive customer support request`
- **Outputs / connections:** Output → `Merge order and inventory data` (Merge input 0)
- **Version notes:** Node typeVersion `4.2` (HTTP Request node behavior depends on n8n version; header/body handling differs across major versions).
- **Edge cases / failures:**
  - 401/403 if token is invalid or missing required Shopify scopes.
  - 404 if store domain or API version path is wrong.
  - The Shopify Orders API typically returns an object like `{ orders: [...] }`; downstream code expects `data.orders` to be an array—this mismatch can break the analysis unless normalized.

#### Node: Fetch inventory levels from Shopify
- **Type / role:** HTTP Request; Shopify Admin API call to list inventory levels.
- **Configuration choices:**
  - **URL:** `.../inventory_levels.json?limit=50`
  - **Headers:** `X-Shopify-Access-Token`
- **Inputs:** Webhook payload item from `Receive customer support request`
- **Outputs / connections:** Output → `Merge order and inventory data` (Merge input 1)
- **Edge cases / failures:**
  - Inventory Levels endpoint often requires `location_ids` and/or `inventory_item_ids`; calling without them may return an error or empty result depending on Shopify behavior and permissions.
  - Similar response-shape issue: Shopify returns `{ inventory_levels: [...] }`, while downstream code expects `data.inventory_levels` to be an array.

#### Node: Merge order and inventory data
- **Type / role:** Merge; combines the two branches into one item.
- **Configuration choices:**
  - **Mode:** `combine` (combines data from both inputs into a single item)
- **Inputs / connections:**
  - Input 0 ← `Fetch order details from Shopify`
  - Input 1 ← `Fetch inventory levels from Shopify`
- **Outputs / connections:** Output → `AI agent analyzes query and classifies intent`
- **Edge cases / failures:**
  - If one branch errors, merge won’t execute unless you enable error handling paths.
  - Combined JSON structure depends on n8n merge semantics; if Shopify responses are nested, you may end up with `{ orders: {orders:[...]}, inventory: {inventory_levels:[...]}}` rather than flat arrays.

---

### Block 3 — Intent Classification & Enrichment
**Overview:** Converts raw input + Shopify context into a standardized “ticket report” object: intent, confidence, response text, low-stock detection, refund eligibility, and summarized order context.  
**Nodes involved:** `AI agent analyzes query and classifies intent`

#### Node: AI agent analyzes query and classifies intent
- **Type / role:** Code node; deterministic “AI agent” logic (no external LLM call despite the name).
- **Configuration choices:**
  - Runs **once per item**
  - Extracts message/email/name/sessionId from either `body.*` or top-level fields
  - Intent classification uses keyword matching:
    - `REFUND_REQUEST`, `ORDER_INQUIRY`, `INVENTORY_CHECK`, `PRICING_INQUIRY`, else `GENERAL_SUPPORT`
  - Builds `orderSummary` from the “latest” order (`orders[0]`)
  - Low stock detection: any inventory level where `(available || 0) <= 5`
  - Refund eligibility: order must be `paid`, not fulfilled, and within 30 days
  - Constructs `aiResponse` text and flags:
    - `actionRequired`: `PROCESS_REFUND` or `SEND_STOCK_ALERT` or `null`
    - `requiresEscalation`: boolean
  - Generates `ticketId` and timestamp
- **Inputs:** Output of `Merge order and inventory data`
- **Outputs / connections:** Output → `Route by customer intent`
- **Key output fields produced (used later):**
  - `ticketId`, `timestamp`, `sessionId`
  - `customerName`, `customerEmail`, `customerMessage`
  - `intent`, `confidence`, `aiResponse`
  - `actionRequired`, `requiresEscalation`
  - `orderSummary`, `hasOrder`
  - `lowStockItems`, `lowStockCount`, `hasLowStock`
  - `isRefundEligible`, `daysSinceOrder`
- **Edge cases / failures:**
  - **Response shape mismatch**: if merged Shopify data doesn’t expose arrays at `data.orders` and `data.inventory_levels`, then `orders` and `inventoryLevels` become `[]`, producing degraded behavior (no orders found, no low stock).
  - Refund endpoint later references `order_number`; Shopify refund APIs usually require the **order id**, not order number.
  - Contains emoji characters in response strings; fine for email/Slack but be mindful of encoding and any downstream systems.

---

### Block 4 — Routing & Action Execution
**Overview:** Branches into refund processing, low stock alerting, or default support handling based on the computed fields.  
**Nodes involved:** `Route by customer intent`, `Process refund via Shopify API`, `Send low stock alert to team`, `Escalate complex cases to human agent`

#### Node: Route by customer intent
- **Type / role:** Switch; routes items to named outputs.
- **Configuration choices (rules):**
  1. **“Refund Request”** when `intent == "REFUND_REQUEST"`
  2. **“Low Stock Alert”** when `hasLowStock == true`
  3. **“Order & General Support”** when intent is any of:
     - `ORDER_INQUIRY`, `GENERAL_SUPPORT`, `PRICING_INQUIRY`
  - Rule outputs are renamed for readability.
- **Inputs:** From `AI agent analyzes query and classifies intent`
- **Outputs / connections:**
  - Refund Request → `Process refund via Shopify API`, `Store interaction in PostgreSQL`, `Escalate complex cases to human agent` (all in parallel)
  - Low Stock Alert → `Send low stock alert to team`, `Store interaction in PostgreSQL` (parallel)
  - Order & General Support → `Store interaction in PostgreSQL`
- **Edge cases / failures:**
  - Current wiring escalates *all* refund intents regardless of eligibility because escalation is directly connected to the “Refund Request” output (no gating on `requiresEscalation`).

#### Node: Process refund via Shopify API
- **Type / role:** HTTP Request; attempts to create a refund in Shopify.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `.../orders/{{ $json.orderSummary.order_number }}/refunds.json`
  - **Body params:** `refund[notify]=true`, `refund[note]=AI agent processed refund — Ticket {{ticketId}}`
  - **Headers:** Shopify access token + JSON content type
- **Inputs:** From Switch “Refund Request”
- **Outputs / connections:** Output → `Build final customer response`
- **Edge cases / failures:**
  - **Likely incorrect identifier:** Shopify refund endpoint typically uses **order_id** not **order_number**; this can cause 404 or refund creation failure.
  - Missing `orderSummary` (no orders found) will make URL invalid or reference undefined.
  - If `actionRequired` is null but intent is REFUND_REQUEST (e.g., outside window), this node still runs due to routing, potentially causing errors.

#### Node: Send low stock alert to team
- **Type / role:** HTTP Request; posts a Slack incoming-webhook alert about low stock.
- **Configuration choices:**
  - **URL:** `YOUR_SLACK_WEBHOOK_URL`
  - **Method:** POST
  - **Body:** JSON “blocks” message including `lowStockCount`, `customerEmail`, `ticketId`
- **Inputs:** From Switch “Low Stock Alert”
- **Outputs / connections:** Output → `Build final customer response`
- **Edge cases / failures:**
  - Slack webhook URL missing/invalid → 4xx
  - Slack payload size limits if expanded in future to include item lists

#### Node: Escalate complex cases to human agent
- **Type / role:** HTTP Request; posts an escalation message to Slack.
- **Configuration choices:**
  - **URL:** `YOUR_SLACK_WEBHOOK_URL`
  - **Method:** POST
  - **Body:** Slack blocks include ticket/customer/intent/message/AI response
- **Inputs:** From Switch “Refund Request” (currently unconditional for all refund intents)
- **Outputs / connections:** Output → `Build final customer response`
- **Edge cases / failures:**
  - As wired, may spam escalations for every refund request even when auto-refund succeeded and `requiresEscalation=false`.
  - Slack webhook errors are not caught; could prevent reaching later steps depending on n8n error settings.

---

### Block 5 — Persistence, Response Building, Notifications, and Webhook Reply
**Overview:** Stores the interaction, composes final email/Slack messages, acknowledges the webhook, sends customer email, posts Slack summary, and tracks cumulative stats.  
**Nodes involved:** `Store interaction in PostgreSQL`, `Build final customer response`, `Send immediate webhook acknowledgement`, `Email AI response to customer`, `Post interaction summary to Slack`, `Log success and track agent statistics`

#### Node: Store interaction in PostgreSQL
- **Type / role:** Postgres; inserts a record into `public.support_interactions`.
- **Configuration choices:**
  - **Operation:** Insert (implied by node configuration; uses auto-map)
  - **Mapping:** `autoMapInputData` (attempts to map incoming JSON keys to table columns)
  - **Credentials:** `Postgres-test`
- **Inputs:** From Switch outputs (all branches connect here)
- **Outputs / connections:** Output → `Build final customer response`
- **Edge cases / failures:**
  - Table schema mismatch: if `support_interactions` doesn’t contain columns matching incoming keys, insert will fail.
  - Missing required columns / constraints not satisfied (e.g., NOT NULL).
  - Type conversion disabled; may fail if columns expect strict types.

#### Node: Build final customer response
- **Type / role:** Code node; formats customer-facing HTML email + internal Slack summary.
- **Configuration choices:**
  - Maps intent to an emoji label (used in subject/Slack)
  - Builds styled HTML email with optional order details card
  - Creates `slackSummary`, `emailSubject`, plus flags `shouldSendEmail`, `shouldNotifySlack`
- **Inputs:** From:
  - `Store interaction in PostgreSQL` (all branches)
  - `Process refund via Shopify API` (refund branch)
  - `Send low stock alert to team` (low stock branch)
  - `Escalate complex cases to human agent` (refund branch)
- **Outputs / connections (in parallel):**
  - → `Send immediate webhook acknowledgement`
  - → `Email AI response to customer`
  - → `Post interaction summary to Slack`
- **Edge cases / failures:**
  - Assumes `report.intent` exists; if upstream data is malformed, subject/HTML generation may error.
  - The node generates `shouldSendEmail` but the workflow does not use it to conditionally skip the Email node.

#### Node: Send immediate webhook acknowledgement
- **Type / role:** Respond to Webhook; returns JSON response to the original webhook caller.
- **Configuration choices:**
  - Responds with JSON string containing `ticket_id` and `intent_detected`
  - Message: “Your request has been received…”
- **Inputs:** From `Build final customer response`
- **Outputs:** None (terminates the webhook response)
- **Edge cases / failures:**
  - If upstream steps are slow or fail before this node runs, webhook caller may time out.
  - Response body uses `JSON.stringify(...)` inside a JSON response mode; it returns a JSON *string*, not an object (often acceptable but commonly unintended).

#### Node: Email AI response to customer
- **Type / role:** Email Send (SMTP); sends the HTML response to the customer.
- **Configuration choices:**
  - **To:** `customerEmail`
  - **Subject:** `emailSubject`
  - **HTML body:** `emailHtml`
  - **From:** `user@example.com`
  - **Credentials:** `SMTP -test`
- **Inputs:** From `Build final customer response`
- **Outputs / connections:** Output → `Log success and track agent statistics`
- **Edge cases / failures:**
  - If `customerEmail` is empty/invalid, SMTP send fails.
  - No conditional guard despite `shouldSendEmail`.
  - HTML may be blocked by some mail clients; consider plaintext alternative if required.

#### Node: Post interaction summary to Slack
- **Type / role:** HTTP Request; posts a summary message to Slack via incoming webhook.
- **Configuration choices:**
  - **URL:** `YOUR_SLACK_WEBHOOK_URL`
  - **Body:** uses `emailSubject` as top text + blocks with `slackSummary`
- **Inputs:** From `Build final customer response`
- **Outputs / connections:** Output → `Log success and track agent statistics`
- **Edge cases / failures:**
  - Slack webhook invalid → prevents stats logging if node failure stops execution (depending on n8n error settings).

#### Node: Log success and track agent statistics
- **Type / role:** Code node; logs to console and accumulates global stats via workflow static data.
- **Configuration choices:**
  - Writes to `$getWorkflowStaticData('global').agentStats`
  - Tracks counts by intent, refunds processed, stock alerts, escalations, last processed timestamp
- **Inputs:** From:
  - `Email AI response to customer`
  - `Post interaction summary to Slack`
- **Outputs:** Final JSON with `success`, `ticketId`, intent, actionTaken, escalated, timestamp, `cumulativeStats`
- **Edge cases / failures:**
  - Because it has two upstream inputs, it will run once per incoming path (likely twice per ticket: once after email and once after Slack), inflating counters unless execution is constrained/merged.
  - Static data is shared across executions; in multi-instance setups, counts are per instance and not transactional.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | High-level workflow documentation | — | — | ## Smart E-Commerce AI Agent (Inventory + Orders + Support) / Creates an AI-powered sales and support agent connected to live store data from Shopify/WooCommerce. MCP ensures controlled access to inventory and order systems. Automatically handles customer queries, stock alerts, and refund logic to reduce manual workload. / ## How it works / 1. Trigger — Listens for incoming customer support requests via Webhook (chat, email, or API) / 2. Fetch context — Retrieves live order status and inventory data from Shopify in parallel / 3. AI reasoning — Claude AI analyzes the query with full store context and decides action / 4. Route intent — Classifies into: Order Inquiry, Inventory Check, Refund Request, or General Support / 5. Act — Processes refunds, sends stock alerts, or responds to customer automatically / 6. Log & notify — Saves interaction to PostgreSQL and notifies team via Slack for escalations / ## Setup steps / 1. Shopify / WooCommerce — Add your store API credentials to the HTTP Request nodes / 2. Claude AI — Set your Anthropic API key in the AI node credentials / 3. PostgreSQL — Create a `support_interactions` table to log all AI-handled tickets / 4. Slack — Add your incoming webhook URL to the Slack notification node / 5. Email — Configure SMTP credentials for customer-facing response emails / 6. Test — Send a test webhook payload, verify all branches, then activate |
| Sticky Note1 | Sticky Note | Block label: trigger stage | — | — | ## 1. Trigger & receive request / Listens for incoming customer support messages via Webhook — supports chat widgets, email-to-API bridges, and direct API calls |
| Sticky Note2 | Sticky Note | Block label: data fetch stage | — | — | ## 2. Fetch live store data / Retrieves real-time order status and inventory levels from Shopify in parallel to give the AI full context before reasoning |
| Sticky Note3 | Sticky Note | Block label: reasoning/routing stage | — | — | ## 3. AI reasoning & routing / Claude AI analyzes the query with live store context, classifies intent, and decides the appropriate action: respond, refund, restock alert, or escalate to human agent |
| Sticky Note4 | Sticky Note | Block label: act/log/notify stage | — | — | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Receive customer support request | Webhook | Entry point; receives support payload | — | Fetch order details from Shopify; Fetch inventory levels from Shopify | ## 1. Trigger & receive request / Listens for incoming customer support messages via Webhook — supports chat widgets, email-to-API bridges, and direct API calls |
| Fetch order details from Shopify | HTTP Request | Pull recent orders by customer email | Receive customer support request | Merge order and inventory data | ## 2. Fetch live store data / Retrieves real-time order status and inventory levels from Shopify in parallel to give the AI full context before reasoning |
| Fetch inventory levels from Shopify | HTTP Request | Pull inventory levels | Receive customer support request | Merge order and inventory data | ## 2. Fetch live store data / Retrieves real-time order status and inventory levels from Shopify in parallel to give the AI full context before reasoning |
| Merge order and inventory data | Merge | Combine order + inventory context | Fetch order details from Shopify; Fetch inventory levels from Shopify | AI agent analyzes query and classifies intent | ## 2. Fetch live store data / Retrieves real-time order status and inventory levels from Shopify in parallel to give the AI full context before reasoning |
| AI agent analyzes query and classifies intent | Code | Intent classification, eligibility checks, response draft | Merge order and inventory data | Route by customer intent | ## 3. AI reasoning & routing / Claude AI analyzes the query with live store context, classifies intent, and decides the appropriate action: respond, refund, restock alert, or escalate to human agent |
| Route by customer intent | Switch | Branching by intent / low-stock | AI agent analyzes query and classifies intent | Process refund via Shopify API; Send low stock alert to team; Store interaction in PostgreSQL; Escalate complex cases to human agent | ## 3. AI reasoning & routing / Claude AI analyzes the query with live store context, classifies intent, and decides the appropriate action: respond, refund, restock alert, or escalate to human agent |
| Process refund via Shopify API | HTTP Request | Attempt refund creation in Shopify | Route by customer intent | Build final customer response | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Send low stock alert to team | HTTP Request | Slack low-stock alert via incoming webhook | Route by customer intent | Build final customer response | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Store interaction in PostgreSQL | Postgres | Persist ticket/interactions | Route by customer intent | Build final customer response | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Escalate complex cases to human agent | HTTP Request | Slack escalation message | Route by customer intent | Build final customer response | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Build final customer response | Code | Build HTML email + Slack summary | Process refund via Shopify API; Send low stock alert to team; Store interaction in PostgreSQL; Escalate complex cases to human agent | Send immediate webhook acknowledgement; Email AI response to customer; Post interaction summary to Slack | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Send immediate webhook acknowledgement | Respond to Webhook | Return immediate JSON to caller | Build final customer response | — | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Email AI response to customer | Email Send (SMTP) | Send customer email reply | Build final customer response | Log success and track agent statistics | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Post interaction summary to Slack | HTTP Request | Post ticket summary to Slack | Build final customer response | Log success and track agent statistics | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |
| Log success and track agent statistics | Code | Console log + cumulative stats in static data | Email AI response to customer; Post interaction summary to Slack | — | ## 4. Act, log & notify / Executes the appropriate action — sends customer reply, triggers refund, fires low-stock alert — then logs the interaction and notifies the team via Slack and Email |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Smart E-Commerce AI Agent (Inventory + Orders + Support)**
   - Ensure workflow setting **Execution Order** is `v1` (default in many versions).

2. **Add the Webhook trigger**
   - Node: **Webhook**
   - Name: `Receive customer support request`
   - Method: **POST**
   - Path: `ecommerce-support`
   - Response mode: **Respond with Respond to Webhook node**
   - Save and note the test URL.

3. **Add Shopify Orders HTTP Request**
   - Node: **HTTP Request**
   - Name: `Fetch order details from Shopify`
   - Method: **GET**
   - URL:  
     `https://YOUR-STORE.myshopify.com/admin/api/2024-01/orders.json?email={{ encodeURIComponent($json.body.customer_email) }}&status=any&limit=5`
   - Headers:
     - `X-Shopify-Access-Token`: your Shopify Admin API token
     - `Content-Type`: `application/json`
   - Connect: `Receive customer support request` → this node

4. **Add Shopify Inventory HTTP Request**
   - Node: **HTTP Request**
   - Name: `Fetch inventory levels from Shopify`
   - Method: **GET**
   - URL:  
     `https://YOUR-STORE.myshopify.com/admin/api/2024-01/inventory_levels.json?limit=50`
   - Headers:
     - `X-Shopify-Access-Token`: your Shopify Admin API token
   - Connect: `Receive customer support request` → this node (parallel to orders node)

5. **Merge the two Shopify responses**
   - Node: **Merge**
   - Name: `Merge order and inventory data`
   - Mode: **Combine**
   - Connect:
     - `Fetch order details from Shopify` → Merge (Input 1 / index 0)
     - `Fetch inventory levels from Shopify` → Merge (Input 2 / index 1)

6. **Add the intent analysis Code node**
   - Node: **Code**
   - Name: `AI agent analyzes query and classifies intent`
   - Mode: **Run once for each item**
   - Paste the classification/enrichment script (the one shown in your workflow).
   - Connect: `Merge order and inventory data` → this node

7. **Add the Switch routing node**
   - Node: **Switch**
   - Name: `Route by customer intent`
   - Add rules:
     - Output “Refund Request”: `{{$json.intent}} equals REFUND_REQUEST`
     - Output “Low Stock Alert”: `{{$json.hasLowStock}} is true`
     - Output “Order & General Support”: intent equals `ORDER_INQUIRY` OR `GENERAL_SUPPORT` OR `PRICING_INQUIRY`
   - Connect: `AI agent analyzes...` → `Route by customer intent`

8. **Refund action (Shopify)**
   - Node: **HTTP Request**
   - Name: `Process refund via Shopify API`
   - Method: **POST**
   - URL:  
     `https://YOUR-STORE.myshopify.com/admin/api/2024-01/orders/{{ $json.orderSummary.order_number }}/refunds.json`
   - Send body: enabled
   - Body parameters:
     - `refund[notify]` = `true`
     - `refund[note]` = `AI agent processed refund — Ticket {{ $json.ticketId }}`
   - Headers:
     - `X-Shopify-Access-Token` = your token
     - `Content-Type` = `application/json`
   - Connect: Switch output “Refund Request” → this node

9. **Low stock Slack alert**
   - Node: **HTTP Request**
   - Name: `Send low stock alert to team`
   - Method: **POST**
   - URL: your **Slack incoming webhook URL**
   - Body: **JSON** (specify body: json)
   - Use the JSON block template from the workflow (includes `lowStockCount`, `customerEmail`, `ticketId`).
   - Connect: Switch output “Low Stock Alert” → this node

10. **Escalation Slack message**
   - Node: **HTTP Request**
   - Name: `Escalate complex cases to human agent`
   - Method: **POST**
   - URL: your Slack incoming webhook URL
   - Body: JSON blocks (use the workflow’s template; references ticket/customer/intent/message/aiResponse)
   - Connect: Switch output “Refund Request” → this node (matches the current design)

11. **PostgreSQL logging**
   - Node: **Postgres**
   - Name: `Store interaction in PostgreSQL`
   - Credentials: create/select a Postgres credential (host/db/user/password/SSL as required)
   - Schema: `public`
   - Table: `support_interactions`
   - Columns mapping: **Auto-map input data**
   - Connect:
     - Switch “Refund Request” → Postgres
     - Switch “Low Stock Alert” → Postgres
     - Switch “Order & General Support” → Postgres

12. **Build final response Code node**
   - Node: **Code**
   - Name: `Build final customer response`
   - Mode: run once per item
   - Paste the HTML+Slack composition script from the workflow.
   - Connect into it from all action/logging nodes:
     - `Process refund via Shopify API` → Build node
     - `Send low stock alert to team` → Build node
     - `Escalate complex cases to human agent` → Build node
     - `Store interaction in PostgreSQL` → Build node

13. **Respond to webhook**
   - Node: **Respond to Webhook**
   - Name: `Send immediate webhook acknowledgement`
   - Respond with: **JSON**
   - Body expression: use the workflow’s expression returning `ticket_id` and `intent_detected`.
   - Connect: `Build final customer response` → this node

14. **Send the customer email**
   - Node: **Email Send**
   - Name: `Email AI response to customer`
   - Credentials: set up **SMTP** credential
   - From: `user@example.com` (replace with your support address)
   - To: `{{$json.customerEmail}}`
   - Subject: `{{$json.emailSubject}}`
   - HTML: `{{$json.emailHtml}}`
   - Connect: `Build final customer response` → this node

15. **Post summary to Slack**
   - Node: **HTTP Request**
   - Name: `Post interaction summary to Slack`
   - Method: POST
   - URL: your Slack incoming webhook URL
   - Body JSON: use the workflow’s template referencing `emailSubject` and `slackSummary`
   - Connect: `Build final customer response` → this node

16. **Track success stats**
   - Node: **Code**
   - Name: `Log success and track agent statistics`
   - Paste the stats script (uses `$getWorkflowStaticData('global')`)
   - Connect:
     - `Email AI response to customer` → Stats node
     - `Post interaction summary to Slack` → Stats node

17. **(Optional) Add sticky notes**
   - Add sticky notes with the same content if you want the canvas documentation blocks.

18. **Test**
   - Call the webhook with a JSON body including at least:
     - `customer_email`, `customer_name`, `message`
   - Verify:
     - webhook receives acknowledgement
     - Slack receives alert/summary (as applicable)
     - email sends
     - Postgres row inserted

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The notes describe “Claude AI” and “Anthropic API key”, but the implemented workflow uses deterministic Code nodes and does not call Anthropic/Claude. | Workflow/canvas documentation vs actual node implementation mismatch |
| Shopify API placeholders must be replaced: `YOUR-STORE`, `YOUR_SHOPIFY_ACCESS_TOKEN`, and Slack `YOUR_SLACK_WEBHOOK_URL`. | Required setup before activation |
| Refund processing likely requires Shopify **order ID** rather than `order_number`. | Potential functional bug in `Process refund via Shopify API` |
| Stats node is fed by both Email and Slack branches, which can double-count tickets. | Workflow design consideration |
| Postgres node uses auto-mapping; you must design `support_interactions` table columns to match incoming JSON keys or switch to explicit mapping. | Database schema requirement |