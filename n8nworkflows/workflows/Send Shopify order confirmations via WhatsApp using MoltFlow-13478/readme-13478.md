Send Shopify order confirmations via WhatsApp using MoltFlow

https://n8nworkflows.xyz/workflows/send-shopify-order-confirmations-via-whatsapp-using-moltflow-13478


# Send Shopify order confirmations via WhatsApp using MoltFlow

## 1. Workflow Overview

**Purpose:** Automatically send a personalized **WhatsApp order confirmation** to customers when a **Shopify order is created**, using the **MoltFlow (Waiflow) API**. The workflow receives a Shopify webhook, formats a human-readable confirmation message, sends it via MoltFlow, and logs whether it was sent or skipped.

**Target use cases:**
- Shopify stores that want immediate WhatsApp confirmations after checkout
- Teams that want a low-friction WhatsApp integration without building a custom app
- Basic customer comms automation with minimal dependencies

### 1.1 Input Reception (Shopify webhook)
Receives an “order creation” event from Shopify via an HTTP POST webhook endpoint in n8n.

### 1.2 Data Extraction & Message Formatting
Extracts customer name, phone number, order metadata, builds `chat_id`, and constructs a WhatsApp-ready message.

### 1.3 Validation / Routing
Checks whether a valid phone number was found; routes to “send” or “skip”.

### 1.4 WhatsApp Send (MoltFlow API)
Sends a POST request to MoltFlow’s message endpoint using an API key in a header-auth credential.

### 1.5 Outcome Logging
Returns a compact JSON payload describing success or skip reason (useful for execution logs or downstream steps).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Shopify webhook)

**Overview:** Exposes a webhook endpoint that Shopify calls when an order is created. This is the workflow’s trigger/entry point.

**Nodes involved:**
- **Shopify Order Webhook**

#### Node: Shopify Order Webhook
- **Type / role:** `Webhook` (trigger). Starts the workflow on incoming HTTP requests.
- **Configuration (interpreted):**
  - **HTTP Method:** `POST`
  - **Path:** `shopify-order`
  - **Webhook ID:** `shopify-order` (internal identifier)
- **Input / output:**
  - **Input:** External HTTP request from Shopify (order payload)
  - **Output:** One item containing request data; downstream nodes expect the order JSON to exist either at `json.body` or directly at `json`.
- **Connections:**
  - Output → **Format Order Message**
- **Version-specific notes:** Node `typeVersion: 2` (Webhook v2 in n8n).
- **Edge cases / failures:**
  - Shopify webhook not firing (wrong topic/event, wrong URL, not enabled)
  - Shopify HMAC verification is **not implemented** here (anyone could POST unless protected by network controls or a secret path)
  - Large payloads rarely matter for orders, but can still cause timeouts if n8n instance/network is constrained
  - If Shopify sends the payload in an unexpected shape, formatting may fail downstream

---

### Block 2 — Data Extraction & Message Formatting

**Overview:** Normalizes the incoming order payload, derives customer identity and phone, formats a WhatsApp message, and prepares MoltFlow API fields.

**Nodes involved:**
- **Format Order Message**

#### Node: Format Order Message
- **Type / role:** `Code` node. Transforms Shopify order JSON into a payload for MoltFlow.
- **Configuration choices (interpreted):**
  - **Mode:** `runOnceForAllItems` (but it effectively processes only the first item via `$input.first()`)
  - Defines a constant `SESSION_ID = 'YOUR_SESSION_ID'` (must be replaced)
  - Reads order from:
    - `const order = $input.first().json.body || $input.first().json;`
- **Key variables / logic:**
  - **customerName**
    - If `order.customer` exists: `first_name + last_name` (trimmed)
    - Else: `"Customer"`
  - **phone** extraction and normalization:
    - Prefers `order.customer.phone`
    - Fallback: `order.billing_address.phone`
    - Normalization: `replace(/[^0-9]/g, '')` (keeps digits only)
  - If no phone after extraction:
    - Returns `{ error: 'No phone number found', order_id: ... }`
    - Note: it does **not** set `phone` in this returned object
  - **orderNumber:** `order.order_number || order.name || order.id`
  - **total:** `order.total_price || '0.00'`
  - **currency:** `order.currency || 'USD'`
  - **items:** from `(order.line_items || []).map(title + ' x' + quantity).join(', ')`
  - **message:** human-formatted multi-line string including items and totals
  - Output fields for MoltFlow:
    - `session_id`
    - `chat_id: phone + '@c.us'`
    - `message`
    - plus `customer_name`, `order_number`, `phone` for logging/routing
- **Input / output:**
  - **Input:** Webhook payload item
  - **Output:** One item containing either:
    - a full send-ready payload including `phone`, or
    - an error-only object when phone is missing
- **Connections:**
  - Output → **Has Phone?**
- **Version-specific notes:** Code node `typeVersion: 2` (uses modern n8n code environment APIs like `$input.first()`).
- **Edge cases / failures:**
  - **Critical routing quirk:** when no phone is found, the node returns an object without `phone`, so the next IF (`Has Phone?`) will treat `{{$json.phone}}` as empty/undefined and route to **No Phone - Skip** (that’s OK), but:
    - **No Phone - Skip** later references `$('Format Order Message').first().json.order_number`, which will be undefined in that error branch (because that branch returns `order_id`, not `order_number`).
  - If `order.line_items` is missing/empty, `items` becomes an empty string; message still sends but “Items:” line will be blank.
  - Phone normalization strips `+` and spaces; MoltFlow expects WhatsApp ID format, so digits-only is typically fine, but requires **country code** to be present in the original phone.

---

### Block 3 — Validation / Routing

**Overview:** Ensures the workflow only attempts to send WhatsApp messages if a phone number exists.

**Nodes involved:**
- **Has Phone?**

#### Node: Has Phone?
- **Type / role:** `IF` node. Branches based on presence of `phone`.
- **Configuration choices (interpreted):**
  - Condition: `{{$json.phone}}` **not equals** `""`
  - Strict type validation enabled (default in newer IF versions)
- **Input / output:**
  - **Input:** Output from “Format Order Message”
  - **Output (true branch):** goes to send message
  - **Output (false branch):** goes to skip
- **Connections:**
  - True → **Send WhatsApp Confirmation**
  - False → **No Phone - Skip**
- **Version-specific notes:** IF node `typeVersion: 2`.
- **Edge cases / failures:**
  - If `phone` is `undefined` (as in the error output from the Code node), the expression resolves empty and will go false branch (desired).
  - If phone is present but invalid (too short, missing country code), it still passes and send may fail downstream.

---

### Block 4 — WhatsApp Send (MoltFlow API)

**Overview:** Sends the prepared message to MoltFlow’s API endpoint using header-based API key authentication.

**Nodes involved:**
- **Send WhatsApp Confirmation**

#### Node: Send WhatsApp Confirmation
- **Type / role:** `HTTP Request` node. Calls MoltFlow API to send a WhatsApp message.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Authentication:** Generic Credential → `httpHeaderAuth`
    - Credential name: **“MoltFlow API Key”**
    - Header key expected per sticky note: `X-API-Key`
  - **Body type:** JSON (`specifyBody: json`, `sendBody: true`)
  - **Body expression:**
    - `={{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
- **Input / output:**
  - **Input:** `session_id`, `chat_id`, `message` from Format node
  - **Output:** MoltFlow API response (commonly includes a message id; exact schema depends on API)
- **Connections:**
  - Output → **Log Success**
- **Version-specific notes:** HTTP Request node `typeVersion: 4.2` (newer request node behavior/options).
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid or header name misconfigured
  - 400 if `session_id` is wrong or `chat_id` format invalid
  - 429 rate limiting if sending too frequently
  - Network/DNS/TLS failures
  - **Body formatting risk:** When `specifyBody` is JSON, n8n usually expects an object; here it is providing `JSON.stringify(...)` (a string). Some configurations still work, but it can also lead to “invalid JSON” / double-encoding depending on node behavior. Safer approach is to pass the object directly (see notes in reproduction section).

---

### Block 5 — Outcome Logging

**Overview:** Produces a simple status object for execution history: either “sent” with a message id, or “skipped” with a reason.

**Nodes involved:**
- **Log Success**
- **No Phone - Skip**

#### Node: Log Success
- **Type / role:** `Code` node. Normalizes “send” result to a simple log record.
- **Configuration choices (interpreted):**
  - Mode: `runOnceForAllItems`
  - Reads `const input = $input.first().json;`
  - Returns:
    - `status: 'sent'`
    - `customer`: `input.customer_name` OR fallback to `$('Format Order Message').first().json.customer_name`
    - `order`: from `$('Format Order Message').first().json.order_number`
    - `message_id`: `input.id || 'N/A'`
- **Input / output:**
  - **Input:** MoltFlow API response
  - **Output:** Log JSON item
- **Connections:** No outgoing connections.
- **Edge cases / failures:**
  - If MoltFlow response does not contain `id`, it logs `N/A`
  - If the API returns an error payload but still as JSON, this node may incorrectly mark as sent (because it doesn’t check HTTP status). Consider enabling “Fail on non-2xx” in HTTP Request or validating response fields.

#### Node: No Phone - Skip
- **Type / role:** `Code` node. Emits a skip status when no phone exists.
- **Configuration choices (interpreted):**
  - Returns:
    - `status: 'skipped'`
    - `reason: 'No phone number on order'`
    - `order: $('Format Order Message').first().json.order_number`
- **Input / output:**
  - **Input:** Item from IF false branch
  - **Output:** Log JSON item
- **Connections:** No outgoing connections.
- **Edge cases / failures:**
  - As noted earlier: if “Format Order Message” produced the “error object” (with `order_id` but without `order_number`), then `order` here will be `undefined`. A more robust implementation would use `order_id` fallback.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / canvas annotation |  |  | ## Shopify → WhatsApp Order Confirmation; Send instant WhatsApp order confirmations when a customer places an order on your Shopify store, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Shopify sends an order webhook to n8n; 2. Customer details and order info are extracted; 3. A personalized WhatsApp confirmation is sent; 4. Result is logged for tracking |
| Sticky Note1 | Sticky Note | Documentation / setup annotation |  |  | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Shopify Admin → Settings → Notifications → Webhooks, add the n8n URL for **Order creation**; 4. Set `YOUR_SESSION_ID` in the Format Order node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Note:** Customer phone must include country code |
| Shopify Order Webhook | Webhook | Receive Shopify “order created” event |  | Format Order Message | ## Shopify → WhatsApp Order Confirmation; Send instant WhatsApp order confirmations when a customer places an order on your Shopify store, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Shopify sends an order webhook to n8n; 2. Customer details and order info are extracted; 3. A personalized WhatsApp confirmation is sent; 4. Result is logged for tracking |
| Format Order Message | Code | Extract fields, normalize phone, compose WhatsApp message, build MoltFlow payload | Shopify Order Webhook | Has Phone? | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Shopify Admin → Settings → Notifications → Webhooks, add the n8n URL for **Order creation**; 4. Set `YOUR_SESSION_ID` in the Format Order node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Note:** Customer phone must include country code |
| Has Phone? | IF | Route based on presence of phone | Format Order Message | Send WhatsApp Confirmation (true); No Phone - Skip (false) |  |
| Send WhatsApp Confirmation | HTTP Request | Send WhatsApp message through MoltFlow API | Has Phone? (true) | Log Success | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Shopify Admin → Settings → Notifications → Webhooks, add the n8n URL for **Order creation**; 4. Set `YOUR_SESSION_ID` in the Format Order node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Note:** Customer phone must include country code |
| Log Success | Code | Produce success log record | Send WhatsApp Confirmation |  |  |
| No Phone - Skip | Code | Produce skip log record when phone missing | Has Phone? (false) |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Shopify Order Confirmation via WhatsApp with MoltFlow”**
   - (Optional) Add tags: `whatsapp`, `shopify`, `e-commerce`, `moltflow`, `order confirmation`

2. **Add node: Webhook**
   - Node name: **Shopify Order Webhook**
   - Type: **Webhook**
   - Settings:
     - **HTTP Method:** POST
     - **Path:** `shopify-order`
   - Save workflow to generate a test/production webhook URL.

3. **Configure Shopify webhook**
   - In **Shopify Admin → Settings → Notifications → Webhooks**
   - Create webhook for **Order creation**
   - Paste the n8n webhook URL (use Production URL for live usage)
   - Ensure Shopify can reach your n8n instance (public URL / firewall rules).

4. **Add node: Code**
   - Node name: **Format Order Message**
   - Type: **Code**
   - Mode: **Run Once for All Items**
   - Paste the logic that:
     - Sets `SESSION_ID = 'YOUR_SESSION_ID'` (replace with your real MoltFlow session id)
     - Extracts order payload from `json.body` or `json`
     - Extracts/normalizes phone and builds:
       - `session_id`, `chat_id = phone + '@c.us'`, `message`, `customer_name`, `order_number`, `phone`
   - Connect: **Shopify Order Webhook → Format Order Message**

5. **Add node: IF**
   - Node name: **Has Phone?**
   - Condition:
     - Left value (expression): `{{$json.phone}}`
     - Operation: **not equals**
     - Right value: empty string
   - Connect: **Format Order Message → Has Phone?**

6. **Add credential: HTTP Header Auth (for MoltFlow)**
   - In n8n Credentials, create **HTTP Header Auth**
   - Name: **MoltFlow API Key**
   - Header Name: `X-API-Key`
   - Header Value: *(your MoltFlow API key from MoltFlow dashboard)*

7. **Add node: HTTP Request**
   - Node name: **Send WhatsApp Confirmation**
   - Type: **HTTP Request**
   - Settings:
     - Method: POST
     - URL: `https://apiv2.waiflow.app/api/v2/messages/send`
     - Authentication: **Generic Credential Type → HTTP Header Auth**
     - Select credential: **MoltFlow API Key**
     - Body content: JSON with fields:
       - `session_id` (from Format node)
       - `chat_id`
       - `message`
   - Connect: **Has Phone? (true output) → Send WhatsApp Confirmation**
   - Implementation note (recommended for reliability): set the JSON body as an **object** (not a string). For example:
     - `session_id: {{$json.session_id}}`
     - `chat_id: {{$json.chat_id}}`
     - `message: {{$json.message}}`

8. **Add node: Code (success logger)**
   - Node name: **Log Success**
   - Type: **Code**
   - Mode: Run Once for All Items
   - Logic:
     - Output `{ status: 'sent', customer: ..., order: ..., message_id: ... }`
     - Use MoltFlow response `id` if available; otherwise `N/A`
   - Connect: **Send WhatsApp Confirmation → Log Success**

9. **Add node: Code (skip logger)**
   - Node name: **No Phone - Skip**
   - Type: **Code**
   - Mode: Run Once for All Items
   - Logic:
     - Output `{ status: 'skipped', reason: 'No phone number on order', order: ... }`
   - Connect: **Has Phone? (false output) → No Phone - Skip**

10. **Add sticky notes (optional but recommended for maintainability)**
   - Add a sticky note describing the overall flow and include the MoltFlow link: https://molt.waiflow.app
   - Add a sticky note listing setup steps, including:
     - Replace `YOUR_SESSION_ID`
     - Configure header auth `X-API-Key`
     - Ensure phone includes country code

11. **Test**
   - Use the Webhook node “Test” mode and trigger an order creation test from Shopify (or POST a sample payload).
   - Verify:
     - True branch sends successfully
     - False branch logs skipped when phone is missing

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow account setup and WhatsApp connection required | https://molt.waiflow.app |
| Shopify webhook setup: add n8n webhook URL for **Order creation** | Shopify Admin → Settings → Notifications → Webhooks |
| Set `YOUR_SESSION_ID` in the “Format Order Message” code | Required for MoltFlow session routing |
| Configure MoltFlow API Key using Header Auth `X-API-Key` | n8n Credentials → HTTP Header Auth |
| Customer phone must include country code | Needed to form valid WhatsApp `chat_id` (digits + `@c.us`) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.