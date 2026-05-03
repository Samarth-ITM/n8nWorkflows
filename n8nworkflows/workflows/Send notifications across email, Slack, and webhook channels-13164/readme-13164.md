Send notifications across email, Slack, and webhook channels

https://n8nworkflows.xyz/workflows/send-notifications-across-email--slack--and-webhook-channels-13164


# Send notifications across email, Slack, and webhook channels

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send notifications across email, Slack, and webhook channels

This workflow exposes a **Webhook endpoint** to receive a notification request, **validates and normalizes** the payload, **builds message templates**, optionally **stores the notification**, **routes delivery** to one or more channels (Gmail, Slack, outbound webhook), **tracks delivery status**, updates a backing store, and finally **responds** to the original HTTP caller. A separate **Error Trigger** flow formats and posts errors to Slack.

### Logical Blocks
1.1 **Request Intake**  
- Webhook receives the request and passes it into validation.

1.2 **Validation & Early Error Response**  
- Validates request shape/required fields; if invalid returns HTTP 400.

1.3 **Message Preparation & Persistence**  
- Builds channel-specific templates and stores the notification (HTTP request to external store/API).

1.4 **Channel Determination & Delivery**  
- Computes which channels to send to; conditionally sends via Gmail / Slack / forwards to a webhook.

1.5 **Tracking, Status Update & HTTP Response**  
- Aggregates delivery outcomes, updates status in store/API, formats and returns HTTP 200 response.

1.6 **Global Error Handling**  
- Any workflow error triggers a Slack alert with formatted error content.

---

## 2. Block-by-Block Analysis

### 2.1 Request Intake
**Overview:** Accepts inbound HTTP requests containing notification details, starting the workflow.  
**Nodes Involved:** Receive Notification

#### Node: Receive Notification
- **Type / Role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point via HTTP.
- **Configuration (interpreted):**
  - Uses a webhook endpoint (ID placeholder: `YOUR_WEBHOOK_ID`).
  - No explicit method/path/options provided in the JSON snippet (defaults apply unless configured in UI).
- **Key variables/expressions:** None visible (node parameters empty).
- **Connections:**
  - **Output →** Validate Request
- **Version requirements:** TypeVersion **2** (Webhook node v2 behavior for responses/registration differs from v1).
- **Edge cases / failures:**
  - If configured for “Respond immediately” vs “Using Respond to Webhook”, mismatch can cause timeouts.
  - Missing authentication (if required) can allow unwanted requests; not shown here.
  - Large payloads can exceed instance limits depending on n8n/server config.

---

### 2.2 Validation & Early Error Response
**Overview:** Validates incoming payload; if invalid, returns a structured HTTP 400 without running delivery.  
**Nodes Involved:** Validate Request, Valid?, Validation Error, Respond 400

#### Node: Validate Request
- **Type / Role:** `Code` — validate schema, required fields, and normalize data.
- **Configuration (interpreted):**
  - Code content not included; expected to:
    - Parse incoming JSON/body
    - Validate fields (e.g., `message`, `channels`, recipients, webhook target, etc.)
    - Output a normalized object and/or a `valid` flag + error details
- **Key variables/expressions:** Not visible; likely sets fields consumed by the IF node.
- **Connections:**
  - **Input ←** Receive Notification
  - **Output →** Valid?
- **Version requirements:** TypeVersion **2** (Code node v2 uses the new JS sandbox/runtime behavior).
- **Edge cases / failures:**
  - Throws if payload is not JSON / unexpected structure → triggers global Error Trigger.
  - Expression assumptions (e.g., `items[0].json...`) can break on empty input.

#### Node: Valid?
- **Type / Role:** `IF` — branch based on validation outcome.
- **Configuration (interpreted):**
  - Condition not provided; typically checks something like `{{$json.valid === true}}`.
- **Connections:**
  - **True →** Build Templates
  - **False →** Validation Error
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - If the validation node doesn’t output expected flag/field, condition evaluates unexpectedly.

#### Node: Validation Error
- **Type / Role:** `Code` — builds a client-facing error payload for bad requests.
- **Configuration (interpreted):**
  - Likely constructs `{ error: "...", details: ... }`
- **Connections:**
  - **Input ←** Valid? (false)
  - **Output →** Respond 400
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - If error details are undefined, response may be unhelpful.

#### Node: Respond 400
- **Type / Role:** `Respond to Webhook` — returns HTTP error response.
- **Configuration (interpreted):**
  - Expected to send status code **400** with JSON body from “Validation Error”.
  - Exact status/body settings not shown (parameters empty).
- **Connections:**
  - **Input ←** Validation Error
  - No further outputs (terminal response).
- **Version requirements:** TypeVersion **1.1**.
- **Edge cases / failures:**
  - If the Webhook node is set to “Respond immediately”, this node may not be used correctly.
  - If status code not set explicitly, may default to 200.

---

### 2.3 Message Preparation & Persistence
**Overview:** Builds channel-ready message templates and stores an initial notification record externally before delivery.  
**Nodes Involved:** Build Templates, Store Notification

#### Node: Build Templates
- **Type / Role:** `Code` — produce email subject/body, Slack text/blocks, webhook payload, etc.
- **Configuration (interpreted):**
  - Code not provided; typical outputs:
    - `email: { to, subject, bodyHtml/bodyText }`
    - `slack: { channel, text/blocks }`
    - `webhook: { url, method, headers, body }`
    - `notificationId` or correlation key
- **Connections:**
  - **Input ←** Valid? (true)
  - **Output →** Store Notification
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - Template generation can fail on missing optional fields (e.g., recipient list empty).
  - HTML generation can create invalid markup; Gmail may sanitize.

#### Node: Store Notification
- **Type / Role:** `HTTP Request` — persist notification metadata to an external service (DB API, Airtable, internal endpoint, etc.).
- **Configuration (interpreted):**
  - Parameters not shown; expected configuration includes:
    - URL (store endpoint)
    - Method (often POST)
    - Body contains normalized notification + templates + initial status (e.g., `queued`)
- **Connections:**
  - **Input ←** Build Templates
  - **Output →** Determine Channels
- **Version requirements:** TypeVersion **4.2**.
- **Edge cases / failures:**
  - Auth failures (API key/OAuth), 4xx/5xx responses.
  - Timeout / retry behavior depending on node settings.
  - If store call fails, deliveries never happen; error will go to global error flow.

---

### 2.4 Channel Determination & Delivery
**Overview:** Decides which channels to deliver to, then conditionally sends via Gmail, Slack, and/or forwards to a webhook.  
**Nodes Involved:** Determine Channels, Email?, Send Email, Slack?, Send Slack, Webhook?, Forward Webhook

#### Node: Determine Channels
- **Type / Role:** `Code` — compute booleans/flags and channel configs for downstream IF nodes.
- **Configuration (interpreted):**
  - Likely sets fields like `sendEmail`, `sendSlack`, `sendWebhook` or prepares items for each path.
- **Connections:**
  - **Input ←** Store Notification
  - **Output →** Email?, Slack?, Webhook? (fan-out to three IF nodes)
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - If it outputs multiple items, IF nodes may execute multiple times unexpectedly.
  - Miscomputed flags can skip channels or send duplicates.

#### Node: Email?
- **Type / Role:** `IF` — gate for email delivery.
- **Configuration (interpreted):**
  - Condition not shown; typically checks `{{$json.channels.email === true}}` or similar.
- **Connections:**
  - **True →** Send Email
  - **False →** Track Status
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - “False → Track Status” means status tracking runs even when email not selected; ensure tracking logic accounts for “skipped”.

#### Node: Send Email
- **Type / Role:** `Gmail` — send email notification.
- **Configuration (interpreted):**
  - Node parameters not shown; typically sets:
    - To, Subject, Message (HTML/Text)
  - Requires Gmail OAuth2 credentials in n8n.
- **Connections:**
  - **Input ←** Email? (true)
  - **Output →** Track Status
- **Version requirements:** TypeVersion **2.1**.
- **Edge cases / failures:**
  - OAuth token expiry / missing scopes.
  - Gmail sending limits, invalid recipient addresses.
  - If templates not present, send will fail or send blank content.

#### Node: Slack?
- **Type / Role:** `IF` — gate for Slack delivery.
- **Configuration (interpreted):**
  - Condition not shown; checks Slack channel enabled.
- **Connections:**
  - **True →** Send Slack
  - **False →** Track Status
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - Same “skipped” tracking concern as Email?.

#### Node: Send Slack
- **Type / Role:** `Slack` — send message to Slack.
- **Configuration (interpreted):**
  - Parameters not shown; typically sets:
    - Channel and text/blocks
  - Requires Slack OAuth or bot token credentials.
- **Connections:**
  - **Input ←** Slack? (true)
  - **Output →** Track Status
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - Missing permissions (chat:write), invalid channel ID, rate limits.

#### Node: Webhook?
- **Type / Role:** `IF` — gate for outbound webhook forwarding.
- **Configuration (interpreted):**
  - Condition not shown; checks if webhook forwarding is enabled/configured.
- **Connections:**
  - **True →** Forward Webhook
  - **False →** Track Status
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - Must ensure target URL exists; otherwise Forward Webhook will fail.

#### Node: Forward Webhook
- **Type / Role:** `HTTP Request` — POST/PUT notification to an external webhook endpoint.
- **Configuration (interpreted):**
  - Parameters not shown; typically:
    - URL from payload (e.g., `{{$json.webhook.url}}`)
    - Method + headers + JSON body
- **Connections:**
  - **Input ←** Webhook? (true)
  - **Output →** Track Status
- **Version requirements:** TypeVersion **4.2**.
- **Edge cases / failures:**
  - SSRF risk if target URL is user-supplied; should validate/allowlist in Validate Request.
  - TLS/cert issues, timeouts, 4xx/5xx responses.

---

### 2.5 Tracking, Status Update & HTTP Response
**Overview:** Consolidates outcomes (sent/skipped/failed), updates the stored notification status, and returns a 200 response to the caller.  
**Nodes Involved:** Track Status, Update Status, Format Response, Respond 200

#### Node: Track Status
- **Type / Role:** `Code` — compile per-channel results and compute overall status.
- **Configuration (interpreted):**
  - Given it receives inputs from:
    - Send Email, Send Slack, Forward Webhook
    - and the “false” branches of Email?/Slack?/Webhook?
  - It likely:
    - Detects which path produced the item
    - Marks channel as `sent`, `skipped`, or `failed`
    - Adds timestamps, message IDs, response codes, etc.
- **Connections:**
  - **Inputs ←** Email?/Slack?/Webhook? (false paths), Send Email, Send Slack, Forward Webhook
  - **Output →** Update Status
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - Multiple incoming branches can produce multiple items; without merge/aggregation, you might update status multiple times.
  - If relying on node names or metadata to identify channel, changes break tracking.

#### Node: Update Status
- **Type / Role:** `HTTP Request` — update persisted record with delivery results.
- **Configuration (interpreted):**
  - Parameters not shown; expected:
    - PATCH/PUT to store by notification ID
    - Writes statuses, provider message IDs, error info
- **Connections:**
  - **Input ←** Track Status
  - **Output →** Format Response
- **Version requirements:** TypeVersion **4.2**.
- **Edge cases / failures:**
  - Partial failures: if status update fails after successful sends, caller may get an error (and you lose tracking).
  - Concurrency: multiple updates can overwrite each other if not handled.

#### Node: Format Response
- **Type / Role:** `Code` — create the HTTP 200 response body.
- **Configuration (interpreted):**
  - Typically returns: `{ id, status, channels: {...}, timestamps }`
- **Connections:**
  - **Input ←** Update Status
  - **Output →** Respond 200
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - If Update Status response differs from expected schema, formatting can fail.

#### Node: Respond 200
- **Type / Role:** `Respond to Webhook` — returns success response to caller.
- **Configuration (interpreted):**
  - Expected to respond with status **200** and JSON body from Format Response.
- **Connections:**
  - **Input ←** Format Response
- **Version requirements:** TypeVersion **1.1**.
- **Edge cases / failures:**
  - As with Respond 400, must be compatible with Webhook node response mode and timeout.

---

### 2.6 Global Error Handling
**Overview:** Captures any unhandled workflow error and posts an alert to Slack.  
**Nodes Involved:** Error Trigger, Format Error, Error Alert

#### Node: Error Trigger
- **Type / Role:** `Error Trigger` — starts when this workflow errors.
- **Configuration (interpreted):** Default error trigger behavior; no parameters.
- **Connections:**
  - **Output →** Format Error
- **Version requirements:** TypeVersion **1**.
- **Edge cases / failures:**
  - Only triggers for workflow execution errors; channel-level handled errors may not reach here if suppressed.

#### Node: Format Error
- **Type / Role:** `Code` — format error payload for Slack alerting.
- **Configuration (interpreted):**
  - Likely extracts error message, stack, node name, execution URL, input snippet.
- **Connections:**
  - **Input ←** Error Trigger
  - **Output →** Error Alert
- **Version requirements:** TypeVersion **2**.
- **Edge cases / failures:**
  - If it tries to stringify circular structures, it can throw (causing missing alerts).

#### Node: Error Alert
- **Type / Role:** `Slack` — posts error notification to Slack.
- **Configuration (interpreted):**
  - Parameters not shown; typically posts to a fixed channel (ops-alerts).
  - Requires Slack credentials.
- **Connections:**
  - **Input ←** Format Error
- **Version requirements:** TypeVersion **2.2**.
- **Edge cases / failures:**
  - If Slack is down / rate-limited, you lose error visibility.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Comment/section header | — | — |  |
| Request Intake | Sticky Note | Comment/section header | — | — |  |
| Message Preparation | Sticky Note | Comment/section header | — | — |  |
| Channel Delivery | Sticky Note | Comment/section header | — | — |  |
| Tracking & Response | Sticky Note | Comment/section header | — | — |  |
| Error Handling | Sticky Note | Comment/section header | — | — |  |
| Receive Notification | Webhook | Inbound notification endpoint | — | Validate Request |  |
| Validate Request | Code | Validate/normalize payload | Receive Notification | Valid? |  |
| Valid? | IF | Branch on validation | Validate Request | Build Templates; Validation Error |  |
| Validation Error | Code | Build 400 response body | Valid? (false) | Respond 400 |  |
| Respond 400 | Respond to Webhook | Return bad request response | Validation Error | — |  |
| Build Templates | Code | Create channel templates | Valid? (true) | Store Notification |  |
| Store Notification | HTTP Request | Persist notification record | Build Templates | Determine Channels |  |
| Determine Channels | Code | Decide target channels | Store Notification | Email?; Slack?; Webhook? |  |
| Email? | IF | Gate email send | Determine Channels | Send Email; Track Status |  |
| Send Email | Gmail | Deliver email | Email? (true) | Track Status |  |
| Slack? | IF | Gate Slack send | Determine Channels | Send Slack; Track Status |  |
| Send Slack | Slack | Deliver Slack message | Slack? (true) | Track Status |  |
| Webhook? | IF | Gate webhook forward | Determine Channels | Forward Webhook; Track Status |  |
| Forward Webhook | HTTP Request | Forward to external webhook | Webhook? (true) | Track Status |  |
| Track Status | Code | Compute per-channel/overall status | Email? (false); Slack? (false); Webhook? (false); Send Email; Send Slack; Forward Webhook | Update Status |  |
| Update Status | HTTP Request | Update stored record | Track Status | Format Response |  |
| Format Response | Code | Prepare 200 response body | Update Status | Respond 200 |  |
| Respond 200 | Respond to Webhook | Return success response | Format Response | — |  |
| Error Trigger | Error Trigger | Catch execution errors | — | Format Error |  |
| Format Error | Code | Format error alert payload | Error Trigger | Error Alert |  |
| Error Alert | Slack | Send error alert to Slack | Format Error | — |  |

*Note:* All sticky notes have empty content in the provided workflow export, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **Send notifications across email, Slack, and webhook channels**

2) **Add entry node: Webhook**
- Add node: **Webhook** → rename **Receive Notification**
- Configure:
  - HTTP Method: (choose your expected method, commonly **POST**)
  - Path: (e.g., `/notify`)
  - Response mode: **Using “Respond to Webhook” node** (so Respond 200/400 work)
  - Optional but recommended: Authentication (Header/API key, Basic, or JWT), and request size limits.

3) **Add validation code**
- Add node: **Code** → rename **Validate Request**
- Connect: Receive Notification → Validate Request
- Implement validation logic (example expectations):
  - Required: `message` (string) and at least one channel enabled.
  - Optional structures:
    - `email.to` (string or array), `email.subject`
    - `slack.channel` and `slack.text` or derive from `message`
    - `webhook.url` and optional method/headers
  - Output a normalized object and a boolean like `valid`.

4) **Add IF for validation**
- Add node: **IF** → rename **Valid?**
- Connect: Validate Request → Valid?
- Configure condition (example):
  - Boolean check: `{{$json.valid}}` is true

5) **Invalid branch response**
- Add node: **Code** → rename **Validation Error**
- Connect: Valid? (false) → Validation Error
- Build a response payload `{ error: "...", details: ... }`
- Add node: **Respond to Webhook** → rename **Respond 400**
- Connect: Validation Error → Respond 400
- Configure Respond 400:
  - Status code: **400**
  - Response body: JSON from input

6) **Template building**
- Add node: **Code** → rename **Build Templates**
- Connect: Valid? (true) → Build Templates
- Generate channel-specific fields for Gmail/Slack/Webhook plus a correlation ID.

7) **Store initial notification**
- Add node: **HTTP Request** → rename **Store Notification**
- Connect: Build Templates → Store Notification
- Configure:
  - Method: POST (typical)
  - URL: your storage endpoint
  - Authentication: as required (API key/OAuth2)
  - Send JSON body: include templates + initial status (e.g., `queued`)
  - Ensure you capture returned `notificationId` (or equivalent)

8) **Determine channels**
- Add node: **Code** → rename **Determine Channels**
- Connect: Store Notification → Determine Channels
- Output flags such as `sendEmail`, `sendSlack`, `sendWebhook`, plus per-channel config.

9) **Email path**
- Add node: **IF** → rename **Email?**
- Connect: Determine Channels → Email?
- Condition: `{{$json.sendEmail === true}}`
- Add node: **Gmail** → rename **Send Email**
- Connect: Email? (true) → Send Email
- Configure Gmail node:
  - Resource/Operation: Send Email
  - To/Subject/Message: map from templates (expressions)
  - Credentials: **Gmail OAuth2** in n8n with appropriate scopes

10) **Slack path**
- Add node: **IF** → rename **Slack?**
- Connect: Determine Channels → Slack?
- Condition: `{{$json.sendSlack === true}}`
- Add node: **Slack** → rename **Send Slack**
- Connect: Slack? (true) → Send Slack
- Configure Slack node:
  - Operation: Post message
  - Channel/Text (or Blocks): map from templates
  - Credentials: Slack app/bot token with **chat:write**

11) **Outbound webhook path**
- Add node: **IF** → rename **Webhook?**
- Connect: Determine Channels → Webhook?
- Condition: `{{$json.sendWebhook === true}}`
- Add node: **HTTP Request** → rename **Forward Webhook**
- Connect: Webhook? (true) → Forward Webhook
- Configure:
  - URL from input (expression) or from a validated allowlist
  - Method/headers/body from templates

12) **Tracking and fan-in**
- Add node: **Code** → rename **Track Status**
- Connect the following into **Track Status**:
  - Send Email → Track Status
  - Send Slack → Track Status
  - Forward Webhook → Track Status
  - Email? (false) → Track Status
  - Slack? (false) → Track Status
  - Webhook? (false) → Track Status
- In Track Status code, compute:
  - For each channel: `sent` / `skipped` / `failed`
  - Provider metadata (gmail messageId, slack ts, webhook http status)
  - Overall status

13) **Update stored status**
- Add node: **HTTP Request** → rename **Update Status**
- Connect: Track Status → Update Status
- Configure:
  - PATCH/PUT to your storage endpoint using `notificationId`
  - Write computed channel results and overall status

14) **Format and respond**
- Add node: **Code** → rename **Format Response**
- Connect: Update Status → Format Response
- Build response JSON for the original caller (id + channel results).
- Add node: **Respond to Webhook** → rename **Respond 200**
- Connect: Format Response → Respond 200
- Configure:
  - Status code: **200**
  - Response body: JSON

15) **Global error handling**
- Add node: **Error Trigger** → rename **Error Trigger**
- Add node: **Code** → rename **Format Error**
- Connect: Error Trigger → Format Error
- Add node: **Slack** → rename **Error Alert**
- Connect: Format Error → Error Alert
- Configure Error Alert to post to an ops/error channel with execution + node + error details.
- Credentials: same Slack credential set (or a dedicated one).

16) **Add sticky notes (optional)**
- Add sticky notes titled:
  - Overview, Request Intake, Message Preparation, Channel Delivery, Tracking & Response, Error Handling
- (In the provided workflow they are present but contain no text.)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky note sections exist but their contents are empty in the provided workflow export. | Visual grouping only (no embedded guidance/links). |
| Consider adding an allowlist for outbound webhook targets to prevent SSRF when `webhook.url` is user-supplied. | Applies to Validate Request + Forward Webhook. |
| Track Status currently receives multiple branches without an explicit Merge node; ensure the code handles multiple executions/items correctly to avoid overwriting status. | Applies to Track Status + Update Status. |