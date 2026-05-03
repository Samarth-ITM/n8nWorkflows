Track inventory levels with automated alerts

https://n8nworkflows.xyz/workflows/track-inventory-levels-with-automated-alerts-13165


# Track inventory levels with automated alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow tracks inventory movements received via an inbound webhook, records the movement in an external system, fetches updated stock levels, evaluates thresholds, and sends automated alerts (Slack + optional email). It also logs an audit trail and returns a structured webhook response. A separate error-handling path posts runtime errors to Slack.

**Target use cases:**
- Receiving stock-in / stock-out events from POS, WMS, ERP, or e-commerce systems
- Monitoring low-stock thresholds and notifying operations in real time
- Maintaining an audit trail of inventory-related automation actions

**Logical blocks (by dependency and function):**
1.1 **Movement Intake**: Webhook reception → basic validation gating  
1.2 **Stock Update**: Format movement → record movement via API → fetch current stock  
1.3 **Alert System**: Evaluate thresholds → route alert severity → send notifications  
1.4 **Audit & Response**: Write audit entry → return success response to caller  
1.5 **Error Handling**: Global error trigger → format error payload → Slack alert

---

## 2. Block-by-Block Analysis

### 2.1 Movement Intake

**Overview:**  
Receives an inventory movement event through a webhook endpoint, validates the incoming payload, and either proceeds or immediately returns a validation error response.

**Nodes involved:**
- Webhook - Inventory Movement
- Validate Movement
- IF - Valid Movement
- Format Validation Error
- Respond - Validation Error

#### Node: Webhook - Inventory Movement
- **Type / role:** `Webhook` — workflow entry point; accepts HTTP requests.
- **Configuration (interpreted):**
  - Uses a predefined `webhookId` placeholder (`YOUR_WEBHOOK_ID`), implying this is a template and must be generated in your n8n instance.
  - HTTP method, path, authentication, and response mode are not defined in the JSON (defaults apply).
- **Inputs / outputs:**
  - **Output:** to **Validate Movement**
- **Version notes:** `typeVersion: 2` (newer webhook node behavior vs older versions).
- **Potential failures / edge cases:**
  - Wrong method/path used by the caller (404/405 depending on configuration)
  - Large payload size or invalid JSON (depending on n8n/web server limits)
  - If “Respond to Webhook” nodes are used, ensure webhook “Response mode” is compatible (commonly “Using Respond to Webhook”).

#### Node: Validate Movement
- **Type / role:** `Code` — validates and possibly normalizes incoming movement data.
- **Configuration choices:**
  - The code is not included in the JSON, so the intended validation rules must be implemented by the user.
  - Typically this node would:
    - Verify required fields (e.g., `sku`, `location`, `quantity`, `movementType`, `timestamp`)
    - Enforce numeric quantity and allowed movement types (IN/OUT/ADJUST)
    - Output a boolean/flag used by the downstream IF node
- **Expressions / variables:** Not visible (empty parameters).
- **Inputs / outputs:**
  - **Input:** Webhook payload
  - **Output:** to **IF - Valid Movement**
- **Edge cases:**
  - Missing/undefined fields causing runtime errors in code
  - Quantity as string, null, or negative where not expected
  - Multiple items/batch payload vs single movement event

#### Node: IF - Valid Movement
- **Type / role:** `IF` — branches between valid and invalid payload paths.
- **Configuration choices:**
  - Rules are not present (empty parameters); must be configured to check the validation result from **Validate Movement** (e.g., `{{$json.valid === true}}`).
- **Inputs / outputs:**
  - **True branch (output 0):** to **Prepare API Body**
  - **False branch (output 1):** to **Format Validation Error**
- **Edge cases:**
  - If validation node does not produce a deterministic boolean, routing may misbehave
  - If multiple items are output, IF evaluates per item

#### Node: Format Validation Error
- **Type / role:** `Code` — builds an error response body for invalid requests.
- **Configuration choices:** Not included; typically sets:
  - HTTP status code (e.g., 400)
  - JSON response structure (e.g., `{ success:false, error:"...", details:... }`)
- **Inputs / outputs:**
  - **Output:** to **Respond - Validation Error**
- **Edge cases:** code assumptions about fields present in invalid payload

#### Node: Respond - Validation Error
- **Type / role:** `Respond to Webhook` — returns a response and terminates the webhook request.
- **Configuration choices:** Not shown; typically uses JSON from previous node, status code 4xx.
- **Inputs / outputs:**
  - **Input:** formatted error payload
  - **No outputs** (ends request lifecycle)
- **Version notes:** `typeVersion: 1.1`.
- **Edge cases:**
  - If webhook is not configured to “respond via Respond to Webhook,” caller may hang or n8n may auto-respond unexpectedly.

---

### 2.2 Stock Update

**Overview:**  
Transforms validated movement data into the external API schema, records the movement, then queries the updated stock level for the relevant item(s).

**Nodes involved:**
- Prepare API Body
- API - Record Movement
- API - Get Current Stock

#### Node: Prepare API Body
- **Type / role:** `Code` — maps/constructs request body for the movement-recording API.
- **Configuration choices:** Missing; typically builds:
  - Endpoint payload with SKU/product ID, warehouse/location, delta quantity, reference/order ID
- **Inputs / outputs:**
  - **Output:** to **API - Record Movement**
- **Edge cases:**
  - Wrong mapping leading to rejected API calls
  - Handling movement batches vs single movement

#### Node: API - Record Movement
- **Type / role:** `HTTP Request` — writes movement to inventory system.
- **Configuration choices:** Missing; must define:
  - Method (usually POST)
  - URL (inventory service endpoint)
  - Auth (API key/OAuth2)
  - JSON body from **Prepare API Body**
- **Inputs / outputs:**
  - **Output:** to **API - Get Current Stock**
- **Version notes:** `typeVersion: 4.2` (relevant for authentication options and response handling behavior).
- **Potential failures:**
  - 401/403 auth failures
  - 4xx validation errors from remote API
  - 5xx service outages
  - Timeouts / rate-limits

#### Node: API - Get Current Stock
- **Type / role:** `HTTP Request` — reads current stock after recording movement.
- **Configuration choices:** Missing; must define:
  - Method (GET/POST depending on API)
  - URL (stock lookup endpoint)
  - Query params/path param (SKU/location) derived from prior nodes
- **Inputs / outputs:**
  - **Output:** to **Check Stock Levels**
- **Potential failures / edge cases:**
  - Eventual consistency: stock may not reflect the movement immediately
  - Missing SKU/location in the remote system
  - Response schema differences (e.g., `available`, `onHand`, `reserved`)

---

### 2.3 Alert System

**Overview:**  
Evaluates stock levels against thresholds, routes to a severity lane (critical/urgent/warning/none), formats messages, and sends Slack (and email for critical).

**Nodes involved:**
- Check Stock Levels
- Alert Router
- Format Critical Alert
- Slack - Critical Alert
- Email - Critical Alert
- Format Urgent Alert
- Slack - Urgent Alert
- Format Warning Alert
- Slack - Warning Alert
- No Alert Needed

#### Node: Check Stock Levels
- **Type / role:** `Code` — derives alert severity and message context from stock data.
- **Configuration choices:** Not included; typically:
  - Reads current stock quantity
  - Compares to thresholds (e.g., `<= 0` critical, `<= 5` urgent, `<= 10` warning)
  - Outputs a routing field (e.g., `severity = "critical"|"urgent"|"warning"|"none"`)
- **Inputs / outputs:**
  - **Output:** to **Alert Router**
- **Edge cases:**
  - Stock field not present or non-numeric
  - Multiple SKUs: requires per-item evaluation and routing

#### Node: Alert Router
- **Type / role:** `Switch` — routes by computed severity.
- **Configuration choices:** Empty in JSON; must be set to match **Check Stock Levels** output (e.g., switch on `{{$json.severity}}`).
- **Inputs / outputs (configured as 4 branches):**
  1. Output 0 → **Format Critical Alert**
  2. Output 1 → **Format Urgent Alert**
  3. Output 2 → **Format Warning Alert**
  4. Output 3 → **No Alert Needed**
- **Version notes:** `typeVersion: 3`.
- **Edge cases:**
  - If severity values don’t match cases, items may drop (depending on switch settings)
  - Ensure a default route exists (here “No Alert Needed” is likely intended)

#### Node: Format Critical Alert
- **Type / role:** `Code` — builds Slack/email-ready payload for critical stock state.
- **Outputs:** to **Slack - Critical Alert**
- **Edge cases:** message formatting errors, missing product metadata

#### Node: Slack - Critical Alert
- **Type / role:** `Slack` — posts critical alert to Slack.
- **Configuration choices:**
  - Slack node present with a `webhookId` (internal n8n identifier). Actual Slack credential/config must be set in n8n.
  - Operation/channel/message not shown; must be set.
- **Outputs:** to **Email - Critical Alert**
- **Version notes:** `typeVersion: 2.2`.
- **Potential failures:**
  - Missing/invalid Slack credentials
  - Channel not found, insufficient permissions
  - Rate limiting

#### Node: Email - Critical Alert
- **Type / role:** `Gmail` — sends email for critical alerts (in addition to Slack).
- **Configuration choices:**
  - Requires Gmail OAuth2 credentials in n8n.
  - Recipient, subject, and body not defined in JSON.
- **Outputs:** to **Prepare Audit Entry**
- **Version notes:** `typeVersion: 2.1`.
- **Potential failures:**
  - OAuth token expired / missing scopes
  - Sending limits, invalid recipients

#### Node: Format Urgent Alert
- **Type / role:** `Code` — builds Slack message for urgent alerts.
- **Outputs:** to **Slack - Urgent Alert**

#### Node: Slack - Urgent Alert
- **Type / role:** `Slack` — posts urgent alert to Slack.
- **Outputs:** to **Prepare Audit Entry**
- **Potential failures:** same as other Slack nodes

#### Node: Format Warning Alert
- **Type / role:** `Code` — builds Slack message for warning alerts.
- **Outputs:** to **Slack - Warning Alert**

#### Node: Slack - Warning Alert
- **Type / role:** `Slack` — posts warning alert to Slack.
- **Outputs:** to **Prepare Audit Entry**

#### Node: No Alert Needed
- **Type / role:** `No Operation` — explicit pass-through when no alert is required.
- **Outputs:** to **Prepare Audit Entry**
- **Edge cases:** none (used for clarity/branch completeness)

---

### 2.4 Audit & Response

**Overview:**  
Creates an audit record for the processed movement (and whether/what alert was triggered), stores it via API, then returns a success response to the webhook caller.

**Nodes involved:**
- Prepare Audit Entry
- API - Log Audit Trail
- Format Success Response
- Respond - Success

#### Node: Prepare Audit Entry
- **Type / role:** `Code` — constructs an audit log payload.
- **Configuration choices:** Missing; typically includes:
  - Correlation ID / webhook request ID
  - Movement details (SKU, delta, location)
  - Stock after update
  - Alert severity + notification delivery results (if available)
- **Inputs / outputs:**
  - Receives input from **No Alert Needed**, **Slack - Urgent Alert**, **Slack - Warning Alert**, **Email - Critical Alert**
  - Outputs to **API - Log Audit Trail**
- **Edge cases:**
  - Inconsistent payload shape across branches unless normalized
  - If Slack/Gmail nodes output different schemas, audit code must handle both

#### Node: API - Log Audit Trail
- **Type / role:** `HTTP Request` — persists audit entry to an external system (DB service, logging API, etc.).
- **Configuration choices:** Missing; must define URL/method/auth/body.
- **Outputs:** to **Format Success Response**
- **Potential failures:**
  - Audit logging endpoint unavailable; decide whether to fail workflow or proceed (current flow will fail unless “Continue On Fail” is enabled)

#### Node: Format Success Response
- **Type / role:** `Code` — builds final webhook success response body.
- **Outputs:** to **Respond - Success**
- **Edge cases:** ensure response is deterministic regardless of alert branch taken

#### Node: Respond - Success
- **Type / role:** `Respond to Webhook` — sends success response to the caller.
- **Version notes:** `typeVersion: 1.1`.
- **Edge cases:**
  - Must align with webhook response mode
  - If workflow errors before reaching this node, caller receives error/timeout unless separately handled

---

### 2.5 Error Handling

**Overview:**  
Catches workflow execution errors using n8n’s Error Trigger and posts a formatted alert to Slack.

**Nodes involved:**
- Error Trigger
- Format Error
- Slack - Error Alert

#### Node: Error Trigger
- **Type / role:** `Error Trigger` — special trigger that fires when another workflow execution fails (depending on how configured in n8n).
- **Configuration choices:** Not shown.
- **Outputs:** to **Format Error**
- **Version notes:** `typeVersion: 1`.
- **Edge cases:**
  - Requires proper n8n error workflow setup/association
  - May not capture errors if not configured as the global error workflow

#### Node: Format Error
- **Type / role:** `Code` — shapes error details into a Slack-friendly payload.
- **Outputs:** to **Slack - Error Alert**
- **Edge cases:** sensitive data leakage (ensure payload redacts secrets/tokens)

#### Node: Slack - Error Alert
- **Type / role:** `Slack` — posts workflow error notification to Slack.
- **Configuration choices:** Slack credentials required; message/channel not present in JSON.
- **Version notes:** `typeVersion: 2.2`.
- **Potential failures:** Slack auth/permissions/rate limits

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Section label / visual grouping | — | — |  |
| Movement Intake | Sticky Note | Section label / visual grouping | — | — |  |
| Stock Update | Sticky Note | Section label / visual grouping | — | — |  |
| Alert System | Sticky Note | Section label / visual grouping | — | — |  |
| Audit & Response | Sticky Note | Section label / visual grouping | — | — |  |
| Error Handling | Sticky Note | Section label / visual grouping | — | — |  |
| Webhook - Inventory Movement | Webhook | Receive movement event | — | Validate Movement |  |
| Validate Movement | Code | Validate/normalize inbound payload | Webhook - Inventory Movement | IF - Valid Movement |  |
| IF - Valid Movement | IF | Branch on validation result | Validate Movement | Prepare API Body; Format Validation Error |  |
| Format Validation Error | Code | Build 4xx error response payload | IF - Valid Movement | Respond - Validation Error |  |
| Respond - Validation Error | Respond to Webhook | Return validation error to caller | Format Validation Error | — |  |
| Prepare API Body | Code | Map movement to API request body | IF - Valid Movement | API - Record Movement |  |
| API - Record Movement | HTTP Request | Persist movement to external system | Prepare API Body | API - Get Current Stock |  |
| API - Get Current Stock | HTTP Request | Fetch updated stock level(s) | API - Record Movement | Check Stock Levels |  |
| Check Stock Levels | Code | Compute severity from stock thresholds | API - Get Current Stock | Alert Router |  |
| Alert Router | Switch | Route by severity | Check Stock Levels | Format Critical Alert; Format Urgent Alert; Format Warning Alert; No Alert Needed |  |
| Format Critical Alert | Code | Create critical alert message/payload | Alert Router | Slack - Critical Alert |  |
| Slack - Critical Alert | Slack | Send critical alert to Slack | Format Critical Alert | Email - Critical Alert |  |
| Email - Critical Alert | Gmail | Send critical alert email | Slack - Critical Alert | Prepare Audit Entry |  |
| Format Urgent Alert | Code | Create urgent alert payload | Alert Router | Slack - Urgent Alert |  |
| Slack - Urgent Alert | Slack | Send urgent alert to Slack | Format Urgent Alert | Prepare Audit Entry |  |
| Format Warning Alert | Code | Create warning alert payload | Alert Router | Slack - Warning Alert |  |
| Slack - Warning Alert | Slack | Send warning alert to Slack | Format Warning Alert | Prepare Audit Entry |  |
| No Alert Needed | NoOp | Explicit no-alert branch | Alert Router | Prepare Audit Entry |  |
| Prepare Audit Entry | Code | Build audit log payload | No Alert Needed; Slack - Urgent Alert; Slack - Warning Alert; Email - Critical Alert | API - Log Audit Trail |  |
| API - Log Audit Trail | HTTP Request | Write audit entry to external system | Prepare Audit Entry | Format Success Response |  |
| Format Success Response | Code | Build webhook success payload | API - Log Audit Trail | Respond - Success |  |
| Respond - Success | Respond to Webhook | Return success response to caller | Format Success Response | — |  |
| Error Trigger | Error Trigger | Catch failed executions | — | Format Error |  |
| Format Error | Code | Shape error into Slack message | Error Trigger | Slack - Error Alert |  |
| Slack - Error Alert | Slack | Post error notification to Slack | Format Error | — |  |

> Sticky notes in this workflow contain no text content, so the **Sticky Note** column is blank.

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger (Webhook)**
   1. Add node: **Webhook** → name it **Webhook - Inventory Movement**.  
   2. Choose method (commonly **POST**) and set a path (e.g., `/inventory/movement`).  
   3. Set **Response mode** to **Using “Respond to Webhook” node** (recommended, since the workflow includes Respond nodes).

2) **Validate the inbound payload**
   1. Add node: **Code** → **Validate Movement**.  
   2. Implement validation logic that outputs a clear flag, e.g. `valid: true/false`, and optionally `errors: []`.  
   3. Connect: **Webhook - Inventory Movement → Validate Movement**.

3) **Branch valid vs invalid**
   1. Add node: **IF** → **IF - Valid Movement**.  
   2. Configure condition to check your validation flag (example): `{{$json.valid === true}}`.  
   3. Connect: **Validate Movement → IF - Valid Movement**.

4) **Invalid path response**
   1. Add node: **Code** → **Format Validation Error**; create a response object (and optionally a status code field).  
   2. Add node: **Respond to Webhook** → **Respond - Validation Error**; configure to return JSON from previous node and set HTTP status (e.g., 400).  
   3. Connect: **IF - Valid Movement (false) → Format Validation Error → Respond - Validation Error**.

5) **Prepare and record movement (valid path)**
   1. Add node: **Code** → **Prepare API Body**; map inbound movement into the external API schema.  
   2. Add node: **HTTP Request** → **API - Record Movement**:
      - Set URL + method (usually POST)
      - Configure authentication (API key/OAuth2 as required)
      - Set body to the output of **Prepare API Body**
   3. Connect: **IF - Valid Movement (true) → Prepare API Body → API - Record Movement**.

6) **Fetch current stock**
   1. Add node: **HTTP Request** → **API - Get Current Stock**:
      - Configure endpoint/method
      - Pass SKU/location identifiers from the movement
   2. Connect: **API - Record Movement → API - Get Current Stock**.

7) **Compute severity and route alerts**
   1. Add node: **Code** → **Check Stock Levels**; compute `severity` and any message context.  
   2. Add node: **Switch** → **Alert Router**:
      - Switch value: `{{$json.severity}}`
      - Create 4 cases: `critical`, `urgent`, `warning`, default/none
   3. Connect: **API - Get Current Stock → Check Stock Levels → Alert Router**.

8) **Critical alert branch (Slack then Email)**
   1. Add node: **Code** → **Format Critical Alert**; build Slack text/blocks and email subject/body fields as needed.
   2. Add node: **Slack** → **Slack - Critical Alert**; configure Slack credentials, channel, and message from previous node.
   3. Add node: **Gmail** → **Email - Critical Alert**; configure Gmail OAuth2 credentials, recipients, subject/body.
   4. Connect: **Alert Router (critical) → Format Critical Alert → Slack - Critical Alert → Email - Critical Alert**.

9) **Urgent alert branch (Slack)**
   1. Add **Code** → **Format Urgent Alert**; then **Slack** → **Slack - Urgent Alert**.
   2. Connect: **Alert Router (urgent) → Format Urgent Alert → Slack - Urgent Alert**.

10) **Warning alert branch (Slack)**
   1. Add **Code** → **Format Warning Alert**; then **Slack** → **Slack - Warning Alert**.
   2. Connect: **Alert Router (warning) → Format Warning Alert → Slack - Warning Alert**.

11) **No alert branch**
   1. Add node: **NoOp** → **No Alert Needed**.
   2. Connect: **Alert Router (none/default) → No Alert Needed**.

12) **Audit trail and webhook success response**
   1. Add node: **Code** → **Prepare Audit Entry**; normalize inputs from all branches (critical/urgent/warning/none) into one audit schema.
   2. Connect the end of each branch to **Prepare Audit Entry**:
      - **Email - Critical Alert → Prepare Audit Entry**
      - **Slack - Urgent Alert → Prepare Audit Entry**
      - **Slack - Warning Alert → Prepare Audit Entry**
      - **No Alert Needed → Prepare Audit Entry**
   3. Add node: **HTTP Request** → **API - Log Audit Trail**; configure endpoint/auth/body.
   4. Add node: **Code** → **Format Success Response**; build `{success:true, ...}` payload.
   5. Add node: **Respond to Webhook** → **Respond - Success**; return JSON and set status code 200.
   6. Connect: **Prepare Audit Entry → API - Log Audit Trail → Format Success Response → Respond - Success**.

13) **Global error workflow path**
   1. Add node: **Error Trigger** → **Error Trigger**.
   2. Add node: **Code** → **Format Error**; format error details for Slack.
   3. Add node: **Slack** → **Slack - Error Alert**; configure Slack credentials/channel.
   4. Connect: **Error Trigger → Format Error → Slack - Error Alert**.
   5. In n8n settings, ensure this workflow (or a dedicated one) is configured as the error workflow if that’s your intended behavior.

**Credentials to configure**
- **Slack:** Slack API credentials (or webhook-based configuration depending on node operation) with permission to post to target channel(s).
- **Gmail:** OAuth2 credentials with send email scope; verify from-address and quotas.
- **HTTP Request APIs:** API key/OAuth2/basic auth depending on your inventory and audit services.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist for layout (Overview, Movement Intake, Stock Update, Alert System, Audit & Response, Error Handling) but contain no text. | Visual grouping only |
| Template placeholders detected: webhookId is `YOUR_WEBHOOK_ID` for the main webhook. | Must be generated by n8n when creating the webhook node |
| Many nodes have empty parameters (validation logic, switch cases, API endpoints, message formats). | The workflow is a structural scaffold; implement business rules and integrations accordingly |