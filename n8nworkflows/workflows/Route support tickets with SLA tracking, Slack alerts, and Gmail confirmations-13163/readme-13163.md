Route support tickets with SLA tracking, Slack alerts, and Gmail confirmations

https://n8nworkflows.xyz/workflows/route-support-tickets-with-sla-tracking--slack-alerts--and-gmail-confirmations-13163


# Route support tickets with SLA tracking, Slack alerts, and Gmail confirmations

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Route support tickets with SLA tracking, Slack alerts, and Gmail confirmations

**Purpose:**  
This workflow receives new support tickets via a webhook, normalizes and validates the payload, categorizes the ticket and assigns an SLA, checks for duplicates, creates the ticket in an external system, routes notifications to Slack based on priority, sends a confirmation email via Gmail, and finally responds to the webhook caller. It also includes centralized error alerting to Slack via an Error Trigger.

**Target use cases:**
- Fronting a support intake form (website, help widget, internal tool) with an n8n webhook
- Enforcing consistent ticket schema and validation at ingestion time
- Priority-based routing + SLA assignment
- Avoiding duplicate ticket creation
- Team notifications (Slack) and requester confirmations (Gmail)
- Ops visibility via Slack error alerts

### Logical blocks
**1.1 Input Reception & Normalization**  
Webhook intake → normalize fields → validate required values.

**1.2 Categorization, Deduplication & SLA Assignment**  
Categorize ticket → check duplicates via HTTP → branch on duplicate → assign SLA.

**1.3 Ticket Creation & Priority Routing**  
Create ticket in external system → switch by priority → format message payload.

**1.4 Notifications & Webhook Response**  
Slack notification → Gmail confirmation → format success response → respond 200.

**1.5 Validation Failure Response**  
Format validation error → respond 400.

**1.6 Global Error Handling**  
Error Trigger → format error message → Slack alert.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Normalization
**Overview:** Receives inbound ticket data, converts it into a consistent internal structure, and checks whether it meets minimum validity requirements before proceeding.

**Nodes involved:** `New Ticket`, `Normalize Data`, `Valid Ticket?`

#### Node: New Ticket
- **Type / role:** Webhook (trigger). Entry point for new tickets.
- **Configuration (interpreted):**
  - Uses a webhook endpoint (webhookId placeholder: `YOUR_WEBHOOK_ID`).
  - HTTP method / path are not specified in JSON (defaults depend on n8n UI settings).
- **Inputs/outputs:**
  - **Output:** to `Normalize Data`.
- **Version:** Webhook node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Unexpected payload shape (missing fields) leading to downstream expression/code errors.
  - If “Respond to Webhook” is enabled/required and not properly configured, callers may time out.
  - Large payloads / attachment payloads may hit size limits.

#### Node: Normalize Data
- **Type / role:** Code node. Transforms inbound JSON into a normalized schema.
- **Configuration (interpreted):**
  - Parameters are empty in the JSON export, meaning the actual JS code is not included here.
  - Expected to: map incoming fields to canonical ones (e.g., requester email, subject, description, priority), sanitize types, set defaults.
- **Key variables/expressions:** Not visible (code omitted).
- **Inputs/outputs:**
  - **Input:** `New Ticket`
  - **Output:** `Valid Ticket?`
- **Version:** Code node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Runtime exceptions if expected properties are absent.
  - Producing an unexpected output structure that breaks `Valid Ticket?`.

#### Node: Valid Ticket?
- **Type / role:** IF node. Gatekeeper validation.
- **Configuration (interpreted):**
  - Conditions are not included; assumed to check required fields (e.g., `email`, `subject`, `message`) and possibly valid `priority`.
- **Inputs/outputs:**
  - **Input:** `Normalize Data`
  - **True output (index 0):** to `Categorize Ticket`
  - **False output (index 1):** to `Validation Error`
- **Version:** IF node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Misconfigured conditions causing false negatives/positives.
  - If normalization output changes, conditions may reference nonexistent paths.

---

### 2.2 Validation Failure Response
**Overview:** If the inbound payload is invalid, formats a client-friendly error and responds with HTTP 400.

**Nodes involved:** `Validation Error`, `Respond 400`

#### Node: Validation Error
- **Type / role:** Code node. Builds an error payload for the webhook caller.
- **Configuration (interpreted):**
  - Code not present in export; likely constructs JSON with validation messages and missing fields list.
- **Inputs/outputs:**
  - **Input:** `Valid Ticket?` (false branch)
  - **Output:** `Respond 400`
- **Version:** Code node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - If it expects certain validation context not produced upstream.
  - Improperly structured output could cause `Respond 400` to return empty/incorrect body.

#### Node: Respond 400
- **Type / role:** Respond to Webhook. Terminates request with error status.
- **Configuration (interpreted):**
  - Parameters not shown; expected to set **HTTP status code 400** and return the formatted error body.
- **Inputs/outputs:**
  - **Input:** `Validation Error`
  - **Output:** none (terminating node)
- **Version:** RespondToWebhook `typeVersion: 1.1`.
- **Potential failures / edge cases:**
  - If configured to respond with data from a field that does not exist.
  - If webhook is set to “Response: Last node”, ensure this node is reached on validation failures.

---

### 2.3 Categorization, Deduplication & SLA Assignment
**Overview:** Categorizes the ticket, checks whether a similar ticket already exists via HTTP request, and assigns SLA parameters if it is not a duplicate.

**Nodes involved:** `Categorize Ticket`, `Check Duplicate`, `Not Duplicate?`, `Assign SLA`

#### Node: Categorize Ticket
- **Type / role:** Code node. Determines category (and likely priority hints/tags).
- **Configuration (interpreted):**
  - Code not included; typically uses keyword matching or rules to set fields like `category`, `priority`, `team`, `tags`.
- **Inputs/outputs:**
  - **Input:** `Valid Ticket?` (true branch)
  - **Output:** `Check Duplicate`
- **Version:** Code node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Inconsistent classification leading to incorrect routing later.
  - Missing priority causing `Route by Priority` switch mismatch.

#### Node: Check Duplicate
- **Type / role:** HTTP Request. Queries an external system to detect duplicates.
- **Configuration (interpreted):**
  - Parameters omitted; expected to call a ticketing DB/search endpoint (e.g., search by requester email + subject hash).
- **Key data dependencies:** Likely uses fields from normalized/categorized ticket (e.g., `{{$json.email}}`, `{{$json.subject}}`), but expressions are not visible.
- **Inputs/outputs:**
  - **Input:** `Categorize Ticket`
  - **Output:** `Not Duplicate?`
- **Version:** HTTP Request `typeVersion: 4.2`.
- **Potential failures / edge cases:**
  - Auth errors (401/403), rate limits (429), timeouts.
  - Non-JSON response parsing issues.
  - Duplicate logic ambiguity: multiple “similar” results.

#### Node: Not Duplicate?
- **Type / role:** IF node. Branches between creating a new ticket vs returning success without creating.
- **Configuration (interpreted):**
  - Conditions not shown; assumed to check HTTP response (e.g., `total == 0`).
- **Inputs/outputs:**
  - **Input:** `Check Duplicate`
  - **True output:** `Assign SLA`
  - **False output:** `Format Success` (i.e., duplicate found → do not create; return success)
- **Version:** IF node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Incorrect condition mapping to the HTTP response shape.
  - If upstream returns error payload, condition could evaluate unexpectedly.

#### Node: Assign SLA
- **Type / role:** Code node. Computes SLA deadlines/targets from category/priority.
- **Configuration (interpreted):**
  - Code not present; likely sets `sla` fields such as `first_response_due_at`, `resolve_due_at`, or durations.
- **Inputs/outputs:**
  - **Input:** `Not Duplicate?` (true branch)
  - **Output:** `Create Ticket`
- **Version:** Code node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Timezone handling (UTC vs local).
  - Business-hours SLA vs absolute-time SLA not accounted for.
  - Missing/unknown priority leading to undefined SLA.

---

### 2.4 Ticket Creation & Priority Routing
**Overview:** Creates the ticket in an external system, then routes formatting logic based on priority to prepare the Slack notification payload.

**Nodes involved:** `Create Ticket`, `Route by Priority`, `Format Critical`, `Format High`, `Format Medium`, `Format Low`

#### Node: Create Ticket
- **Type / role:** HTTP Request. Creates the ticket record in an external system.
- **Configuration (interpreted):**
  - Parameters omitted; expected to POST ticket data (normalized + category + SLA) to a helpdesk/DB API.
- **Inputs/outputs:**
  - **Input:** `Assign SLA`
  - **Output:** `Route by Priority`
- **Version:** HTTP Request `typeVersion: 4.2`.
- **Potential failures / edge cases:**
  - Auth/rate limit/timeouts.
  - Partial success: ticket created but response not parseable.
  - API schema mismatch (missing required fields).

#### Node: Route by Priority
- **Type / role:** Switch node. Branches by priority value.
- **Configuration (interpreted):**
  - Rules not included; expected cases: `Critical`, `High`, `Medium`, `Low`.
- **Inputs/outputs:**
  - **Input:** `Create Ticket`
  - **Outputs (in order):**
    1. to `Format Critical`
    2. to `Format High`
    3. to `Format Medium`
    4. to `Format Low`
- **Version:** Switch node `typeVersion: 3`.
- **Potential failures / edge cases:**
  - Priority value not matching any case → no output (ticket created but no notification/email/response).
  - Case sensitivity (“high” vs “High”).

#### Node: Format Critical / Format High / Format Medium / Format Low
- **Type / role:** Code nodes. Build Slack message payloads tailored to priority.
- **Configuration (interpreted):**
  - Code not present; likely sets message text, channel, mentions, urgency formatting, includes ticket ID/URL.
- **Inputs/outputs:**
  - Each receives from `Route by Priority` (its matching branch)
  - Each outputs to `Notify Team`
- **Version:** Code node `typeVersion: 2` (all).
- **Potential failures / edge cases:**
  - Missing fields from ticket creation response (e.g., ticket URL/id).
  - Slack payload shape errors if constructing blocks/attachments incorrectly.

---

### 2.5 Notifications & Webhook Response
**Overview:** Sends the team notification in Slack, sends requester confirmation via Gmail, then responds to the original webhook caller with a success payload.

**Nodes involved:** `Notify Team`, `Send Confirmation`, `Format Success`, `Respond 200`

#### Node: Notify Team
- **Type / role:** Slack node. Posts a message/alert to Slack.
- **Configuration (interpreted):**
  - Uses Slack credentials (not shown). A `webhookId` is present in export metadata but actual auth is via credentials in n8n.
  - Operation (post message, channel, blocks) not visible.
- **Inputs/outputs:**
  - **Input:** from any of the format nodes (Critical/High/Medium/Low)
  - **Output:** `Send Confirmation`
- **Version:** Slack node `typeVersion: 2.2`.
- **Potential failures / edge cases:**
  - Slack credential revoked.
  - Channel not found / missing permissions.
  - Rate limiting for bursts of tickets.

#### Node: Send Confirmation
- **Type / role:** Gmail node. Sends confirmation email to requester.
- **Configuration (interpreted):**
  - Gmail OAuth2 credentials required.
  - Email fields (to/subject/body) are not shown; expected to use normalized requester email and ticket details.
- **Inputs/outputs:**
  - **Input:** `Notify Team`
  - **Output:** `Format Success`
- **Version:** Gmail node `typeVersion: 2.1`.
- **Potential failures / edge cases:**
  - OAuth token expired/invalid.
  - Sending limits, blocked recipient, malformed email address.
  - If requester email missing despite earlier validation, this node fails.

#### Node: Format Success
- **Type / role:** Code node. Creates the success response body for webhook caller.
- **Configuration (interpreted):**
  - Code not included; likely returns JSON including `status`, `ticketId`, `priority`, `sla`.
  - Also used when a duplicate is found (from `Not Duplicate?` false branch).
- **Inputs/outputs:**
  - **Input:** from `Send Confirmation` OR from `Not Duplicate?` false branch
  - **Output:** `Respond 200`
- **Version:** Code node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Must handle two contexts:
    - Newly created ticket path (ticket ID present)
    - Duplicate path (ticket ID may be existing or absent)
  - If it assumes fields that only exist after `Create Ticket`, duplicate path could break unless guarded.

#### Node: Respond 200
- **Type / role:** Respond to Webhook. Returns success to caller.
- **Configuration (interpreted):**
  - Parameters not shown; expected HTTP 200 with body from `Format Success`.
- **Inputs/outputs:**
  - **Input:** `Format Success`
  - **Output:** none (terminating node)
- **Version:** RespondToWebhook `typeVersion: 1.1`.
- **Potential failures / edge cases:**
  - If webhook expects response earlier and execution takes too long (Slack/Gmail delay), caller may time out.
  - If configured to return a field that does not exist.

---

### 2.6 Global Error Handling
**Overview:** Captures any workflow execution error and pushes a formatted alert to Slack for operational visibility.

**Nodes involved:** `Error Trigger`, `Format Error`, `Error Alert`

#### Node: Error Trigger
- **Type / role:** Error Trigger. Runs when the workflow errors.
- **Configuration (interpreted):**
  - Default error trigger behavior (no parameters).
- **Inputs/outputs:**
  - **Output:** `Format Error`
- **Version:** `typeVersion: 1`.
- **Potential failures / edge cases:**
  - Only triggers for workflow errors; logical “invalid ticket” is handled separately (400 path).
  - If Slack alert also fails, errors may be silent unless n8n-level alerts are configured.

#### Node: Format Error
- **Type / role:** Code node. Builds a Slack-friendly error summary.
- **Configuration (interpreted):**
  - Code not included; likely extracts error message, node name, execution id, timestamp, and input excerpt.
- **Inputs/outputs:**
  - **Input:** `Error Trigger`
  - **Output:** `Error Alert`
- **Version:** Code node `typeVersion: 2`.
- **Potential failures / edge cases:**
  - Error object shape differences between n8n versions.
  - Oversized payloads if including full input data.

#### Node: Error Alert
- **Type / role:** Slack node. Posts error alert to Slack.
- **Configuration (interpreted):**
  - Slack credentials required; operation details not visible.
- **Inputs/outputs:**
  - **Input:** `Format Error`
  - **Output:** none
- **Version:** Slack node `typeVersion: 2.2`.
- **Potential failures / edge cases:**
  - Same Slack failures as `Notify Team` (auth/permissions/rate limit).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Canvas section header | — | — |  |
| Ticket Intake | Sticky Note | Canvas section header | — | — |  |
| Categorization & SLA | Sticky Note | Canvas section header | — | — |  |
| Database & Notifications | Sticky Note | Canvas section header | — | — |  |
| Response | Sticky Note | Canvas section header | — | — |  |
| Error Handling | Sticky Note | Canvas section header | — | — |  |
| New Ticket | Webhook | Ticket intake endpoint | — | Normalize Data |  |
| Normalize Data | Code | Normalize inbound payload | New Ticket | Valid Ticket? |  |
| Valid Ticket? | IF | Validate required fields | Normalize Data | Categorize Ticket; Validation Error |  |
| Validation Error | Code | Build 400 response body | Valid Ticket? | Respond 400 |  |
| Respond 400 | Respond to Webhook | Return HTTP 400 | Validation Error | — |  |
| Categorize Ticket | Code | Categorize / enrich ticket | Valid Ticket? | Check Duplicate |  |
| Check Duplicate | HTTP Request | Search duplicates in external system | Categorize Ticket | Not Duplicate? |  |
| Not Duplicate? | IF | Branch on duplicate result | Check Duplicate | Assign SLA; Format Success |  |
| Assign SLA | Code | Compute SLA targets | Not Duplicate? | Create Ticket |  |
| Create Ticket | HTTP Request | Create ticket in external system | Assign SLA | Route by Priority |  |
| Route by Priority | Switch | Route formatting by priority | Create Ticket | Format Critical; Format High; Format Medium; Format Low |  |
| Format Critical | Code | Build Slack payload (critical) | Route by Priority | Notify Team |  |
| Format High | Code | Build Slack payload (high) | Route by Priority | Notify Team |  |
| Format Medium | Code | Build Slack payload (medium) | Route by Priority | Notify Team |  |
| Format Low | Code | Build Slack payload (low) | Route by Priority | Notify Team |  |
| Notify Team | Slack | Post notification to Slack | Format Critical/High/Medium/Low | Send Confirmation |  |
| Send Confirmation | Gmail | Email confirmation to requester | Notify Team | Format Success |  |
| Format Success | Code | Build success response body | Send Confirmation; Not Duplicate? | Respond 200 |  |
| Respond 200 | Respond to Webhook | Return HTTP 200 | Format Success | — |  |
| Error Trigger | Error Trigger | Catch workflow errors | — | Format Error |  |
| Format Error | Code | Build Slack error payload | Error Trigger | Error Alert |  |
| Error Alert | Slack | Post error alert to Slack | Format Error | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n. Name it: *Route support tickets with SLA tracking, Slack alerts, and Gmail confirmations*.

2) **Add Webhook trigger**
   - Node: **Webhook** named `New Ticket`
   - Configure:
     - Set a path like `/support/ticket/new` (or your preferred path)
     - Choose method `POST`
     - Response mode: ensure it waits for a **Respond to Webhook** node (recommended)
   - Save to obtain the production URL.

3) **Add Normalize step**
   - Node: **Code** named `Normalize Data`
   - Implement JS to output a normalized object (typical fields):
     - `requesterEmail`, `requesterName`, `subject`, `description`, `priority` (default), `source`, `createdAt`
   - Connect: `New Ticket` → `Normalize Data`.

4) **Add Validation gate**
   - Node: **IF** named `Valid Ticket?`
   - Conditions (example expectations):
     - `requesterEmail` exists and matches basic email pattern
     - `subject` is not empty
     - `description` is not empty
   - Connect: `Normalize Data` → `Valid Ticket?`.

5) **Invalid path: build 400 response**
   - Node: **Code** named `Validation Error`
     - Build `{ "error": "Validation failed", "details": [...] }`
   - Node: **Respond to Webhook** named `Respond 400`
     - Status code: `400`
     - Response body: from `Validation Error` output
   - Connect: `Valid Ticket?` (false) → `Validation Error` → `Respond 400`.

6) **Categorization**
   - Node: **Code** named `Categorize Ticket`
     - Set `category`, normalize `priority` into one of: `Critical|High|Medium|Low`
     - Optionally set `tags`, `routingTeam`
   - Connect: `Valid Ticket?` (true) → `Categorize Ticket`.

7) **Duplicate check**
   - Node: **HTTP Request** named `Check Duplicate`
     - Configure to call your search endpoint (examples: helpdesk search API / database query service)
     - Use expressions for query parameters/body from the ticket (email + subject hash, etc.)
     - Authentication: set credentials (API key/OAuth2) as needed
   - Node: **IF** named `Not Duplicate?`
     - Condition based on HTTP response (example: `results.length === 0`)
   - Connect: `Categorize Ticket` → `Check Duplicate` → `Not Duplicate?`.

8) **Duplicate found path**
   - Connect: `Not Duplicate?` (false) → `Format Success` (created later in step 12)
   - In `Format Success`, ensure you handle duplicate scenario (e.g., set `duplicate: true`).

9) **Assign SLA**
   - Node: **Code** named `Assign SLA`
     - Compute SLA fields based on priority/category (e.g., response within X minutes/hours)
     - Output should include SLA metadata and pass through ticket fields
   - Connect: `Not Duplicate?` (true) → `Assign SLA`.

10) **Create ticket**
   - Node: **HTTP Request** named `Create Ticket`
     - Method: `POST`
     - URL: your ticket system endpoint
     - Body: mapped from `Assign SLA` output
     - Auth: configure credentials
   - Connect: `Assign SLA` → `Create Ticket`.

11) **Route by priority**
   - Node: **Switch** named `Route by Priority`
     - Value to evaluate: your priority field (e.g., `{{$json.priority}}`)
     - Add 4 cases: `Critical`, `High`, `Medium`, `Low`
   - Add 4 **Code** nodes:
     - `Format Critical`, `Format High`, `Format Medium`, `Format Low`
     - Each outputs a Slack message payload appropriate to urgency (e.g., mentions for Critical)
   - Connect:
     - `Create Ticket` → `Route by Priority`
     - Each switch output → corresponding `Format ...` node

12) **Slack notify**
   - Node: **Slack** named `Notify Team`
     - Credential: Slack OAuth2 (recommended) or Slack app token depending on node operation
     - Operation: “Post message” (or equivalent)
     - Channel: your support channel
     - Message/blocks: from each `Format ...` node output
   - Connect each `Format ...` node → `Notify Team`.

13) **Gmail confirmation**
   - Node: **Gmail** named `Send Confirmation`
     - Credential: Gmail OAuth2
     - Operation: Send Email
     - To: requester email field
     - Subject/body: include ticket id/summary (from `Create Ticket` response)
   - Connect: `Notify Team` → `Send Confirmation`.

14) **Format success + respond**
   - Node: **Code** named `Format Success`
     - Return response JSON (include `ticketId` if created, `duplicate` flag if not)
   - Node: **Respond to Webhook** named `Respond 200`
     - Status code: `200`
     - Body: from `Format Success`
   - Connect: `Send Confirmation` → `Format Success` → `Respond 200`.
   - Also connect: `Not Duplicate?` (false) → `Format Success` (from step 8).

15) **Global error handling**
   - Node: **Error Trigger** named `Error Trigger`
   - Node: **Code** named `Format Error` (extract node name, message, execution URL/id)
   - Node: **Slack** named `Error Alert` (post to ops/errors channel)
   - Connect: `Error Trigger` → `Format Error` → `Error Alert`.

16) **Credentials checklist**
   - Slack: configure Slack credentials for `Notify Team` and `Error Alert` (can be same credential).
   - Gmail: configure OAuth2 credential for `Send Confirmation`.
   - HTTP Request nodes: configure required auth (API keys/OAuth2/headers) for `Check Duplicate` and `Create Ticket`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist as section headers: Overview, Ticket Intake, Categorization & SLA, Database & Notifications, Response, Error Handling. (They contain no additional text in the exported JSON.) | Canvas organization only; no runtime effect. |
| Webhook and external API details are placeholders in the export (e.g., `YOUR_WEBHOOK_ID`) and must be configured in your n8n instance. | Applies to Webhook/HTTP nodes. |
| Multiple nodes are Code nodes but their scripts are not included in the provided JSON; you must implement normalization, validation error formatting, categorization, SLA computation, and message/response formatting. | Applies to all Code nodes. |