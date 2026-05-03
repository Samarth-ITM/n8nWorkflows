Schedule appointments with manual approval using Google Calendar and Gmail

https://n8nworkflows.xyz/workflows/schedule-appointments-with-manual-approval-using-google-calendar-and-gmail-12419


# Schedule appointments with manual approval using Google Calendar and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (in JSON):** *Schedule appointments using Google Calendar and Gmail*  
**Stated title (user):** *Schedule appointments with manual approval using Google Calendar and Gmail*

**Purpose:**  
This workflow receives appointment requests via an HTTP webhook, checks the requested time slot against Google Calendar availability, immediately responds to the requester (slot available/unavailable), and—if available—initiates a manual approval via Gmail. Once approved, it books the event in Google Calendar; if rejected, it emails the requester a rejection message.

**Primary use cases:**
- Website “Book a call” form that posts appointment details to n8n
- Internal “human-in-the-loop” scheduling (HR/assistant approval)
- Prevent double booking by checking calendar availability before approval

### 1.1 Input Reception & Normalization
Receives the POST request, extracts the relevant fields, and computes an end time (+1 hour).

### 1.2 Availability Check & Immediate HTTP Response
Queries Google Calendar free/busy availability and returns a JSON response to the caller right away.

### 1.3 Manual Approval, Booking, and Notifications
If the slot is available, sends an approval email (Gmail “send and wait”), then branches:
- Approved → create Google Calendar event (with Meet link, attendees, description)
- Rejected → send a rejection email to the requester

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization

**Overview:**  
Accepts an appointment request via webhook and shapes incoming data into a consistent schema used by later nodes, including calculating an end date/time.

**Nodes involved:**
- Receive appointment request
- Format appointment data

#### Node: Receive appointment request
- **Type / role:** `Webhook` — entry point; accepts HTTP POST requests.
- **Key configuration:**
  - **Path:** `appointment-request`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (meaning the HTTP response is produced by a later “Respond to Webhook” node)
  - **Options:**
    - `ignoreBots: true` (helps avoid automated/bot hits)
    - Adds response header `Content-Type: application/json`
- **Input/Output:**
  - **Input:** External HTTP caller
  - **Output:** Flows to **Format appointment data**
- **Edge cases / failures:**
  - Missing/invalid JSON body fields will propagate as expression errors or null values later (e.g., `body.appointmentDateTime`).
  - If no “Respond to Webhook” executes (e.g., workflow errors before it), the webhook caller can time out.

#### Node: Format appointment data
- **Type / role:** `Set` — transforms and renames fields; computes end time.
- **Configuration choices (interpreted):**
  - Keeps selected fields and sets:
    - `name` ← `body.name`
    - `company` ← `body.organization`
    - `email` ← `body.email`
    - `phone` ← `body.phone`
    - `appointmentStartDate` ← `body.appointmentDateTime`
    - `appointmentEndDate` ← start time converted to `Asia/Kolkata` then **+ 1 hour**
    - `message` ← `body.optionalMessage`
  - Includes original `body` (because `includeFields: body` + `includeOtherFields: true`)
  - Dot notation disabled (`dotNotation: false`)
- **Key expressions / variables:**
  - `appointmentEndDate`:
    - `DateTime.fromISO($json.body.appointmentDateTime).setZone('Asia/Kolkata').plus(1, 'hours')`
- **Input/Output:**
  - Input from **Receive appointment request**
  - Output to **Check Availability**
- **Edge cases / failures:**
  - If `appointmentDateTime` is not valid ISO-8601, `DateTime.fromISO(...)` may yield invalid DateTime → downstream failures.
  - Time zone mismatch risk: workflow settings timezone is `Asia/Kolkata`, but this node also forces `Asia/Kolkata`. If the incoming time includes an offset, double conversion misunderstandings are possible.
  - End time fixed to +1 hour; if you need variable duration, you must add a duration field.

---

### Block 2 — Availability Check & Immediate HTTP Response

**Overview:**  
Checks if the requested slot is free in Google Calendar, then immediately responds to the HTTP caller with success/message; only then continues to approval/booking logic.

**Nodes involved:**
- Check Availability
- Respond to Webhook
- Is Slot Available?
- End - Slot Busy

#### Node: Check Availability
- **Type / role:** `Google Calendar` (resource: `calendar`) — checks availability for a given time window.
- **Configuration choices:**
  - **Output format:** `availability`
  - **timeMin:** `{{$json.appointmentStartDate}}`
  - **timeMax:** `{{$json.appointmentEndDate}}`
  - **Calendar:** `user@example.com` (Primary)
- **Credentials:** Google Calendar OAuth2 (`Google Calendar account 2`)
- **Error handling / output behavior:**
  - `onError: continueRegularOutput` and `alwaysOutputData: true`
  - This means authentication/API errors won’t stop the workflow; you may still get output without a valid `available` flag.
- **Input/Output:**
  - Input from **Format appointment data**
  - Output to **Respond to Webhook**
- **Edge cases / failures:**
  - OAuth token expiry / insufficient scopes → availability may be missing/incorrect.
  - Calendar ID `user@example.com` must match the authenticated account or delegated access.
  - If Google API rate limits or transient errors occur, the workflow may respond incorrectly unless you explicitly handle error states.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` — returns HTTP 200 JSON to the original caller.
- **Configuration choices:**
  - **Status code:** 200
  - **Headers:** `Content-Type: application/json`
  - **Respond with:** JSON
  - **Response body (computed):**
    - `success` = `{{$json.available}}`
    - `message` = conditional:
      - If available: “Thank you… Meeting details will be emailed shortly once approved.”
      - If not available: “We cannot confirm… try again…”
- **Input/Output:**
  - Input from **Check Availability**
  - Output to **Is Slot Available?**
- **Edge cases / failures:**
  - If `available` is undefined (e.g., calendar error but continued output), `success` may be blank/falsey and message may incorrectly indicate unavailability.
  - Because the workflow responds here, any later failures (approval/booking/email) will not be visible to the original HTTP caller.

#### Node: Is Slot Available?
- **Type / role:** `IF` — branches based on availability.
- **Condition logic:**
  - Checks: `{{ $('Check Availability').item.json?.available }}` is boolean true
- **Input/Output:**
  - Input from **Respond to Webhook**
  - **True path:** to **Get Approval (HR) [1 hour wait]**
  - **False path:** to **End - Slot Busy**
- **Edge cases / failures:**
  - If the “Check Availability” node returns no `available` due to an API error, this condition will evaluate false and end as “busy” (possibly masking errors).

#### Node: End - Slot Busy
- **Type / role:** `NoOp` — explicit terminal node for busy slots.
- **Input/Output:**
  - Input from **Is Slot Available?** (false branch)
  - Output: none
- **Edge cases / failures:** none (acts as a clean stop)

---

### Block 3 — Manual Approval, Booking, and Notifications

**Overview:**  
Requests approval via Gmail with a “send and wait” action. Based on approval decision, either books the Google Calendar event or emails the requester that the request was rejected.

**Nodes involved:**
- Get Approval (HR) [1 hour wait]
- Is approved?
- Book Appointment
- Appointment Failed

#### Node: Get Approval (HR) [1 hour wait]
- **Type / role:** `Gmail` — sends an approval request and waits for a reply/approval action.
- **Operation:** `sendAndWait`
- **Configuration choices:**
  - **Send to:** `info@example.com` (approver mailbox)
  - **Subject:** `Approval Required`
  - **Message body:** includes request details and formatted time:
    - `DateTime.fromISO(appointmentStartDate).toFormat('cccc, MMMM dd @ hh:mm a')`
  - **Approval options:** `approvalType: double` (requires two-step confirmation depending on Gmail node semantics/version)
  - `appendAttribution: false`
- **Credentials:** Gmail OAuth2 (`Gmail account 2`)
- **Error handling:**
  - `onError: continueRegularOutput`
  - If Gmail API fails, workflow continues and may produce missing approval data.
- **Input/Output:**
  - Input from **Is Slot Available?** (true branch)
  - Output to **Is approved?**
- **Version-specific requirements / notes:**
  - This depends on n8n’s Gmail node capability for **approval flows** (“send and wait”). Ensure your n8n version supports this operation and that the instance can receive callbacks for the waiting mechanism.
- **Edge cases / failures:**
  - If approval is not completed within the node’s wait window (node name suggests 1 hour), the execution may time out or return a non-approved state depending on configuration/version.
  - Approver email parsing issues: if the approver replies in an unexpected format, approval status may not populate as expected.
  - If onError continues and `data.approved` is missing, the next IF will route to rejection by default.

#### Node: Is approved?
- **Type / role:** `IF` — checks whether approval was granted.
- **Condition logic:**
  - `{{$json.data.approved}}` is boolean true
- **Input/Output:**
  - Input from **Get Approval (HR) [1 hour wait]**
  - **True path:** to **Book Appointment**
  - **False path:** to **Appointment Failed**
- **Edge cases / failures:**
  - If `$json.data` is missing due to Gmail error/timeout, expression may evaluate as false → rejection path.

#### Node: Book Appointment
- **Type / role:** `Google Calendar` — creates the calendar event and invites attendees.
- **Configuration choices:**
  - **Start/End:** from **Format appointment data**:
    - `start` = `appointmentStartDate`
    - `end` = `appointmentEndDate`
  - **Calendar:** `user@example.com` (Primary)
  - **Additional fields:**
    - `summary`: “Project discovery meet with {name} at {formatted datetime}”
    - `attendees`: requester email + `info@example.com`
    - `description`: confirmation text
    - `sendUpdates: all` (sends invites/updates to attendees)
    - `conferenceSolution: hangoutsMeet` (adds Google Meet)
    - `guestsCanSeeOtherGuests: true`
    - `color: 11`
- **Credentials:** Google Calendar OAuth2 (`Google Calendar account 2`)
- **Input/Output:**
  - Input from **Is approved?** (true branch)
  - Output: none (end)
- **Edge cases / failures:**
  - If the slot becomes busy between availability check and booking, event creation may fail or create conflicts (depending on Google Calendar behavior).
  - Attendee invite sending can fail if external sharing restrictions exist.
  - Conference (Meet) creation may require specific account permissions.

#### Node: Appointment Failed
- **Type / role:** `Gmail` — notifies the requester that the appointment was rejected.
- **Configuration choices:**
  - **Send to:** requester email (pulled from *Format appointment data*’s original body path):
    - `{{ $('Format appointment data').item.json.body.email }}`
  - **Subject:** “RE: Appointment request…” with formatted date
  - **Message:** rejection explanation; BCCs `info@example.com`
  - `appendAttribution: false`
- **Credentials:** Gmail OAuth2 (`Gmail account 2`)
- **Input/Output:**
  - Input from **Is approved?** (false branch)
  - Output: none (end)
- **Edge cases / failures:**
  - If requester email is missing/invalid, Gmail send fails.
  - Deliverability: consider SPF/DKIM and “from” alignment for production.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive appointment request | Webhook | Receives appointment booking requests via HTTP POST | — | Format appointment data | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Format appointment data | Set | Normalizes fields and computes end time | Receive appointment request | Check Availability | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Check Availability | Google Calendar | Checks free/busy for requested time window | Format appointment data | Respond to Webhook | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Respond to Webhook | Respond to Webhook | Returns immediate JSON response to caller | Check Availability | Is Slot Available? | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Is Slot Available? | If | Branches on calendar availability | Respond to Webhook | Get Approval (HR) [1 hour wait]; End - Slot Busy | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Get Approval (HR) [1 hour wait] | Gmail | Sends approval request and waits for decision | Is Slot Available? | Is approved? | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Is approved? | If | Branches on approval result | Get Approval (HR) [1 hour wait] | Book Appointment; Appointment Failed | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Book Appointment | Google Calendar | Creates confirmed event + Meet link + invites | Is approved? | — | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Appointment Failed | Gmail | Sends rejection email to requester | Is approved? | — | ## Smart Appointment Scheduler / How it works… Setup steps… |
| End - Slot Busy | NoOp | Terminates flow when slot is not available | Is Slot Available? | — | ## Smart Appointment Scheduler / How it works… Setup steps… |
| Sticky Note | Sticky Note | Documentation note (non-executing) | — | — | ## Smart Appointment Scheduler… |
| Sticky Note Group 1 | Sticky Note | Documentation note (non-executing) | — | — | ## Step 1: Ingestion Receives and formats the request. |
| Sticky Note Group 2 | Sticky Note | Documentation note (non-executing) | — | — | ## Step 2: Availability Checks calendar availability and returns immediate response. |
| Sticky Note Group 3 | Sticky Note | Documentation note (non-executing) | — | — | ## Step 3: Approval & Booking Handles manual approval and booking. |

> Note: The “Sticky Note …Group” nodes are annotation-only and do not connect logically; n8n still treats them as nodes in the workflow graph.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Set **Workflow settings → Timezone** to `Asia/Kolkata` (matches the provided workflow).

2) **Add Webhook node: “Receive appointment request”**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `appointment-request`
   - Response mode: **Respond to Webhook node**
   - Options:
     - Ignore bots: enabled
     - Response header: `Content-Type: application/json`

3) **Add Set node: “Format appointment data”**
   - Node type: **Set**
   - Configure fields (string unless noted):
     - `name` = `{{ $json.body.name }}`
     - `company` = `{{ $json.body.organization }}`
     - `email` = `{{ $json.body.email }}`
     - `phone` = `{{ $json.body.phone }}`
     - `appointmentStartDate` = `{{ $json.body.appointmentDateTime }}`
     - `appointmentEndDate` = `{{ DateTime.fromISO($json.body.appointmentDateTime).setZone('Asia/Kolkata').plus(1, 'hours') }}`
     - `message` = `{{ $json.body.optionalMessage }}`
   - Keep/include original `body` content (so later nodes can reference `item.json.body.email` if desired).

4) **Connect:** Webhook → Set

5) **Add Google Calendar node: “Check Availability”**
   - Node type: **Google Calendar**
   - Resource: `calendar`
   - Operation/output option: set **Output Format = availability**
   - timeMin = `{{ $json.appointmentStartDate }}`
   - timeMax = `{{ $json.appointmentEndDate }}`
   - Calendar: select your target calendar (in the JSON it’s `user@example.com` / Primary)
   - **Credentials:** configure **Google Calendar OAuth2**
     - Ensure scopes allow reading free/busy/availability.
   - Set node option (if available): **Continue on Fail** (matches `onError: continueRegularOutput`) and/or ensure it outputs data even on error if you want identical behavior.

6) **Connect:** Set → Check Availability

7) **Add Respond to Webhook node: “Respond to Webhook”**
   - Respond with: **JSON**
   - Status: **200**
   - Header: `Content-Type: application/json`
   - Body (expression):
     ```js
     {
       "success": "{{ $json.available }}",
       "message": "{{ $json.available ? 'Thank you for your interest, Meeting details will be emailed shortly once approved.' : 'We cannot confirm the appointment at this time, Please try again with a different day/time-slot.' }}"
     }
     ```

8) **Connect:** Check Availability → Respond to Webhook

9) **Add IF node: “Is Slot Available?”**
   - Condition: Boolean true check on:
     - Left value: `{{ $('Check Availability').item.json?.available }}`
     - Operation: “is true”
   - True branch = slot free, False branch = busy

10) **Connect:** Respond to Webhook → Is Slot Available?

11) **Add NoOp node: “End - Slot Busy”**
   - Connect **Is Slot Available? (false)** → End - Slot Busy

12) **Add Gmail node: “Get Approval (HR) [1 hour wait]”**
   - Node type: **Gmail**
   - Operation: **Send and Wait** (approval flow)
   - To: `info@example.com` (your approver)
   - Subject: `Approval Required`
   - Body: include the requester details and time formatting, e.g.:
     - `{{ DateTime.fromISO($('Format appointment data').item.json.appointmentStartDate).toFormat('cccc, MMMM dd @ hh:mm a') }}`
   - Approval options: set **approval type = double** (if your n8n Gmail node supports it)
   - **Credentials:** Gmail OAuth2
     - Ensure Gmail API is enabled in Google Cloud for the OAuth client.

13) **Connect:** Is Slot Available? (true) → Get Approval (HR) [1 hour wait]

14) **Add IF node: “Is approved?”**
   - Condition: `{{ $json.data.approved }}` is true

15) **Connect:** Get Approval (HR) [1 hour wait] → Is approved?

16) **Add Google Calendar node: “Book Appointment”**
   - Operation: **Create event**
   - Start: `{{ $('Format appointment data').item.json.appointmentStartDate }}`
   - End: `{{ $('Format appointment data').item.json.appointmentEndDate }}`
   - Calendar: same as above
   - Additional fields:
     - Summary using name + formatted date
     - Attendees: requester email + `info@example.com`
     - Description: confirmation text
     - Send updates: `all`
     - Conference data: Google Meet / Hangouts Meet
     - Guests can see other guests: enabled
     - Color: 11
   - Credentials: Google Calendar OAuth2

17) **Connect:** Is approved? (true) → Book Appointment

18) **Add Gmail node: “Appointment Failed”**
   - Operation: **Send**
   - To: `{{ $('Format appointment data').item.json.body.email }}`
   - Subject and body: rejection message; add BCC `info@example.com` if desired
   - Credentials: Gmail OAuth2

19) **Connect:** Is approved? (false) → Appointment Failed

20) **(Optional but recommended) Add validation and error routing**
   - Add an IF node before “Check Availability” to ensure required fields exist (`email`, `appointmentDateTime`).
   - Add explicit handling if Google Calendar/Gmail nodes error while “continue on fail” is enabled.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Smart Appointment Scheduler**: Receives appointment request via Webhook → checks Google Calendar availability → if free requests HR approval via Gmail → sends confirmation or rejection email. | From sticky note “Smart Appointment Scheduler” |
| **Setup steps**: Configure Google Calendar and Gmail credentials; update Timezone in workflow settings; send POST request with JSON body. | From sticky note “Smart Appointment Scheduler” |
| **Step 1: Ingestion** — Receives and formats the request. | Sticky Note Group 1 |
| **Step 2: Availability** — Checks calendar availability and returns immediate response. | Sticky Note Group 2 |
| **Step 3: Approval & Booking** — Handles manual approval and booking. | Sticky Note Group 3 |