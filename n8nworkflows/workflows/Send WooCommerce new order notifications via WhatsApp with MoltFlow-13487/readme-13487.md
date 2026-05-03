Send WooCommerce new order notifications via WhatsApp with MoltFlow

https://n8nworkflows.xyz/workflows/send-woocommerce-new-order-notifications-via-whatsapp-with-moltflow-13487


# Send WooCommerce new order notifications via WhatsApp with MoltFlow

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Receive a WooCommerce “order created” webhook, format the order into one or two WhatsApp messages (store owner + optional customer confirmation), send them via **MoltFlow (waiflow)** API, then log a compact “notified” result.

**Target use cases:**
- Instant operational alerts to a store owner/operator when a new WooCommerce order arrives.
- Optional auto-confirmation to the customer via WhatsApp (if a phone number exists and passes a basic length check).

**Logical blocks**
1. **1.1 Webhook Intake (WooCommerce → n8n)**
   - Receives the WooCommerce webhook payload on a POST endpoint.
2. **1.2 Order Parsing & Message Building**
   - Extracts order fields, normalizes phone, builds owner message, conditionally builds customer message.
   - Produces 1–2 output items for downstream sending.
3. **1.3 WhatsApp Dispatch (MoltFlow API)**
   - Sends each prepared message item to MoltFlow’s “send message” endpoint using header-based API key auth.
4. **1.4 Execution Logging**
   - Aggregates results and returns a minimal summary (order_id, count of messages sent).

---

## 2. Block-by-Block Analysis

### 2.1 Webhook Intake (WooCommerce → n8n)

**Overview:** Exposes an n8n webhook URL to be configured inside WooCommerce. When WooCommerce fires the “Order created” webhook, this node triggers the workflow and passes the order payload forward.

**Nodes involved:**
- **WooCommerce Webhook**

**Node details**

#### Node: WooCommerce Webhook
- **Type / role:** `Webhook` — workflow trigger and HTTP endpoint.
- **Key configuration choices:**
  - **HTTP Method:** `POST`
  - **Path:** `woo-order` (the endpoint will be based on your n8n base URL plus this path)
- **Input / output:**
  - **Input:** external HTTP POST from WooCommerce.
  - **Output:** one item containing webhook data (commonly includes `body` if n8n parses JSON body).
- **Connections:**
  - Outputs to **Format Order**.
- **Version-specific requirements:**
  - `typeVersion: 2` (standard modern Webhook node behavior; payload structure commonly includes `json.body` when content-type is JSON).
- **Edge cases / failures:**
  - WooCommerce not sending JSON / wrong content-type can change where the order payload appears.
  - Missing authentication on webhook endpoint (this workflow uses no webhook auth options); endpoint could be abused if publicly accessible.
  - Large payloads or WooCommerce retries may trigger duplicate notifications if WooCommerce retries delivery.

---

### 2.2 Order Parsing & Message Building

**Overview:** Interprets the WooCommerce order payload, extracts customer/order/item fields, builds WhatsApp message text, and emits one message for the owner and optionally one for the customer.

**Nodes involved:**
- **Format Order**

**Node details**

#### Node: Format Order
- **Type / role:** `Code` — transforms incoming webhook JSON into message payload(s) for the API.
- **Configuration choices (interpreted):**
  - **Mode:** `runOnceForAllItems` (processes all incoming items at once; uses the first item only).
  - Hard-coded placeholders:
    - `SESSION_ID = 'YOUR_SESSION_ID'`
    - `OWNER_PHONE = 'YOUR_PHONE'`
  - Payload detection:
    - `const order = $input.first().json.body || $input.first().json;`
      - Supports either `json.body` (typical webhook body) or direct `json`.
- **Key extracted fields:**
  - `orderId`: `order.id || order.number || 'N/A'`
  - `status`: default `'new'`
  - `total`: default `'0.00'`
  - `currency`: default `'USD'`
  - `billing` object for name/email/phone
  - `customerPhone` normalization: removes non-digits via `.replace(/[^0-9]/g, '')`
  - `items` string: joins `line_items` into `"Name xQty, ..."`
- **Output construction:**
  - Always returns **one item** for the owner notification:
    - `session_id`, `chat_id = OWNER_PHONE + '@c.us'`, `message`, `type`, `order_id`
  - Conditionally returns **a second item** for customer confirmation if:
    - `customerPhone` exists and `customerPhone.length >= 7`
- **Connections:**
  - Output goes to **Send WhatsApp** (each output item becomes one send request).
- **Version-specific requirements:**
  - `typeVersion: 2` (Code node using current `$input` helper API).
- **Edge cases / failures:**
  - If WooCommerce payload differs (e.g., `line_items` missing), `items` becomes an empty string; message still sends but less informative.
  - Phone formatting: removing non-digits may remove the `+` country code indicator; you must ensure numbers are in the format MoltFlow expects (often E.164 digits without plus is acceptable depending on provider).
  - `OWNER_PHONE` must be a WhatsApp-valid number; if not, MoltFlow send will fail.
  - If `billing` is absent (guest checkout variations), name/phone may be empty; customer confirmation will be skipped if phone is too short.
  - Potential duplicate messages if webhook is sent multiple times (no deduping by order_id).

---

### 2.3 WhatsApp Dispatch (MoltFlow API)

**Overview:** Sends each message item created in the previous block to MoltFlow’s API endpoint using header authentication.

**Nodes involved:**
- **Send WhatsApp**

**Node details**

#### Node: Send WhatsApp
- **Type / role:** `HTTP Request` — calls MoltFlow (waiflow) API to send WhatsApp messages.
- **Configuration choices (interpreted):**
  - **Method:** `POST`
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Body type:** JSON
  - **Authentication:** `Header Auth` via a generic credential (`httpHeaderAuth`)
    - Header key is expected to be: `X-API-Key` (per sticky note instructions)
- **Key expressions / variables:**
  - JSON body is built from current item:
    - `session_id: $json.session_id`
    - `chat_id: $json.chat_id`
    - `message: $json.message`
  - Implemented as stringified JSON in the node:
    - `={{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
- **Input / output:**
  - **Input:** 1–2 items from **Format Order**.
  - **Output:** API response per item (success/failure details depend on MoltFlow).
- **Connections:**
  - Output goes to **Log**.
- **Version-specific requirements:**
  - `typeVersion: 4.2` (HTTP Request node; UI fields may differ slightly across versions).
- **Edge cases / failures:**
  - Missing/incorrect API key → 401/403 responses.
  - Invalid `session_id` or disconnected WhatsApp session → API error.
  - Invalid `chat_id` formatting (must be `<digits>@c.us`) → send failure.
  - Rate limiting / transient failures; no retry/backoff logic is implemented in this workflow.
  - If the body is already treated as JSON by the node, double-stringifying could be problematic in some setups; here it is explicitly configured as JSON body but provides a string—verify n8n behavior in your version (if issues occur, set body fields directly without `JSON.stringify`).

---

### 2.4 Execution Logging

**Overview:** Aggregates all send results and emits a single summary item indicating notifications were sent.

**Nodes involved:**
- **Log**

**Node details**

#### Node: Log
- **Type / role:** `Code` — summarizes execution outcome.
- **Configuration choices (interpreted):**
  - **Mode:** `runOnceForAllItems`
  - Reads all incoming send results:
    - `const items = $input.all();`
  - Retrieves original order_id from the **Format Order** node:
    - `const orderId = $('Format Order').first().json.order_id;`
  - Returns one summary item:
    - `{ status: 'notified', order_id: orderId, messages_sent: items.length }`
- **Input / output:**
  - **Input:** one item per API call result.
  - **Output:** single item summary.
- **Connections:**
  - No downstream nodes.
- **Version-specific requirements:**
  - `typeVersion: 2`
- **Edge cases / failures:**
  - If **Format Order** didn’t run (or was renamed), the node reference `$('Format Order')` will fail.
  - If the HTTP request node fails hard and stops execution (depending on “Continue On Fail” settings—none specified), this node might not run.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / on-canvas guidance | — | — | ## WooCommerce → WhatsApp Order Alert \nGet instant WhatsApp notifications when a new order is placed on your WooCommerce store, powered by [MoltFlow](https://molt.waiflow.app).\n\n**How it works:**\n1. WooCommerce fires a webhook on new order\n2. Order details are extracted (customer, items, total)\n3. A WhatsApp notification is sent to the store owner\n4. Optionally sends a confirmation to the customer too |
| Sticky Note1 | Sticky Note | Documentation / setup steps | — | — | ## Setup (5 min)\n1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp\n2. Activate this workflow — copy the webhook URL\n3. In WooCommerce → Settings → Advanced → Webhooks, add a webhook for **Order created** pointing to this URL\n4. Set `YOUR_SESSION_ID` and `OWNER_PHONE` in the Format Order node\n5. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| WooCommerce Webhook | Webhook | Entry point: receives WooCommerce order-created webhook | — | Format Order | ## WooCommerce → WhatsApp Order Alert \nGet instant WhatsApp notifications when a new order is placed on your WooCommerce store, powered by [MoltFlow](https://molt.waiflow.app).\n\n**How it works:**\n1. WooCommerce fires a webhook on new order\n2. Order details are extracted (customer, items, total)\n3. A WhatsApp notification is sent to the store owner\n4. Optionally sends a confirmation to the customer too |
| Format Order | Code | Parse order payload; build owner + optional customer message items | WooCommerce Webhook | Send WhatsApp | ## Setup (5 min)\n1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp\n2. Activate this workflow — copy the webhook URL\n3. In WooCommerce → Settings → Advanced → Webhooks, add a webhook for **Order created** pointing to this URL\n4. Set `YOUR_SESSION_ID` and `OWNER_PHONE` in the Format Order node\n5. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| Send WhatsApp | HTTP Request | Send WhatsApp message via MoltFlow API | Format Order | Log | ## Setup (5 min)\n1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp\n2. Activate this workflow — copy the webhook URL\n3. In WooCommerce → Settings → Advanced → Webhooks, add a webhook for **Order created** pointing to this URL\n4. Set `YOUR_SESSION_ID` and `OWNER_PHONE` in the Format Order node\n5. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| Log | Code | Summarize execution (order_id, message count) | Send WhatsApp | — | ## WooCommerce → WhatsApp Order Alert \nGet instant WhatsApp notifications when a new order is placed on your WooCommerce store, powered by [MoltFlow](https://molt.waiflow.app).\n\n**How it works:**\n1. WooCommerce fires a webhook on new order\n2. Order details are extracted (customer, items, total)\n3. A WhatsApp notification is sent to the store owner\n4. Optionally sends a confirmation to the customer too |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“WooCommerce New Order Notification via WhatsApp with MoltFlow”**
   - (Optional) Add tags: `whatsapp`, `woocommerce`, `e-commerce`, `moltflow`, `wordpress`, `orders`

2. **Add node: Webhook**
   - Node type: **Webhook**
   - Name: **WooCommerce Webhook**
   - Set:
     - **HTTP Method:** `POST`
     - **Path:** `woo-order`
   - Activate/save to obtain a **Production webhook URL** (and/or Test URL).

3. **Configure WooCommerce webhook**
   - In WordPress admin: **WooCommerce → Settings → Advanced → Webhooks**
   - Add webhook:
     - **Event:** *Order created*
     - **Delivery URL:** paste the n8n webhook URL from step 2
     - Ensure webhook is enabled and uses JSON (WooCommerce default is typically fine).

4. **Add node: Code**
   - Node type: **Code**
   - Name: **Format Order**
   - Mode: **Run Once for All Items**
   - Paste/implement logic equivalent to:
     - Read order payload from `first().json.body` fallback to `first().json`
     - Extract `id/number`, `status`, `total`, `currency`, `billing`, and `line_items`
     - Normalize phone digits
     - Output:
       - Owner item always: `{ session_id, chat_id: OWNER_PHONE + '@c.us', message, type, order_id }`
       - Customer item optionally if phone length ≥ 7
   - Replace placeholders:
     - `YOUR_SESSION_ID` → your MoltFlow WhatsApp session identifier
     - `YOUR_PHONE` (owner) → the owner WhatsApp number digits (country code + number)

5. **Add MoltFlow API credential (Header Auth)**
   - Create credential: **HTTP Header Auth**
   - Name it: **MoltFlow API Key**
   - Set header:
     - **Name:** `X-API-Key`
     - **Value:** your MoltFlow API key from MoltFlow dashboard

6. **Add node: HTTP Request**
   - Node type: **HTTP Request**
   - Name: **Send WhatsApp**
   - Configure:
     - **Method:** `POST`
     - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
     - **Authentication:** `Generic Credential Type` → select **HTTP Header Auth** → **MoltFlow API Key**
     - **Send Body:** enabled
     - **Body Content Type:** JSON
     - **Body:** map from incoming item:
       - `session_id` = current item `session_id`
       - `chat_id` = current item `chat_id`
       - `message` = current item `message`
     - If your n8n version supports object-based JSON body directly, prefer direct fields rather than a `JSON.stringify(...)` expression.

7. **Add node: Code (logger)**
   - Node type: **Code**
   - Name: **Log**
   - Mode: **Run Once for All Items**
   - Logic:
     - Count inputs from Send WhatsApp
     - Read `order_id` from **Format Order** node output
     - Return one item: `status: notified`, `order_id`, `messages_sent`

8. **Connect nodes in order**
   - **WooCommerce Webhook → Format Order → Send WhatsApp → Log**

9. **Activate workflow**
   - Ensure WooCommerce points to the **Production** URL (not the test URL).
   - Place a test order; confirm one owner message (and optional customer message) arrives.

**Sub-workflows:** None (no “Execute Workflow” nodes).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow service used to send WhatsApp messages | https://molt.waiflow.app |
| MoltFlow API endpoint used by this workflow: `POST /api/v2/messages/send` on `apiv2.waiflow.app` | Referenced in “Send WhatsApp” node configuration |
| WooCommerce webhook must be configured for **Order created** and set to the n8n webhook URL | WooCommerce → Settings → Advanced → Webhooks |
| MoltFlow API authentication expects header `X-API-Key` | Set in n8n credential “HTTP Header Auth” |
| Placeholders to replace in code: `YOUR_SESSION_ID`, `YOUR_PHONE` | In the “Format Order” node |