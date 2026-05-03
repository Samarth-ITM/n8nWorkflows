Process e-commerce orders with payment verification, inventory, Gmail, and Slack

https://n8nworkflows.xyz/workflows/process-e-commerce-orders-with-payment-verification--inventory--gmail--and-slack-13197


# Process e-commerce orders with payment verification, inventory, Gmail, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow processes incoming e-commerce orders end-to-end: it receives an order via webhook, validates the payload, verifies payment, checks and reserves inventory, creates the order and shipping label via APIs, sends a confirmation email (Gmail), notifies the team (Slack), logs an audit record, and finally responds to the webhook caller. A separate error-handling path alerts Slack whenever the workflow errors.

**Target use cases:**
- Headless commerce storefronts sending “new order” events to n8n
- Payment-provider verification gates before fulfillment
- Inventory reservation and fulfillment orchestration via internal APIs
- Operational notifications (customer via email, team via Slack)
- Auditable processing with centralized logging

### Logical Blocks (by dependency chain)
1.1 **Input Reception & Validation**: Webhook → Validate payload → route valid vs invalid  
1.2 **Payment Verification**: Verify payment API → route paid vs failed  
1.3 **Inventory Check**: Stock check API → route in-stock vs out-of-stock  
1.4 **Fulfillment Orchestration**: Reserve inventory → create order → generate shipping → prepare email  
1.5 **Notifications**: Send Gmail confirmation → Slack team notification  
1.6 **Audit & Webhook Response**: Prepare audit entry → log audit → respond success  
1.7 **Global Error Handling**: Error Trigger → format error → Slack alert

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Validation
**Overview:** Accepts incoming order requests and validates the order structure. Invalid orders get a structured error response immediately.

**Nodes Involved:**
- Webhook - New Order
- Validate Order
- IF - Valid Order
- Format Validation Error
- Respond - Validation Error

#### Node: **Webhook - New Order**
- **Type / role:** `Webhook` (entry point). Receives the new order payload.
- **Configuration (interpreted):** No parameters shown in JSON (defaults). Has a placeholder `webhookId: YOUR_WEBHOOK_ID`.
- **Key variables:** Incoming JSON becomes the item data available to downstream nodes.
- **Connections:** Output → **Validate Order**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - Wrong HTTP method/path settings (not visible here) can cause 404/405.
  - Large payloads/timeouts if caller expects quick response; design already includes early responses for failures.
  - If “Respond to Webhook” is not configured to respond automatically, the request may hang unless explicitly responded later.

#### Node: **Validate Order**
- **Type / role:** `Code` node (custom validation).
- **Configuration:** Not provided in JSON; expected to check required fields (e.g., orderId, customer email, line items, totals).
- **Expressions/variables:** Likely reads `$json` and outputs a normalized structure plus a validation flag/reasons.
- **Connections:** Output → **IF - Valid Order**
- **Version:** typeVersion **2**
- **Edge cases / failures:**
  - Missing expected keys can throw runtime errors in code if not guarded.
  - If it returns no items, the flow stops (caller may never get a response).

#### Node: **IF - Valid Order**
- **Type / role:** `IF` router.
- **Configuration:** Not provided; expected condition like “isValid === true”.
- **Connections:**
  - True → **API - Verify Payment**
  - False → **Format Validation Error**
- **Version:** typeVersion **2**
- **Edge cases / failures:** Misconfigured condition can invert behavior (valid orders rejected or invalid accepted).

#### Node: **Format Validation Error**
- **Type / role:** `Code` node to create a consistent error payload for the webhook response.
- **Connections:** Output → **Respond - Validation Error**
- **Version:** typeVersion **2**
- **Edge cases:** Should ensure proper HTTP status and message; otherwise caller may retry incorrectly.

#### Node: **Respond - Validation Error**
- **Type / role:** `Respond to Webhook` (terminates request with error).
- **Configuration:** Not provided; typically sets status code (e.g., 400) and body from previous node.
- **Connections:** No outgoing connections.
- **Version:** typeVersion **1.1**
- **Edge cases:** If webhook response mode is “Last node” but execution doesn’t reach this node, requests may time out.

---

### 2.2 Payment Verification
**Overview:** Calls a payment verification endpoint and routes to either fulfillment (paid) or early failure response.

**Nodes Involved:**
- API - Verify Payment
- IF - Payment OK
- Format Payment Failed
- Respond - Payment Failed

#### Node: **API - Verify Payment**
- **Type / role:** `HTTP Request` to a payment gateway/internal service.
- **Configuration:** Not shown; expected method/URL/auth headers and mapping order/payment identifiers.
- **Connections:** Output → **IF - Payment OK**
- **Version:** typeVersion **4.2**
- **Edge cases / failures:**
  - 401/403 if credentials missing/expired.
  - 4xx for invalid payment reference, 5xx for provider outage.
  - Timeouts; consider retries/backoff in node options.

#### Node: **IF - Payment OK**
- **Type / role:** `IF` router (paid vs not paid).
- **Configuration:** Not shown; likely checks response fields like `status === "paid"` or `verified === true`.
- **Connections:**
  - True → **API - Check Stock**
  - False → **Format Payment Failed**
- **Version:** typeVersion **2**
- **Edge cases:** Payment APIs vary (authorized vs captured). Condition should match your business rule.

#### Node: **Format Payment Failed**
- **Type / role:** `Code` node formatting failure response.
- **Connections:** Output → **Respond - Payment Failed**
- **Version:** typeVersion **2**

#### Node: **Respond - Payment Failed**
- **Type / role:** `Respond to Webhook` error response.
- **Version:** typeVersion **1.1**
- **Edge cases:** Should return an appropriate status (commonly 402/400/409 depending on design).

---

### 2.3 Inventory Check
**Overview:** Checks inventory availability and blocks fulfillment if items are out of stock.

**Nodes Involved:**
- API - Check Stock
- IF - In Stock
- Format Out of Stock
- Respond - Out of Stock

#### Node: **API - Check Stock**
- **Type / role:** `HTTP Request` to inventory service.
- **Configuration:** Not shown; expected to pass SKU/quantity list.
- **Connections:** Output → **IF - In Stock**
- **Version:** typeVersion **4.2**
- **Edge cases / failures:** Partial stock scenarios (some items available) must be handled explicitly in API or IF logic.

#### Node: **IF - In Stock**
- **Type / role:** `IF` router.
- **Configuration:** Not shown; likely checks `allAvailable === true`.
- **Connections:**
  - True → **API - Reserve Inventory**
  - False → **Format Out of Stock**
- **Version:** typeVersion **2**

#### Node: **Format Out of Stock**
- **Type / role:** `Code` node to format stock failure response.
- **Connections:** Output → **Respond - Out of Stock**
- **Version:** typeVersion **2**

#### Node: **Respond - Out of Stock**
- **Type / role:** `Respond to Webhook` error response.
- **Version:** typeVersion **1.1**
- **Edge cases:** Decide whether to use 409 Conflict, 400, or 200 with business error object (depending on caller expectations).

---

### 2.4 Fulfillment Orchestration
**Overview:** Reserves inventory, creates the order record, generates shipping, then prepares email content.

**Nodes Involved:**
- API - Reserve Inventory
- API - Create Order
- API - Generate Shipping
- Format Email

#### Node: **API - Reserve Inventory**
- **Type / role:** `HTTP Request` to reserve stock.
- **Connections:** Output → **API - Create Order**
- **Version:** typeVersion **4.2**
- **Edge cases / failures:**
  - Reservation race conditions (stock changes between check and reserve).
  - Idempotency: retrying reservation may double-reserve unless reservation keys are used.

#### Node: **API - Create Order**
- **Type / role:** `HTTP Request` to create/order management system entry.
- **Connections:** Output → **API - Generate Shipping**
- **Version:** typeVersion **4.2**
- **Edge cases:** If shipping generation fails after order creation, you need compensating actions (cancel order / release reservation) — not implemented here.

#### Node: **API - Generate Shipping**
- **Type / role:** `HTTP Request` to shipping provider/internal shipping service.
- **Connections:** Output → **Format Email**
- **Version:** typeVersion **4.2**
- **Edge cases:** Label generation can be slow; configure timeouts and retries carefully.

#### Node: **Format Email**
- **Type / role:** `Code` node to build the email: recipient, subject, HTML/text, and include order/shipping details.
- **Connections:** Output → **Send Confirmation Email**
- **Version:** typeVersion **2**
- **Edge cases:** Ensure customer email exists and is validated; sanitize user-provided strings if templating HTML.

---

### 2.5 Notifications
**Overview:** Sends customer confirmation via Gmail and posts an internal Slack notification to the team.

**Nodes Involved:**
- Send Confirmation Email
- Slack - Notify Team

#### Node: **Send Confirmation Email**
- **Type / role:** `Gmail` node (send message).
- **Configuration:** Not shown; requires Gmail OAuth2 credentials in n8n.
- **Connections:** Output → **Slack - Notify Team**
- **Version:** typeVersion **2.1**
- **Edge cases / failures:**
  - OAuth token expiration/revocation.
  - Gmail API quota / “from” alias restrictions.
  - Invalid recipient addresses causing bounce (Gmail may still accept send).

#### Node: **Slack - Notify Team**
- **Type / role:** `Slack` node (send message).
- **Configuration:** Not shown; typically uses Slack OAuth or bot token credentials in n8n.
- **Connections:** Output → **Prepare Audit Entry**
- **Version:** typeVersion **2.2**
- **Edge cases:** Channel permissions, missing scopes, rate limits.

---

### 2.6 Audit & Webhook Response
**Overview:** Creates an audit record for traceability and returns a success response to the webhook caller.

**Nodes Involved:**
- Prepare Audit Entry
- API - Log Audit
- Format Success
- Respond - Success

#### Node: **Prepare Audit Entry**
- **Type / role:** `Code` node to assemble audit payload (order id, payment status, stock reservation id, timestamps, correlation ids).
- **Connections:** Output → **API - Log Audit**
- **Version:** typeVersion **2**

#### Node: **API - Log Audit**
- **Type / role:** `HTTP Request` to logging/audit service.
- **Connections:** Output → **Format Success**
- **Version:** typeVersion **4.2**
- **Edge cases:** If audit logging fails, the workflow currently cannot reach success response unless errors are handled; consider “Continue On Fail” or a fallback.

#### Node: **Format Success**
- **Type / role:** `Code` node building the final response body (e.g., `{ ok: true, orderId, shippingTracking }`).
- **Connections:** Output → **Respond - Success**
- **Version:** typeVersion **2**

#### Node: **Respond - Success**
- **Type / role:** `Respond to Webhook` final response.
- **Version:** typeVersion **1.1**
- **Edge cases:** Ensure only one response path is executed for a given webhook run.

---

### 2.7 Global Error Handling
**Overview:** Catches workflow execution errors (node failures) and alerts Slack with a formatted error message.

**Nodes Involved:**
- Error Trigger
- Format Error
- Slack - Error Alert

#### Node: **Error Trigger**
- **Type / role:** `Error Trigger` (secondary entry point). Fires when workflow execution errors occur.
- **Connections:** Output → **Format Error**
- **Version:** typeVersion **1**
- **Edge cases:** Only triggers for actual execution errors, not business-rule “false” branches that respond normally.

#### Node: **Format Error**
- **Type / role:** `Code` node to format error context (failed node name, message, stack, execution id).
- **Connections:** Output → **Slack - Error Alert**
- **Version:** typeVersion **2**
- **Edge cases:** Error object shape differs by node; code should defensively access fields.

#### Node: **Slack - Error Alert**
- **Type / role:** `Slack` node posting incident alert.
- **Version:** typeVersion **2.2**
- **Edge cases:** If Slack is down or token invalid, errors won’t be delivered; consider adding email/SMS fallback.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Section label | — | — |  |
| Order intake | Sticky Note | Section label | — | — |  |
| Webhook - New Order | Webhook | Receives new order payload | — | Validate Order |  |
| Validate Order | Code | Validates/normalizes order data | Webhook - New Order | IF - Valid Order |  |
| IF - Valid Order | IF | Routes valid vs invalid order | Validate Order | API - Verify Payment; Format Validation Error |  |
| Format Validation Error | Code | Builds validation error response | IF - Valid Order (false) | Respond - Validation Error |  |
| Respond - Validation Error | Respond to Webhook | Returns validation failure | Format Validation Error | — |  |
| Payment verification | Sticky Note | Section label | — | — |  |
| API - Verify Payment | HTTP Request | Verifies payment status | IF - Valid Order (true) | IF - Payment OK |  |
| IF - Payment OK | IF | Routes paid vs payment failed | API - Verify Payment | API - Check Stock; Format Payment Failed |  |
| Format Payment Failed | Code | Builds payment failure response | IF - Payment OK (false) | Respond - Payment Failed |  |
| Respond - Payment Failed | Respond to Webhook | Returns payment failure | Format Payment Failed | — |  |
| Stock verification | Sticky Note | Section label | — | — |  |
| API - Check Stock | HTTP Request | Checks stock availability | IF - Payment OK (true) | IF - In Stock |  |
| IF - In Stock | IF | Routes in-stock vs out-of-stock | API - Check Stock | API - Reserve Inventory; Format Out of Stock |  |
| Format Out of Stock | Code | Builds out-of-stock response | IF - In Stock (false) | Respond - Out of Stock |  |
| Respond - Out of Stock | Respond to Webhook | Returns out-of-stock error | Format Out of Stock | — |  |
| Order fulfillment | Sticky Note | Section label | — | — |  |
| API - Reserve Inventory | HTTP Request | Reserves inventory | IF - In Stock (true) | API - Create Order |  |
| API - Create Order | HTTP Request | Creates order in OMS | API - Reserve Inventory | API - Generate Shipping |  |
| API - Generate Shipping | HTTP Request | Generates shipment/label | API - Create Order | Format Email |  |
| Format Email | Code | Prepares email payload | API - Generate Shipping | Send Confirmation Email |  |
| Notifications | Sticky Note | Section label | — | — |  |
| Send Confirmation Email | Gmail | Sends customer confirmation | Format Email | Slack - Notify Team |  |
| Slack - Notify Team | Slack | Notifies internal team | Send Confirmation Email | Prepare Audit Entry |  |
| Audit and response | Sticky Note | Section label | — | — |  |
| Prepare Audit Entry | Code | Builds audit log payload | Slack - Notify Team | API - Log Audit |  |
| API - Log Audit | HTTP Request | Sends audit record to logging API | Prepare Audit Entry | Format Success |  |
| Format Success | Code | Builds success response | API - Log Audit | Respond - Success |  |
| Respond - Success | Respond to Webhook | Returns success | Format Success | — |  |
| Error handling | Sticky Note | Section label | — | — |  |
| Error Trigger | Error Trigger | Catches execution errors | — | Format Error |  |
| Format Error | Code | Formats error alert content | Error Trigger | Slack - Error Alert |  |
| Slack - Error Alert | Slack | Posts error alert | Format Error | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **Process e-commerce orders with payment verification, inventory, Gmail, and Slack**

2) **Add entry node: Webhook**
- Add node: **Webhook** → rename **Webhook - New Order**
- Configure:
  - Choose the HTTP method your storefront will call (commonly POST)
  - Set a path like `/orders/new`
  - Response mode: use explicit **Respond to Webhook** nodes (recommended since this workflow has multiple response paths)

3) **Add order validation**
- Add node: **Code** → rename **Validate Order**
- Implement validation and normalization (example expectations):
  - Required: `orderId`, `customer.email`, `items[]` with `sku`, `qty`, and order `total`
  - Output something like:
    - `isValid: boolean`
    - `validationErrors: string[]`
    - normalized fields for later nodes
- Connect: **Webhook - New Order → Validate Order**

4) **Route valid vs invalid**
- Add node: **IF** → rename **IF - Valid Order**
- Condition: `{{$json.isValid}}` is true (or your chosen flag)
- Connect: **Validate Order → IF - Valid Order**

5) **Invalid order response path**
- Add node: **Code** → rename **Format Validation Error**
  - Build response body and desired HTTP status (commonly 400)
- Add node: **Respond to Webhook** → rename **Respond - Validation Error**
  - Set body from **Format Validation Error**
  - Set HTTP status (if supported in your configuration)
- Connect:  
  - **IF - Valid Order (false) → Format Validation Error → Respond - Validation Error**

6) **Payment verification**
- Add node: **HTTP Request** → rename **API - Verify Payment**
- Configure:
  - URL: your payment verification endpoint
  - Auth: API key/OAuth as required
  - Send required identifiers from the order (e.g., `paymentId`, `orderId`, amount)
  - Parse JSON response
- Connect: **IF - Valid Order (true) → API - Verify Payment**

7) **Route payment OK vs failed**
- Add node: **IF** → rename **IF - Payment OK**
- Condition based on payment API response (e.g., `status === "paid"` / `verified === true`)
- Connect: **API - Verify Payment → IF - Payment OK**

8) **Payment failed response path**
- Add **Code**: **Format Payment Failed**
- Add **Respond to Webhook**: **Respond - Payment Failed**
- Connect: **IF - Payment OK (false) → Format Payment Failed → Respond - Payment Failed**

9) **Stock check**
- Add **HTTP Request**: **API - Check Stock**
  - Pass items SKUs/qty
  - Expect response indicating availability
- Connect: **IF - Payment OK (true) → API - Check Stock**

10) **Route in-stock vs out-of-stock**
- Add **IF**: **IF - In Stock**
  - Condition like `{{$json.allAvailable === true}}` (depending on your stock API response)
- Connect: **API - Check Stock → IF - In Stock**

11) **Out-of-stock response path**
- Add **Code**: **Format Out of Stock**
- Add **Respond to Webhook**: **Respond - Out of Stock**
- Connect: **IF - In Stock (false) → Format Out of Stock → Respond - Out of Stock**

12) **Reserve inventory**
- Add **HTTP Request**: **API - Reserve Inventory**
  - Use an idempotency key (recommended) derived from `orderId`
- Connect: **IF - In Stock (true) → API - Reserve Inventory**

13) **Create order**
- Add **HTTP Request**: **API - Create Order**
  - Send reservation reference + customer/order details
- Connect: **API - Reserve Inventory → API - Create Order**

14) **Generate shipping**
- Add **HTTP Request**: **API - Generate Shipping**
  - Use shipping address + items/weight + service level
- Connect: **API - Create Order → API - Generate Shipping**

15) **Prepare confirmation email content**
- Add **Code**: **Format Email**
  - Output fields needed by Gmail node: `to`, `subject`, `message` (text/html), and include order/shipping details
- Connect: **API - Generate Shipping → Format Email**

16) **Send Gmail confirmation**
- Add **Gmail** node: **Send Confirmation Email**
- Credentials:
  - Configure **Gmail OAuth2** in n8n credentials
  - Ensure scopes allow sending email
- Map fields from **Format Email**
- Connect: **Format Email → Send Confirmation Email**

17) **Slack notify team**
- Add **Slack** node: **Slack - Notify Team**
- Credentials:
  - Slack OAuth/bot token with permission to post to the chosen channel
- Compose message with orderId, totals, and shipping/tracking
- Connect: **Send Confirmation Email → Slack - Notify Team**

18) **Prepare and log audit**
- Add **Code**: **Prepare Audit Entry**
  - Build an audit payload (orderId, payment verification result, reservation id, shipment id, timestamps)
- Add **HTTP Request**: **API - Log Audit**
  - Send to your audit/logging endpoint
- Connect: **Slack - Notify Team → Prepare Audit Entry → API - Log Audit**

19) **Success response**
- Add **Code**: **Format Success**
  - Create final response body (include orderId, shipping tracking, etc.)
- Add **Respond to Webhook**: **Respond - Success**
- Connect: **API - Log Audit → Format Success → Respond - Success**

20) **Global error handling**
- Add node: **Error Trigger**
- Add node: **Code** → **Format Error** (extract `execution.id`, `error.message`, `node.name` if available)
- Add node: **Slack** → **Slack - Error Alert**
- Connect: **Error Trigger → Format Error → Slack - Error Alert**
- Configure Slack to post to an incidents/errors channel.

21) **(Optional but recommended) Hardening**
- Add timeouts/retries to HTTP Request nodes.
- Make payment, reserve-inventory, and create-order calls idempotent.
- Ensure only one webhook response path can execute per run.
- Decide what happens if audit logging fails (continue and still respond success vs fail the whole run).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| No sticky note content was provided (all section sticky notes are empty). | Workflow canvas annotations |
| Webhook node uses a placeholder webhookId (`YOUR_WEBHOOK_ID`). Replace by activating the workflow and using the generated Production/Test webhook URLs. | Webhook configuration in n8n |