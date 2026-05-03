Notify customers on WhatsApp when Stripe payment succeeds with MoltFlow

https://n8nworkflows.xyz/workflows/notify-customers-on-whatsapp-when-stripe-payment-succeeds-with-moltflow-13483


# Notify customers on WhatsApp when Stripe payment succeeds with MoltFlow

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow listens for a successful Stripe Checkout payment event and immediately sends a WhatsApp “receipt/confirmation” message to the customer using **MoltFlow (waiflow.app)**. It skips sending if the Stripe event does not contain a usable phone number.

**Target use cases:**
- Send instant purchase confirmations after Stripe Checkout payments
- Reduce support requests by proactively confirming payment and providing a reference (payment ID suffix)
- Simple post-payment messaging automation via WhatsApp

**Logical blocks (by node dependency):**
1.1 **Webhook Reception (Stripe → n8n)**  
Receives the Stripe webhook request.

1.2 **Receipt Formatting & Data Extraction**  
Parses the Stripe payload, extracts customer info and payment amount, and builds the WhatsApp message. Flags executions to skip if the phone is missing.

1.3 **Routing: Phone Available?**  
Checks the `skip` flag to determine whether to send or stop.

1.4 **WhatsApp Delivery via MoltFlow + Logging**  
Sends the message through MoltFlow’s HTTP API and outputs a simplified “sent” log object.

---

## 2. Block-by-Block Analysis

### 1.1 Webhook Reception (Stripe → n8n)

**Overview:**  
Accepts Stripe webhook calls (HTTP POST) on a fixed path. It is the entry point of the workflow.

**Nodes involved:**
- **Stripe Webhook**

**Node details**

#### Node: Stripe Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow trigger via HTTP endpoint.
- **Configuration (interpreted):**
  - **HTTP Method:** POST  
  - **Path:** `stripe-payment`  
  - **Webhook ID:** `stripe-payment` (internal identifier)
- **Key expressions/variables:** None.
- **Inputs:** None (trigger node).
- **Outputs:** Sends the incoming request payload to **Format Receipt**.
- **Version-specific notes:** TypeVersion `2` (modern webhook node behavior).
- **Edge cases / failures:**
  - **Stripe signature validation is not implemented** here (no verification of `Stripe-Signature`). This means the endpoint could be called by non-Stripe sources unless protected externally.
  - If Stripe sends an unexpected event shape, downstream parsing may fail or produce empty fields.

---

### 1.2 Receipt Formatting & Data Extraction

**Overview:**  
Transforms the Stripe event into a MoltFlow send-message payload. If there’s no phone number, it marks the item as `skip: true`.

**Nodes involved:**
- **Format Receipt**

**Node details**

#### Node: Format Receipt
- **Type / role:** `Code` (n8n-nodes-base.code) — custom JavaScript transformation.
- **Configuration (interpreted):**
  - **Mode:** `runOnceForAllItems` (the code runs once and returns a single item).
  - Extracts the inbound data from:
    - `body = $input.first().json.body || $input.first().json`
    - `event = body.data ? body.data.object : body`
  - Builds output fields required by MoltFlow:
    - `session_id` (static placeholder)
    - `chat_id` derived from phone (`<digits>@c.us`)
    - `message` (templated receipt text)
    - plus informational fields (`customer_name`, `amount`, `currency`, `phone`, `skip`)
- **Key expressions/variables used:**
  - `const SESSION_ID = 'YOUR_SESSION_ID';` **(must be replaced)**
  - Phone normalization: `phone.replace(/[^0-9]/g, '')`
  - Amount formatting: `(event.amount_total / 100).toFixed(2)`
  - Currency uppercase: `(event.currency || 'usd').toUpperCase()`
  - Payment reference: `event.payment_intent || event.id || ''`
  - Skip logic:
    - If no phone or `phone.length < 7` → returns `{ skip: true, reason: 'No phone number on payment', ... }`
- **Inputs:** From **Stripe Webhook** (raw webhook payload).
- **Outputs:** To **Has Phone?**
- **Version-specific notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - If Stripe payload is not in expected structure, `event.customer_details` may be undefined (handled with conditional checks).
  - If `amount_total` is missing, amount becomes `'0.00'`.
  - If `paymentId` is empty, `paymentId.slice(-8)` yields an empty string (safe but may reduce usefulness).
  - Uses WhatsApp `@c.us` addressing; if MoltFlow expects a different format for certain accounts, delivery may fail.

---

### 1.3 Routing: Phone Available?

**Overview:**  
A simple gate: only send WhatsApp if the formatter did not mark the item as skipped.

**Nodes involved:**
- **Has Phone?**

**Node details**

#### Node: Has Phone?
- **Type / role:** `IF` (n8n-nodes-base.if) — conditional routing.
- **Configuration (interpreted):**
  - Condition: `{{$json.skip}} equals false` (boolean strict comparison).
  - True branch → send WhatsApp
  - False branch → no connected node (workflow ends silently)
- **Key expressions/variables used:**
  - `={{ $json.skip }}`
- **Inputs:** From **Format Receipt**
- **Outputs:**
  - **True:** to **Send WhatsApp Receipt**
  - **False:** unconnected (execution ends)
- **Version-specific notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - If `skip` is missing or not boolean (e.g., string `"false"`), strict type validation could cause unexpected routing. Current formatter always sets boolean, so this is mostly safe unless modified.

---

### 1.4 WhatsApp Delivery via MoltFlow + Logging

**Overview:**  
Sends a WhatsApp message via MoltFlow API using header-based authentication, then outputs a small success log.

**Nodes involved:**
- **Send WhatsApp Receipt**
- **Log Success**

**Node details**

#### Node: Send WhatsApp Receipt
- **Type / role:** `HTTP Request` (n8n-nodes-base.httpRequest) — calls MoltFlow send-message endpoint.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Body type:** JSON (`specifyBody: json`, `sendBody: true`)
  - **Request body content:** built from upstream JSON:
    - `session_id: $json.session_id`
    - `chat_id: $json.chat_id`
    - `message: $json.message`
  - **Authentication:** Generic credential type → `httpHeaderAuth`
    - Credential name: **“MoltFlow API Key”**
    - Expected header name (per sticky note): `X-API-Key`
- **Key expressions/variables used:**
  - Body expression:  
    `={{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
- **Inputs:** From **Has Phone?** (true branch).
- **Outputs:** To **Log Success**
- **Version-specific notes:** TypeVersion `4.2` (HTTP Request node has evolving auth/body options across versions).
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid or header name incorrect.
  - 4xx if `session_id` invalid or WhatsApp session not connected on MoltFlow side.
  - 429 rate limits if sending high volume.
  - Message delivery can succeed at API level but still fail downstream in WhatsApp; depends on MoltFlow’s semantics.

#### Node: Log Success
- **Type / role:** `Code` (n8n-nodes-base.code) — create a simplified confirmation output.
- **Configuration (interpreted):**
  - Reads data from **Format Receipt** node directly:  
    `const data = $('Format Receipt').first().json;`
  - Returns:
    - `status: 'sent'`
    - `customer: <customer_name>`
    - `amount: <currency> <amount>`
- **Inputs:** From **Send WhatsApp Receipt**
- **Outputs:** None (end of workflow).
- **Version-specific notes:** TypeVersion `2`.
- **Edge cases / failures:**
  - If node names change (“Format Receipt”), the selector `$('Format Receipt')` will break.
  - If multiple items are introduced later, `.first()` may not correspond to the item being logged.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / visual annotation |  |  | ## Stripe Payment → WhatsApp Receipt; Send an instant WhatsApp confirmation when a customer's Stripe payment succeeds, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Stripe fires a `checkout.session.completed` webhook; 2. Customer name, email, phone, and amount are extracted; 3. A WhatsApp receipt message is sent via MoltFlow; 4. Result is logged for tracking |
| Sticky Note1 | Sticky Note | Documentation / setup steps |  |  | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Stripe Dashboard → Developers → Webhooks, add the n8n URL for `checkout.session.completed`; 4. Set `YOUR_SESSION_ID` in the Format Receipt node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Note:** Customer phone is pulled from Stripe's `customer_details.phone` |
| Stripe Webhook | Webhook | Receive Stripe webhook event |  | Format Receipt |  |
| Format Receipt | Code | Parse Stripe event, build WhatsApp message payload, skip if no phone | Stripe Webhook | Has Phone? |  |
| Has Phone? | IF | Gate sending based on `skip` flag | Format Receipt | Send WhatsApp Receipt (true); (false unconnected) |  |
| Send WhatsApp Receipt | HTTP Request | Send WhatsApp message via MoltFlow API | Has Phone? | Log Success |  |
| Log Success | Code | Output simplified “sent” log | Send WhatsApp Receipt |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Notify Customers on WhatsApp When Stripe Payment Succeeds with MoltFlow”**
   - (Optional) Add tags: `whatsapp`, `stripe`, `payments`, `moltflow`, `receipts`

2. **Add Webhook trigger**
   - Add node: **Webhook**
   - Set:
     - **HTTP Method:** `POST`
     - **Path:** `stripe-payment`
   - Copy the **Production Webhook URL** (later used in Stripe).

3. **Add formatter Code node**
   - Add node: **Code** named **Format Receipt**
   - Set **Mode:** “Run Once for All Items”
   - Paste logic that:
     - Reads inbound payload from `$input.first().json.body || $input.first().json`
     - Sets `event = body.data.object` when present
     - Extracts:
       - `customer_details.name`, `customer_details.phone`, `customer_details.email`
       - `amount_total` (divide by 100)
       - `currency`
       - `payment_intent` (or fallback `id`)
     - Normalizes phone to digits and builds:
       - `chat_id = phone + '@c.us'`
     - Creates `message` string
     - Outputs `{ skip: true }` if phone missing/too short
   - **Important:** Replace `YOUR_SESSION_ID` with your MoltFlow session id value.

4. **Add IF node**
   - Add node: **IF** named **Has Phone?**
   - Condition:
     - Left value: `{{$json.skip}}`
     - Operation: **equals**
     - Right value: `false` (boolean)
   - Ensure it is a boolean comparison (strict).

5. **Add MoltFlow HTTP Request node**
   - Add node: **HTTP Request** named **Send WhatsApp Receipt**
   - Configure:
     - **Method:** `POST`
     - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
     - **Send Body:** enabled
     - **Body Content Type:** JSON
     - **JSON body** fields:
       - `session_id: {{$json.session_id}}`
       - `chat_id: {{$json.chat_id}}`
       - `message: {{$json.message}}`
   - **Credentials (Header Auth):**
     - Create credential: **HTTP Header Auth**
     - Header name: `X-API-Key`
     - Value: your MoltFlow API key
     - Select this credential in the node.

6. **Add logging Code node**
   - Add node: **Code** named **Log Success**
   - Set Mode: “Run Once for All Items”
   - Create output like:
     - `status: 'sent'`
     - `customer: <customer_name>`
     - `amount: <currency> <amount>`
   - If you want the log to match the current workflow, read from the node by name:
     - `const data = $('Format Receipt').first().json;`

7. **Connect nodes in order**
   - `Stripe Webhook` → `Format Receipt`
   - `Format Receipt` → `Has Phone?`
   - `Has Phone?` **true output** → `Send WhatsApp Receipt`
   - `Send WhatsApp Receipt` → `Log Success`
   - Leave the **false** branch of `Has Phone?` unconnected (or connect to a logger if desired).

8. **Configure Stripe webhook**
   - In **Stripe Dashboard → Developers → Webhooks**
   - Add endpoint: your n8n Webhook Production URL
   - Subscribe to event: `checkout.session.completed`
   - (Recommended enhancement) Add signature verification in n8n or via a Stripe verification step.

9. **Activate workflow**
   - Activate in n8n
   - Run a test payment through Stripe Checkout and confirm the WhatsApp message is received.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow service used to send WhatsApp messages | https://molt.waiflow.app |
| MoltFlow API base used in the HTTP Request node | `https://apiv2.waiflow.app/api/v2/messages/send` |
| Stripe event expected | `checkout.session.completed` |
| Phone source in Stripe payload | `customer_details.phone` |
| Security note: Stripe signature verification is not present in this workflow | Consider validating `Stripe-Signature` to ensure requests are authentic |