Capture and score leads with SQL Server and Slack alerts

https://n8nworkflows.xyz/workflows/capture-and-score-leads-with-sql-server-and-slack-alerts-13074


# Capture and score leads with SQL Server and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Capture and score leads with SQL Server and Slack alerts

**Purpose:**  
This workflow receives lead submissions via a webhook, normalizes and validates the incoming data, upserts the lead into an external system via HTTP API (described as SQL Server in the title, but implemented here as HTTP requests), calculates a lead score, routes the lead based on the score (hot/warm/cold), sends Slack alerts for hot leads, updates the score back to the external system, and finally returns a success/error response to the webhook caller. A separate error-handling path uses an Error Trigger to notify Slack on workflow failures.

### Logical blocks
1. **1.1 Input reception & normalization**: Webhook receives lead → code node normalizes fields.
2. **1.2 Validation & early rejection**: IF checks email validity → respond with validation error if invalid.
3. **1.3 “Database” / CRM upsert (via API)**: Prepare request body → HTTP upsert contact.
4. **1.4 Lead scoring & routing**: Calculate score → Switch routes hot/warm/cold.
5. **1.5 Alerts & score persistence**: Hot: format Slack message → Slack alert → API update score. Warm: API update score. Cold: no-op logging.
6. **1.6 Webhook response**: Format success payload → respond to webhook.
7. **1.7 Global error handling**: Error Trigger → format error → Slack error alert.

---

## 2. Block-by-Block Analysis

### 2.1 Input reception & normalization
**Overview:** Receives inbound lead data through a webhook endpoint and standardizes the payload into a predictable structure for downstream nodes.

**Nodes involved:**
- Webhook - Lead Capture
- Normalize Lead Data

#### Node: Webhook - Lead Capture
- **Type / role:** `Webhook` (entry point). Accepts HTTP requests from a form/website/lead source.
- **Configuration (interpreted):**
  - Webhook is present with a placeholder `webhookId: YOUR_WEBHOOK_ID`.
  - Other webhook settings (HTTP method, path, response mode, authentication) are not specified in the provided JSON and must be set in n8n UI.
- **Key variables/expressions:** Not specified.
- **Connections:**
  - **Output →** Normalize Lead Data
- **Version-specific notes:** Node typeVersion 1 (older webhook node variant; still supported).
- **Edge cases / failures:**
  - If response mode is “On Received” but later nodes try to respond, you may get double-response issues. Typically use “Respond to Webhook” nodes with “Response Mode: Using ‘Respond to Webhook’ node”.
  - Missing/invalid incoming JSON shape can break subsequent normalization code.
  - Authentication not configured can expose endpoint publicly.

#### Node: Normalize Lead Data
- **Type / role:** `Code` (data transformation). Normalizes incoming lead fields (names, email, phone, company, etc.).
- **Configuration (interpreted):**
  - Parameters are empty in JSON; actual JS code is not included, so expected behavior must be implemented manually.
  - Typical actions: trim strings, lowercase email, map alternative field names (e.g., `mail` → `email`), ensure consistent keys.
- **Key variables/expressions:** Not available (code content missing).
- **Connections:**
  - **Input ←** Webhook - Lead Capture
  - **Output →** IF - Valid Email
- **Version-specific notes:** Code node typeVersion 2 uses the newer Code node runtime model.
- **Edge cases / failures:**
  - If code assumes fields exist (e.g., `items[0].json.email`) and they don’t, it can throw.
  - If multiple items are received and code only handles one, downstream behavior may be inconsistent.

---

### 2.2 Validation & early rejection
**Overview:** Validates that the lead has a usable email. Invalid submissions return an error response immediately.

**Nodes involved:**
- IF - Valid Email
- Format Error Response
- Respond - Validation Error

#### Node: IF - Valid Email
- **Type / role:** `IF` (branching). Splits flow into “valid” vs “invalid” lead paths.
- **Configuration (interpreted):**
  - IF parameters are empty in JSON; actual condition not provided. You must configure a condition such as:
    - “email” exists AND matches a regex / contains `@`.
- **Key variables/expressions:** Not provided; typically references `{{$json.email}}`.
- **Connections:**
  - **Input ←** Normalize Lead Data
  - **True output (index 0) →** Prepare API Body
  - **False output (index 1) →** Format Error Response
- **Edge cases / failures:**
  - Loose validation (only checking `@`) can allow bad emails; strict regex can block valid edge formats.
  - If normalization didn’t set `email`, all leads may be rejected.

#### Node: Format Error Response
- **Type / role:** `Code` (response shaping). Creates a consistent error payload for invalid leads.
- **Configuration (interpreted):**
  - No code shown; implement an object like `{ success:false, error:"Invalid email", details:{...} }`.
- **Connections:**
  - **Input ←** IF - Valid Email (false)
  - **Output →** Respond - Validation Error
- **Edge cases / failures:**
  - If Respond node expects certain fields (statusCode/body) and they are missing, response may be wrong.

#### Node: Respond - Validation Error
- **Type / role:** `Respond to Webhook` (terminates request). Returns error to caller.
- **Configuration (interpreted):**
  - Parameters empty; should set HTTP status (e.g., 400) and response body from previous node.
- **Connections:**
  - **Input ←** Format Error Response
- **Version-specific notes:** typeVersion 1.
- **Edge cases / failures:**
  - If webhook is set to respond immediately, this node may never run or produce errors.

---

### 2.3 “Database operations” (implemented via API)
**Overview:** Prepares a request payload and upserts the lead into an external system via HTTP. Despite the title mentioning SQL Server, the workflow uses HTTP Request nodes rather than a native SQL Server node.

**Nodes involved:**
- Prepare API Body
- API - Upsert Contact

#### Node: Prepare API Body
- **Type / role:** `Code` (request shaping). Builds the body for the upsert API call.
- **Configuration (interpreted):**
  - No code content provided; typically maps normalized lead fields into the API schema.
- **Connections:**
  - **Input ←** IF - Valid Email (true)
  - **Output →** API - Upsert Contact
- **Edge cases / failures:**
  - Missing required fields for the external API (e.g., email) cause upsert failure.
  - Wrong content-type expectations (JSON vs form) if not configured in HTTP node.

#### Node: API - Upsert Contact
- **Type / role:** `HTTP Request` (external integration). Upserts the contact/lead.
- **Configuration (interpreted):**
  - Parameters empty; you must configure:
    - Method (often `POST` or `PUT`)
    - URL (endpoint of the system)
    - Authentication (API key/OAuth2/etc.)
    - Body from “Prepare API Body”
- **Connections:**
  - **Input ←** Prepare API Body
  - **Output →** Calculate Lead Score
- **Version-specific notes:** typeVersion 4 (newer HTTP Request node).
- **Edge cases / failures:**
  - Auth errors (401/403), endpoint errors (404), validation errors (400), server errors (5xx).
  - Timeouts / retries not configured can lead to intermittent failures.
  - If the API response doesn’t include an ID needed later, score update may be impossible.

---

### 2.4 Lead scoring & routing
**Overview:** Calculates a lead score and routes processing based on score bands: hot, warm, or cold.

**Nodes involved:**
- Calculate Lead Score
- Score Router

#### Node: Calculate Lead Score
- **Type / role:** `Code` (business logic). Computes a numeric score from lead attributes (e.g., company size, email domain, form fields).
- **Configuration (interpreted):**
  - Code missing; must output a field such as `score` and possibly a category.
- **Connections:**
  - **Input ←** API - Upsert Contact
  - **Output →** Score Router
- **Edge cases / failures:**
  - If it expects fields from the upsert response but they aren’t present, scoring can fail.
  - Ensure score is numeric; Switch node comparisons can break with strings.

#### Node: Score Router
- **Type / role:** `Switch` (multi-branch routing). Directs hot/warm/cold flows.
- **Configuration (interpreted):**
  - Parameters empty in JSON; must configure rules based on `score` (e.g., `>= 80` hot, `>= 50` warm, else cold).
- **Connections (as wired):**
  - **Input ←** Calculate Lead Score
  - **Output 0 →** Format Slack Alert (Hot path)
  - **Output 1 →** API - Update Score (Warm)
  - **Output 2 →** Log Cold Lead
- **Version-specific notes:** typeVersion 2.
- **Edge cases / failures:**
  - Unmatched cases: if Switch has no default route, items can be dropped.
  - If `score` is undefined, routing may go to default or nowhere depending on configuration.

---

### 2.5 Alerts & score persistence
**Overview:** Notifies Slack for hot leads, updates score back to the external system for hot and warm leads, and logs cold leads (no-op).

**Nodes involved:**
- Format Slack Alert
- Slack - Hot Lead Alert
- API - Update Score (Hot)
- API - Update Score (Warm)
- Log Cold Lead

#### Node: Format Slack Alert
- **Type / role:** `Code` (message composition). Builds Slack message text/blocks/attachments.
- **Configuration (interpreted):**
  - Code missing; typically includes lead name, email, score, and a link to CRM record.
- **Connections:**
  - **Input ←** Score Router (hot branch)
  - **Output →** Slack - Hot Lead Alert
- **Edge cases / failures:**
  - Slack node may require specific fields; ensure the output matches Slack node operation (message text vs blocks).

#### Node: Slack - Hot Lead Alert
- **Type / role:** `Slack` (notification). Sends alert to a Slack channel.
- **Configuration (interpreted):**
  - `webhookId` is present (internal n8n reference), but actual Slack credentials/config are not shown.
  - Configure operation (likely “Post message”) and channel/message mapping.
- **Connections:**
  - **Input ←** Format Slack Alert
  - **Output →** API - Update Score (Hot)
- **Version-specific notes:** typeVersion 2.
- **Edge cases / failures:**
  - Slack auth misconfiguration (invalid token/webhook) causes failures.
  - Rate limits if many hot leads.

#### Node: API - Update Score (Hot)
- **Type / role:** `HTTP Request`. Updates the lead score/status for hot leads in external system.
- **Configuration (interpreted):**
  - Parameters empty; set URL/method/auth/body. Usually references lead/contact ID from upsert response plus computed score.
- **Connections:**
  - **Input ←** Slack - Hot Lead Alert
  - **Output →** Format Success Response
- **Edge cases / failures:**
  - If Slack fails, this step never runs (unless you add error handling or “Continue On Fail”).
  - Missing ID/keys for update.

#### Node: API - Update Score (Warm)
- **Type / role:** `HTTP Request`. Updates score/status for warm leads (no Slack alert).
- **Configuration (interpreted):** Not provided; same considerations as hot update.
- **Connections:**
  - **Input ←** Score Router (warm branch)
  - **Output →** Format Success Response
- **Edge cases / failures:** Same as above (auth, timeouts, missing identifiers).

#### Node: Log Cold Lead
- **Type / role:** `No Operation`. Placeholder for cold lead handling (could later be replaced by database insert, tagging, etc.).
- **Configuration (interpreted):** Does nothing; passes data through.
- **Connections:**
  - **Input ←** Score Router (cold branch)
  - **Output →** Format Success Response
- **Edge cases / failures:** None functionally; but it also means cold leads are not persisted/updated unless handled earlier.

---

### 2.6 Webhook response
**Overview:** Formats a consistent success response and returns it to the webhook caller after processing (hot/warm/cold).

**Nodes involved:**
- Format Success Response
- Respond - Success

#### Node: Format Success Response
- **Type / role:** `Code` (response shaping). Builds success JSON including score/category and any IDs.
- **Configuration (interpreted):**
  - Code missing; typically `{ success:true, score, route:"hot|warm|cold", contactId }`.
- **Connections:**
  - **Input ←** API - Update Score (Hot) OR API - Update Score (Warm) OR Log Cold Lead
  - **Output →** Respond - Success
- **Edge cases / failures:**
  - If one branch returns different fields, response can vary unless normalized.

#### Node: Respond - Success
- **Type / role:** `Respond to Webhook`. Returns final success response.
- **Configuration (interpreted):**
  - Parameters empty; configure status (200/201) and body from previous node.
- **Connections:**
  - **Input ←** Format Success Response
- **Edge cases / failures:**
  - Same double-response caveat as earlier: ensure webhook is set to respond via this node.

---

### 2.7 Global error handling
**Overview:** Catches workflow execution errors (outside the normal branching logic) and posts an alert to Slack.

**Nodes involved:**
- Error Trigger
- Format Error
- Slack - Error Alert

#### Node: Error Trigger
- **Type / role:** `Error Trigger` (secondary entry point). Runs when the workflow errors (depending on n8n settings).
- **Configuration (interpreted):** Default; no parameters.
- **Connections:**
  - **Output →** Format Error
- **Version-specific notes:** typeVersion 1.
- **Edge cases / failures:**
  - Requires workflow error handling to be enabled/available in the environment.
  - It will not catch errors if nodes are set to “Continue On Fail” (since the workflow may not “error”).

#### Node: Format Error
- **Type / role:** `Code`. Converts error context into a Slack-friendly message payload.
- **Configuration (interpreted):**
  - Code missing; should include workflow name, failing node, error message, execution ID, and timestamp.
- **Connections:**
  - **Input ←** Error Trigger
  - **Output →** Slack - Error Alert
- **Edge cases / failures:**
  - If it assumes error structure fields that aren’t present, it may throw and prevent alerting.

#### Node: Slack - Error Alert
- **Type / role:** `Slack`. Sends error notification to Slack.
- **Configuration (interpreted):**
  - Slack credentials not shown; configure a channel and message mapping.
- **Connections:**
  - **Input ←** Format Error
- **Edge cases / failures:**
  - Slack auth/rate limits; if Slack fails, you lose error visibility (consider fallback like email).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Visual section header | — | — |  |
| Data validation | Sticky Note | Visual section header | — | — |  |
| Webhook - Lead Capture | Webhook | Lead intake endpoint | — | Normalize Lead Data |  |
| Normalize Lead Data | Code | Normalize incoming payload | Webhook - Lead Capture | IF - Valid Email |  |
| IF - Valid Email | IF | Validate email and branch | Normalize Lead Data | Prepare API Body; Format Error Response |  |
| Format Error Response | Code | Build validation error response | IF - Valid Email | Respond - Validation Error |  |
| Respond - Validation Error | Respond to Webhook | Return 4xx error to caller | Format Error Response | — |  |
| Database operations | Sticky Note | Visual section header | — | — |  |
| Prepare API Body | Code | Build upsert request payload | IF - Valid Email | API - Upsert Contact |  |
| API - Upsert Contact | HTTP Request | Upsert lead/contact via API | Prepare API Body | Calculate Lead Score |  |
| Lead scoring and routing | Sticky Note | Visual section header | — | — |  |
| Calculate Lead Score | Code | Compute lead score | API - Upsert Contact | Score Router |  |
| Score Router | Switch | Route hot/warm/cold | Calculate Lead Score | Format Slack Alert; API - Update Score (Warm); Log Cold Lead |  |
| Format Slack Alert | Code | Compose Slack alert message | Score Router | Slack - Hot Lead Alert |  |
| Slack - Hot Lead Alert | Slack | Notify Slack about hot lead | Format Slack Alert | API - Update Score (Hot) |  |
| API - Update Score (Hot) | HTTP Request | Persist score/status for hot lead | Slack - Hot Lead Alert | Format Success Response |  |
| API - Update Score (Warm) | HTTP Request | Persist score/status for warm lead | Score Router | Format Success Response |  |
| Log Cold Lead | NoOp | Placeholder for cold lead handling | Score Router | Format Success Response |  |
| Webhook response | Sticky Note | Visual section header | — | — |  |
| Format Success Response | Code | Build success response payload | API - Update Score (Hot); API - Update Score (Warm); Log Cold Lead | Respond - Success |  |
| Respond - Success | Respond to Webhook | Return 2xx success to caller | Format Success Response | — |  |
| Error handling | Sticky Note | Visual section header | — | — |  |
| Error Trigger | Error Trigger | Runs on workflow errors | — | Format Error |  |
| Format Error | Code | Format error payload for Slack | Error Trigger | Slack - Error Alert |  |
| Slack - Error Alert | Slack | Send error notification to Slack | Format Error | — |  |

> Note: All sticky notes have empty content in the provided workflow export, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: **“Capture and score leads with SQL Server and Slack alerts”**.

2. **Add node: Webhook**
   - Name: **Webhook - Lead Capture**
   - Configure:
     - **HTTP Method:** POST (typical for lead forms)
     - **Path:** e.g., `/lead-capture`
     - **Response mode:** *Respond via “Respond to Webhook” node*
     - Optional: authentication (recommended)
   - Save the node to generate a production/test URL.

3. **Add node: Code**
   - Name: **Normalize Lead Data**
   - Implement normalization logic, e.g.:
     - Extract fields from webhook body
     - `email = email.trim().toLowerCase()`
     - Ensure output JSON has consistent keys: `firstName`, `lastName`, `email`, `company`, `source`, etc.
   - Connect: **Webhook - Lead Capture → Normalize Lead Data**

4. **Add node: IF**
   - Name: **IF - Valid Email**
   - Configure condition(s), for example:
     - `{{$json.email}}` **is not empty**
     - and `{{$json.email}}` **matches regex** (or contains `@`)
   - Connect: **Normalize Lead Data → IF - Valid Email**

5. **Invalid path (false): Add node: Code**
   - Name: **Format Error Response**
   - Create response body, and optionally include an HTTP status indicator you will map in the respond node.
   - Connect: **IF - Valid Email (false) → Format Error Response**

6. **Invalid path: Add node: Respond to Webhook**
   - Name: **Respond - Validation Error**
   - Configure:
     - **Status code:** 400
     - **Body:** use the JSON from **Format Error Response**
   - Connect: **Format Error Response → Respond - Validation Error**

7. **Valid path (true): Add node: Code**
   - Name: **Prepare API Body**
   - Map normalized lead fields into the schema expected by your external API (CRM/SQL layer).
   - Connect: **IF - Valid Email (true) → Prepare API Body**

8. **Add node: HTTP Request**
   - Name: **API - Upsert Contact**
   - Configure:
     - **Method:** POST/PUT (depending on your API)
     - **URL:** your upsert endpoint
     - **Auth:** configure credentials (API key/OAuth2/etc.)
     - **Send Body:** JSON from **Prepare API Body**
   - Connect: **Prepare API Body → API - Upsert Contact**

9. **Add node: Code**
   - Name: **Calculate Lead Score**
   - Compute `score` (number) and optionally `segment` (`hot|warm|cold`).
   - Ensure the output includes any required identifier from the upsert response (e.g., `contactId`).
   - Connect: **API - Upsert Contact → Calculate Lead Score**

10. **Add node: Switch**
    - Name: **Score Router**
    - Configure rules on `{{$json.score}}`, e.g.:
      - Output 0 (Hot): `score >= 80`
      - Output 1 (Warm): `score >= 50`
      - Output 2 (Cold): default/otherwise
    - Connect: **Calculate Lead Score → Score Router**

11. **Hot branch: Add node: Code**
    - Name: **Format Slack Alert**
    - Build Slack message (text/blocks) including lead details and score.
    - Connect: **Score Router (Hot) → Format Slack Alert**

12. **Hot branch: Add node: Slack**
    - Name: **Slack - Hot Lead Alert**
    - Configure Slack credentials (bot token or webhook, depending on node operation).
    - Operation: typically “Post message”
    - Channel: your sales/alerts channel
    - Message: from **Format Slack Alert**
    - Connect: **Format Slack Alert → Slack - Hot Lead Alert**

13. **Hot branch: Add node: HTTP Request**
    - Name: **API - Update Score (Hot)**
    - Configure:
      - Method: PATCH/POST
      - URL: score update endpoint
      - Body includes `contactId` and `score` (+ any “hot” status flag)
      - Auth as required
    - Connect: **Slack - Hot Lead Alert → API - Update Score (Hot)**

14. **Warm branch: Add node: HTTP Request**
    - Name: **API - Update Score (Warm)**
    - Similar to hot update, but set warm status.
    - Connect: **Score Router (Warm) → API - Update Score (Warm)**

15. **Cold branch: Add node: NoOp**
    - Name: **Log Cold Lead**
    - (Optional improvement: replace with DB insert, Slack digest, or tagging.)
    - Connect: **Score Router (Cold) → Log Cold Lead**

16. **Add node: Code**
    - Name: **Format Success Response**
    - Build a unified response body for all branches (hot/warm/cold), e.g. include `score`, `segment`, `contactId`.
    - Connect:
      - **API - Update Score (Hot) → Format Success Response**
      - **API - Update Score (Warm) → Format Success Response**
      - **Log Cold Lead → Format Success Response**

17. **Add node: Respond to Webhook**
    - Name: **Respond - Success**
    - Configure:
      - **Status code:** 200 (or 201)
      - **Body:** from **Format Success Response**
    - Connect: **Format Success Response → Respond - Success**

18. **Add error handling path**
    - Add node: **Error Trigger** (this is a separate entry)
      - Name: **Error Trigger**
    - Add node: **Code**
      - Name: **Format Error**
      - Build Slack message from error trigger payload (execution id, error message, node name).
    - Add node: **Slack**
      - Name: **Slack - Error Alert**
      - Configure Slack credentials/channel for ops/errors
    - Connect: **Error Trigger → Format Error → Slack - Error Alert**

19. **Credentials to configure**
    - **Slack:** Slack node credentials (bot token) or webhook-based configuration depending on node operation in your n8n version.
    - **HTTP APIs:** credentials for your upsert/update endpoints (API key/OAuth2/basic, etc.).
    - **Webhook security:** consider header auth, basic auth, or a secret token in the URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist for sections (“Overview”, “Data validation”, “Database operations”, “Lead scoring and routing”, “Webhook response”, “Error handling”) but their contents are empty in the provided export. | Visual organization only (no embedded documentation). |
| The workflow title mentions “SQL Server”, but the implementation provided uses **HTTP Request** nodes rather than a native SQL Server node. | If SQL Server is intended, replace HTTP nodes with an MSSQL/SQL Server node or a sub-workflow that performs SQL upserts/updates. |