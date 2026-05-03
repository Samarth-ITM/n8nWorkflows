Manage online bookings with webhook, Gmail, Google Calendar and Slack

https://n8nworkflows.xyz/workflows/manage-online-bookings-with-webhook--gmail--google-calendar-and-slack-13196


# Manage online bookings with webhook, Gmail, Google Calendar and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Manage online bookings with webhook, Gmail, Google Calendar and Slack

**Purpose:**  
This workflow receives an online booking request via a webhook, validates it, prevents duplicates, checks availability via external APIs, creates the booking, sends a confirmation email (Gmail), creates a Google Calendar event, notifies the team in Slack, logs an audit entry, and finally responds to the webhook caller. A separate error-handling path alerts Slack if the workflow fails.

### Logical Blocks
1. **1.1 Booking Intake & Validation**: Receive booking request → validate payload → reject invalid requests with a webhook response.
2. **1.2 Duplicate Prevention**: Call an external API to detect duplicates → reject duplicates with a webhook response.
3. **1.3 Availability Check**: Call an external API to check time-slot availability → reject unavailable requests with a webhook response.
4. **1.4 Booking Fulfillment**: Create booking via API → send confirmation email → prepare and create calendar event → notify team in Slack.
5. **1.5 Audit Logging & Success Response**: Build audit log payload → log via API → respond success to webhook caller.
6. **1.6 Global Error Handling**: On any workflow error, format an alert and send it to Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Booking Intake & Validation
**Overview:**  
Accepts booking requests through a webhook, validates required fields/format, and either continues or returns a validation error response.

**Nodes Involved:**  
- Booking intake (Sticky Note)  
- Webhook - New Booking  
- Validate Booking  
- IF - Valid Request  
- Format Validation Error  
- Respond - Validation Error  

#### Node Details

**Booking intake (Sticky Note)**
- **Type/Role:** Sticky Note for documentation grouping.
- **Configuration:** Empty content in this workflow JSON.
- **Failure/Edge cases:** None (non-executing).

**Webhook - New Booking**
- **Type/Role:** `Webhook` (entry point). Receives new booking HTTP requests.
- **Configuration choices (inferred):** Parameters are empty in JSON, so defaults apply (method/path/response mode not visible here). It has a `webhookId` placeholder `YOUR_WEBHOOK_ID`.
- **Connections:** Outputs to **Validate Booking**.
- **Edge cases/failures:**
  - Misconfigured webhook method/path (if not set in UI).
  - Response timing: if workflow does not respond via Respond to Webhook nodes and webhook is set to “respond at end”, caller may time out.
  - Security: no authentication configuration is shown; endpoint may be public unless restricted in UI.

**Validate Booking**
- **Type/Role:** `Code` node. Validates incoming booking payload structure and/or normalizes fields.
- **Configuration choices:** Empty parameters in JSON (the actual JS code is not provided). Typically outputs a structured object and/or a boolean like `isValid`.
- **Connections:** Outputs to **IF - Valid Request**.
- **Edge cases/failures:**
  - Missing/invalid JSON body, unexpected types.
  - Code exceptions cause the error workflow to trigger.

**IF - Valid Request**
- **Type/Role:** `IF` node. Branches based on validation result.
- **Configuration choices:** Not shown; likely checks a field from **Validate Booking** (e.g., `{{$json.valid}} === true`).
- **Connections:**
  - **True** → **API - Check Duplicates**
  - **False** → **Format Validation Error**
- **Edge cases/failures:**
  - Expression references missing properties, leading to false negatives or evaluation issues.

**Format Validation Error**
- **Type/Role:** `Code` node. Creates a consistent error response body for invalid requests.
- **Configuration choices:** Not shown; typically sets HTTP status code metadata and response JSON.
- **Connections:** → **Respond - Validation Error**
- **Edge cases/failures:** Code errors; missing fields to include in response.

**Respond - Validation Error**
- **Type/Role:** `Respond to Webhook` node. Returns validation error to caller.
- **Configuration choices:** Not shown; typically sets status (400/422) and JSON body from previous node.
- **Connections:** Terminal for this branch.
- **Edge cases/failures:** If webhook is configured for “response immediately”, this node may be redundant; if configured to “respond using node”, it is required.

---

### 2.2 Duplicate Prevention
**Overview:**  
Calls an external duplicate-check API and stops processing if the booking appears to be a duplicate.

**Nodes Involved:**  
- Duplicate and availability (Sticky Note)  
- API - Check Duplicates  
- IF - Not Duplicate  
- Format Duplicate Error  
- Respond - Duplicate  

#### Node Details

**Duplicate and availability (Sticky Note)**
- **Type/Role:** Sticky Note grouping.
- **Configuration:** Empty content.
- **Failure/Edge cases:** None.

**API - Check Duplicates**
- **Type/Role:** `HTTP Request` node. Queries an external system to detect whether this booking already exists.
- **Configuration choices:** Not visible (URL, method, auth, query/body). Version `4.2` suggests newer HTTP node behavior (separate authentication options, improved response handling).
- **Connections:** → **IF - Not Duplicate**
- **Edge cases/failures:**
  - Auth errors (API key/OAuth/mTLS) depending on setup.
  - Timeouts, 429 rate limits, 5xx responses.
  - Response schema changes breaking downstream IF logic.

**IF - Not Duplicate**
- **Type/Role:** `IF` node. Continues only if not duplicate.
- **Configuration choices:** Not shown; likely checks API response flag/array length.
- **Connections:**
  - **True** → **API - Check Availability**
  - **False** → **Format Duplicate Error**
- **Edge cases/failures:** Missing response fields or unexpected response types.

**Format Duplicate Error**
- **Type/Role:** `Code` node. Builds a standardized “duplicate booking” response.
- **Connections:** → **Respond - Duplicate**
- **Edge cases/failures:** Code errors.

**Respond - Duplicate**
- **Type/Role:** `Respond to Webhook`. Returns duplicate error to caller.
- **Edge cases/failures:** Same webhook response mode considerations as above.

---

### 2.3 Availability Check
**Overview:**  
Checks whether the requested slot/resource is available. If not, responds immediately with an “unavailable” message.

**Nodes Involved:**  
- Availability check (Sticky Note)  
- API - Check Availability  
- IF - Available  
- Format Unavailable  
- Respond - Unavailable  

#### Node Details

**Availability check (Sticky Note)**
- **Type/Role:** Sticky Note grouping.
- **Configuration:** Empty content.

**API - Check Availability**
- **Type/Role:** `HTTP Request` node. Calls an external availability service.
- **Configuration choices:** Not visible (endpoint, inputs). TypeVersion `4.2`.
- **Connections:** → **IF - Available**
- **Edge cases/failures:** Timeouts, stale availability, race conditions (availability changes after check).

**IF - Available**
- **Type/Role:** `IF` node. Branches based on availability result.
- **Connections:**
  - **True** → **API - Create Booking**
  - **False** → **Format Unavailable**
- **Edge cases/failures:** Incorrect interpretation of API response can lead to overbooking.

**Format Unavailable**
- **Type/Role:** `Code` node. Builds “slot unavailable” response payload.
- **Connections:** → **Respond - Unavailable**

**Respond - Unavailable**
- **Type/Role:** `Respond to Webhook`. Returns unavailability response.
- **Edge cases/failures:** If caller expects a specific status (409/422), ensure node is configured accordingly.

---

### 2.4 Booking Fulfillment
**Overview:**  
Creates the booking in an external system, emails the customer, creates a Google Calendar event, and notifies the team via Slack.

**Nodes Involved:**  
- Booking creation (Sticky Note)  
- API - Create Booking  
- Format Email  
- Send Confirmation Email  
- Prepare Calendar Event  
- Create Calendar Event  
- Slack - Notify Team  

#### Node Details

**Booking creation (Sticky Note)**
- **Type/Role:** Sticky Note grouping.
- **Configuration:** Empty content.

**API - Create Booking**
- **Type/Role:** `HTTP Request` node. Creates the booking in an external booking system.
- **Configuration choices:** Not visible (POST, JSON body, auth, etc.). TypeVersion `4.2`.
- **Connections:** → **Format Email**
- **Edge cases/failures:**
  - Partial failure: booking created but later email/calendar fails (idempotency needed).
  - API returns 201 with body format not expected by Format Email.

**Format Email**
- **Type/Role:** `Code` node. Prepares email subject/body/recipient based on booking data.
- **Connections:** → **Send Confirmation Email**
- **Edge cases/failures:** Missing customer email, invalid email address, formatting errors.

**Send Confirmation Email**
- **Type/Role:** `Gmail` node. Sends confirmation email via Google account.
- **Configuration choices:** Parameters not shown (To/Subject/Message likely set via expressions from Format Email). TypeVersion `2.1`.
- **Credentials:** Gmail OAuth2 credentials required in n8n.
- **Connections:** → **Prepare Calendar Event**
- **Edge cases/failures:**
  - OAuth token expired/revoked.
  - Gmail API quota, send-as restrictions, blocked content.
  - If “To” is empty, node fails.

**Prepare Calendar Event**
- **Type/Role:** `Code` node. Maps booking to Google Calendar event fields (start/end, attendees, summary, description).
- **Connections:** → **Create Calendar Event**
- **Edge cases/failures:** Timezone issues, invalid RFC3339 timestamps, missing end time.

**Create Calendar Event**
- **Type/Role:** `Google Calendar` node. Creates an event for the booking.
- **Configuration choices:** Not visible (calendar ID, operation create). TypeVersion `1.2`.
- **Credentials:** Google Calendar OAuth2 credentials required.
- **Connections:** → **Prepare Audit Entry**
- **Edge cases/failures:** Permissions on calendar, invalid attendee emails, conference data settings, timezone mismatch.

**Slack - Notify Team**
- **Type/Role:** `Slack` node. Sends a team notification about the booking.
- **Configuration choices:** Not visible (channel, message blocks). TypeVersion `2.2`.
- **Credentials:** Slack bot token or Slack OAuth credentials (depending on node configuration). Note: node has a `webhookId` field internally; actual Slack auth still required via credentials.
- **Connections:** → **Prepare Audit Entry**
- **Edge cases/failures:** Missing channel permission, invalid channel ID, rate limiting.

**Important structural note:**  
Both **Create Calendar Event** and **Slack - Notify Team** feed into **Prepare Audit Entry**. In n8n, this means **Prepare Audit Entry** can execute multiple times (once per incoming branch) unless it is configured to merge or you use a Merge node. Since no Merge node exists here, audit logging may occur twice (or the second run may overwrite assumptions). This is a key edge case to address.

---

### 2.5 Audit Logging & Success Response
**Overview:**  
Builds an audit record, sends it to an external audit endpoint, formats a success response, and responds to the webhook caller.

**Nodes Involved:**  
- Audit and response (Sticky Note)  
- Prepare Audit Entry  
- API - Log Audit  
- Format Success  
- Respond - Success  

#### Node Details

**Audit and response (Sticky Note)**
- **Type/Role:** Sticky Note grouping.
- **Configuration:** Empty content.

**Prepare Audit Entry**
- **Type/Role:** `Code` node. Builds audit payload (booking id, timestamps, actions taken).
- **Connections:** → **API - Log Audit**
- **Inputs:** From **Create Calendar Event** and **Slack - Notify Team** (two separate upstreams).
- **Edge cases/failures:**
  - Runs multiple times due to two upstream paths.
  - If it expects fields present only from one upstream node, the other path may produce undefined values.

**API - Log Audit**
- **Type/Role:** `HTTP Request` node. Sends audit record to external logging/audit service.
- **Connections:** → **Format Success**
- **Edge cases/failures:** Logging endpoint down; should not block customer response unless required.

**Format Success**
- **Type/Role:** `Code` node. Creates final webhook response payload (likely includes booking ID, calendar event link, etc.).
- **Connections:** → **Respond - Success**

**Respond - Success**
- **Type/Role:** `Respond to Webhook`. Returns success response to webhook caller.
- **Edge cases/failures:** If the workflow triggers Respond nodes on other branches, ensure only one response is sent per execution.

---

### 2.6 Global Error Handling
**Overview:**  
Catches any workflow execution errors and sends an alert to Slack with formatted error details.

**Nodes Involved:**  
- Error handling (Sticky Note)  
- Error Trigger  
- Format Error  
- Slack - Error Alert  

#### Node Details

**Error handling (Sticky Note)**
- **Type/Role:** Sticky Note grouping.
- **Configuration:** Empty content.

**Error Trigger**
- **Type/Role:** `Error Trigger` node. Starts this workflow when another workflow errors (or when this workflow has an error workflow configured).
- **Configuration choices:** Empty in JSON; behavior depends on how n8n “Error Workflow” is configured at workflow level.
- **Connections:** → **Format Error**
- **Edge cases/failures:** Not triggered unless configured as an error workflow or used properly in project settings.

**Format Error**
- **Type/Role:** `Code` node. Formats error payload for Slack alert (message, stack, workflow info, execution URL).
- **Connections:** → **Slack - Error Alert**
- **Edge cases/failures:** Large stack traces may exceed Slack message limits unless truncated.

**Slack - Error Alert**
- **Type/Role:** `Slack` node. Sends error alert to Slack.
- **Configuration choices:** Not visible (channel/message).
- **Credentials:** Slack credentials required.
- **Edge cases/failures:** Rate limiting; missing scopes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documentation/section header | — | — |  |
| Booking intake | Sticky Note | Documentation/section header | — | — |  |
| Webhook - New Booking | Webhook | Receives booking request | — | Validate Booking |  |
| Validate Booking | Code | Validate/normalize request | Webhook - New Booking | IF - Valid Request |  |
| IF - Valid Request | IF | Route valid vs invalid | Validate Booking | API - Check Duplicates; Format Validation Error |  |
| Format Validation Error | Code | Build validation error response | IF - Valid Request (false) | Respond - Validation Error |  |
| Respond - Validation Error | Respond to Webhook | Return validation error to caller | Format Validation Error | — |  |
| Duplicate and availability | Sticky Note | Documentation/section header | — | — |  |
| API - Check Duplicates | HTTP Request | Duplicate detection via API | IF - Valid Request (true) | IF - Not Duplicate |  |
| IF - Not Duplicate | IF | Route non-duplicate vs duplicate | API - Check Duplicates | API - Check Availability; Format Duplicate Error |  |
| Format Duplicate Error | Code | Build duplicate response | IF - Not Duplicate (false) | Respond - Duplicate |  |
| Respond - Duplicate | Respond to Webhook | Return duplicate error to caller | Format Duplicate Error | — |  |
| Availability check | Sticky Note | Documentation/section header | — | — |  |
| API - Check Availability | HTTP Request | Availability lookup via API | IF - Not Duplicate (true) | IF - Available |  |
| IF - Available | IF | Route available vs unavailable | API - Check Availability | API - Create Booking; Format Unavailable |  |
| Format Unavailable | Code | Build unavailable response | IF - Available (false) | Respond - Unavailable |  |
| Respond - Unavailable | Respond to Webhook | Return unavailability response | Format Unavailable | — |  |
| Booking creation | Sticky Note | Documentation/section header | — | — |  |
| API - Create Booking | HTTP Request | Create booking in system | IF - Available (true) | Format Email |  |
| Format Email | Code | Prepare email fields/content | API - Create Booking | Send Confirmation Email |  |
| Send Confirmation Email | Gmail | Send customer confirmation | Format Email | Prepare Calendar Event |  |
| Prepare Calendar Event | Code | Map booking to calendar event | Send Confirmation Email | Create Calendar Event |  |
| Create Calendar Event | Google Calendar | Create event | Prepare Calendar Event | Prepare Audit Entry |  |
| Slack - Notify Team | Slack | Notify internal team | (Not connected upstream in JSON) | Prepare Audit Entry |  |
| Audit and response | Sticky Note | Documentation/section header | — | — |  |
| Prepare Audit Entry | Code | Create audit log payload | Create Calendar Event; Slack - Notify Team | API - Log Audit |  |
| API - Log Audit | HTTP Request | Persist audit record | Prepare Audit Entry | Format Success |  |
| Format Success | Code | Build success webhook response | API - Log Audit | Respond - Success |  |
| Respond - Success | Respond to Webhook | Return success to caller | Format Success | — |  |
| Error handling | Sticky Note | Documentation/section header | — | — |  |
| Error Trigger | Error Trigger | Entry point on failures | — | Format Error |  |
| Format Error | Code | Format Slack error payload | Error Trigger | Slack - Error Alert |  |
| Slack - Error Alert | Slack | Send error alert | Format Error | — |  |

**Note:** In the provided JSON, **Slack - Notify Team** has no incoming connection. It will never run unless connected in the editor or triggered by another mechanism. This is likely an oversight in the exported connections.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Manage online bookings with webhook, Gmail, Google Calendar and Slack*.

2. **Add Webhook trigger**
   - Node: **Webhook**
   - Name: *Webhook - New Booking*
   - Configure:
     - HTTP Method (commonly POST)
     - Path (e.g., `/new-booking`)
     - Response mode: choose one of:
       - “Using Respond to Webhook node” (recommended here because multiple early exits exist), or
       - Ensure only one response is produced at end.
   - Save and copy the Test/Production URL for your booking form/service.

3. **Add validation Code node**
   - Node: **Code**
   - Name: *Validate Booking*
   - Implement logic to:
     - Parse input fields (name, email, start/end time, service, etc.)
     - Output a flag (e.g., `valid: true/false`) and normalized booking object.
   - Connect: **Webhook - New Booking → Validate Booking**

4. **Add IF node for validation**
   - Node: **IF**
   - Name: *IF - Valid Request*
   - Condition: check your validation flag (e.g., `{{$json.valid}} is true`)
   - Connect: **Validate Booking → IF - Valid Request**

5. **Invalid branch response**
   - Add **Code** node: *Format Validation Error* (build `{ error: "...", details: ... }`)
   - Add **Respond to Webhook**: *Respond - Validation Error*
     - Set status code (e.g., 400/422) and body from previous node.
   - Connect:  
     - **IF - Valid Request (false) → Format Validation Error → Respond - Validation Error**

6. **Duplicate check via API**
   - Add **HTTP Request** node: *API - Check Duplicates*
     - Configure URL/method/auth to your booking datastore (e.g., search by email + start time).
     - Ensure response indicates duplicate or not (boolean or list).
   - Connect: **IF - Valid Request (true) → API - Check Duplicates**

7. **IF node for duplicates**
   - Add **IF** node: *IF - Not Duplicate*
   - Condition: based on duplicate API result (e.g., `{{$json.isDuplicate}} is false`).
   - Connect: **API - Check Duplicates → IF - Not Duplicate**

8. **Duplicate branch response**
   - Add **Code**: *Format Duplicate Error*
   - Add **Respond to Webhook**: *Respond - Duplicate* (status 409 recommended)
   - Connect: **IF - Not Duplicate (false) → Format Duplicate Error → Respond - Duplicate**

9. **Availability check via API**
   - Add **HTTP Request** node: *API - Check Availability*
     - Configure to check slot availability (resource/service/time window).
   - Connect: **IF - Not Duplicate (true) → API - Check Availability**

10. **IF node for availability**
   - Add **IF** node: *IF - Available*
   - Condition: `available === true`
   - Connect: **API - Check Availability → IF - Available**

11. **Unavailable branch response**
   - Add **Code**: *Format Unavailable*
   - Add **Respond to Webhook**: *Respond - Unavailable* (status 409/422)
   - Connect: **IF - Available (false) → Format Unavailable → Respond - Unavailable**

12. **Create booking via API**
   - Add **HTTP Request**: *API - Create Booking*
     - Configure POST to create booking record.
     - Return booking ID and canonical times.
   - Connect: **IF - Available (true) → API - Create Booking**

13. **Prepare and send confirmation email (Gmail)**
   - Add **Code**: *Format Email* (create `to`, `subject`, `text`/`html`)
   - Add **Gmail** node: *Send Confirmation Email*
     - Operation: Send
     - Map “To/Subject/Message” from **Format Email** outputs.
     - **Credentials:** create/select Gmail OAuth2 credentials (Google Cloud project or n8n Google OAuth).
   - Connect: **API - Create Booking → Format Email → Send Confirmation Email**

14. **Prepare and create Google Calendar event**
   - Add **Code**: *Prepare Calendar Event* (summary, start/end with timezone, attendees)
   - Add **Google Calendar** node: *Create Calendar Event*
     - Operation: Create
     - Choose Calendar ID
     - Map start/end/summary/description/attendees
     - **Credentials:** Google Calendar OAuth2 credentials.
   - Connect: **Send Confirmation Email → Prepare Calendar Event → Create Calendar Event**

15. **Slack team notification**
   - Add **Slack** node: *Slack - Notify Team*
     - Operation: Post message (channel)
     - Compose message with booking details.
     - **Credentials:** Slack OAuth/bot token with permission to post to the channel.
   - **Important:** Ensure it has an incoming connection. Commonly:
     - Connect **Create Calendar Event → Slack - Notify Team**, or
     - Use a **Merge** node to combine data before notifying.
   - In the provided JSON, Slack notify is effectively disconnected; fix this when rebuilding.

16. **Audit logging**
   - Add **Code**: *Prepare Audit Entry* (include booking ID, email sent=true, calendarEventId, slackNotified=true, timestamps)
   - Add **HTTP Request**: *API - Log Audit* (POST audit record)
   - Connect:  
     - Either **Slack - Notify Team → Prepare Audit Entry → API - Log Audit**, or  
     - If you need both Calendar + Slack results, insert a **Merge** node (Recommended):
       1) Branch A: Create Calendar Event → Merge  
       2) Branch B: Slack - Notify Team → Merge  
       3) Merge → Prepare Audit Entry → API - Log Audit

17. **Success response**
   - Add **Code**: *Format Success* (build success JSON: bookingId, start/end, calendar link, etc.)
   - Add **Respond to Webhook**: *Respond - Success* (status 200/201)
   - Connect: **API - Log Audit → Format Success → Respond - Success**

18. **Global error workflow**
   - In the same workflow (as in JSON) or as a separate workflow:
     - Add **Error Trigger** node.
     - Add **Code**: *Format Error*
     - Add **Slack**: *Slack - Error Alert*
     - Connect: **Error Trigger → Format Error → Slack - Error Alert**
   - Ensure n8n is configured to use this workflow as the **Error Workflow** (Workflow settings / instance settings depending on your n8n setup).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes present but empty | The workflow contains section headers (“Overview”, “Booking intake”, etc.) with no text content in this export. |
| Slack - Notify Team has no incoming connection in the exported connections | This node will not execute unless connected during rebuild; consider connecting it after calendar creation or using a Merge pattern. |
| Audit step may run twice without a Merge node | Because Prepare Audit Entry has two upstream inputs (Calendar + Slack) in JSON, it can execute more than once unless merged/controlled. |