Handle Calendly bookings, cancellations and reschedules with Gmail, Google Calendar, Sheets and Slack

https://n8nworkflows.xyz/workflows/handle-calendly-bookings--cancellations-and-reschedules-with-gmail--google-calendar--sheets-and-slack-12079


# Handle Calendly bookings, cancellations and reschedules with Gmail, Google Calendar, Sheets and Slack

## 1. Workflow Overview

**Purpose:**  
This workflow receives **Calendly webhook events** (new bookings, cancellations, reschedules), then **routes** them by event category and performs consistent downstream actions: **log to Google Sheets**, **sync to Google Calendar (new bookings only)**, **notify a Slack channel**, and **send an attendee email via Gmail**.

**Primary use cases:**
- Keep a “Bookings” spreadsheet audit trail for all scheduling events.
- Automatically create a Google Calendar event when a meeting is booked.
- Notify internal teams in Slack when meetings are booked/canceled/rescheduled.
- Send branded HTML emails to attendees for confirmations and changes.

### 1.1 Input Reception & Normalization
Receives Calendly payloads and converts them into a stable internal schema (email, name, times, URLs, etc.), attempting to be compatible with multiple Calendly payload variants.

### 1.2 Event Routing
Routes events into one of three branches: **created**, **canceled**, **rescheduled**.

### 1.3 New Booking Processing
Appends a “Confirmed” row to Sheets, creates a Calendar event, posts a Slack message, and emails a confirmation.

### 1.4 Cancellation Processing
Appends a “Canceled” row to Sheets, posts a Slack message, and emails a cancellation notice.

### 1.5 Reschedule Processing
Appends a “Rescheduled” row to Sheets, posts a Slack message, and emails updated details.

### 1.6 Error Alerting (present but not fully wired)
Formats errors and sends an alert to Slack. **However, no error trigger node is connected**, so this block will not run unless you add an Error Trigger or connect it explicitly.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Webhook & Routing (Input Reception & Normalization)
**Overview:**  
Captures incoming Calendly webhook calls, parses the payload into a consistent structure, and assigns an `eventCategory` used for routing.

**Nodes involved:**
- **Calendly Webhook**
- **Parse Calendly Event**
- **Event Router**

#### Node: Calendly Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook). Entry point that receives HTTP POST requests from Calendly.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `calendly-webhook`
  - **Response:** `allEntries` (returns workflow output to caller)
  - **Webhook ID:** placeholder `YOUR_WEBHOOK_ID` (n8n internal)
- **Input / Output:**
  - Input: external HTTP request
  - Output → **Parse Calendly Event**
- **Potential failures / edge cases:**
  - Calendly signature verification is **not implemented** (no header validation). Anyone could POST unless protected by n8n auth/network controls.
  - If Calendly sends a different payload shape, downstream parsing may produce empty fields.
  - Large payloads: possible n8n payload limits depending on instance settings.

#### Node: Parse Calendly Event
- **Type / role:** `Code` node. Normalizes Calendly payload and derives event category.
- **Key configuration choices (interpreted):**
  - Tries to support **Calendly API v1 and v2** shapes by checking:
    - `payload = $input.item.json.body || $input.item.json`
    - `event = payload.event || 'invitee.created'`
    - `data = payload.payload || payload`
    - `invitee = data.invitee || data`
    - `eventDetails = data.event || data.scheduled_event || {}`
    - `questions_and_answers` extraction into `customAnswers` map.
  - Computes:
    - `eventCategory`: `"created"` default; switches to `"canceled"` if event includes “canceled”; `"rescheduled"` if includes “rescheduled”.
    - `formattedDate` and `formattedTime` using `toLocaleDateString` / `toLocaleTimeString` with `timeZone = invitee.timezone || 'UTC'`.
    - `eventId`: `CAL-${Date.now()}-${random}`
- **Key output fields produced:**
  - `eventCategory`, `eventType`, `processedAt`
  - `email`, `name`, `timezone`
  - `meetingName`, `startTime`, `endTime`, `formattedDate`, `formattedTime`
  - `meetingUrl`, `cancelUrl`, `rescheduleUrl`
  - `customAnswers` (object)
  - `eventId` (random tracking ID)
- **Input / Output:**
  - Input ← Calendly Webhook
  - Output → **Event Router**
- **Potential failures / edge cases:**
  - **Timezone formatting risk:** `toLocale*` with an invalid/unknown `timezone` string can throw or format unexpectedly depending on Node.js/ICU data.
  - **Start time parsing:** `new Date(startTime)` expects a valid ISO date/time; empty string yields `Invalid Date`.
  - **Meeting URL path:** uses `eventDetails.location?.join_url` or `invitee.meeting_url`; may be blank for in-person/phone meetings.
  - **eventCategory detection:** relies on substring matching; Calendly event names must contain `canceled`/`rescheduled` consistently.

#### Node: Event Router
- **Type / role:** `Switch` node. Routes items based on `eventCategory`.
- **Key configuration:**
  - Value1: `={{ $json.eventCategory }}`
  - Rules (string matches):
    1. `created`
    2. `canceled`
    3. `rescheduled`
- **Input / Output:**
  - Input ← Parse Calendly Event
  - Outputs:
    - `created` → **Log to Google Sheets**
    - `canceled` → **Log Cancellation**
    - `rescheduled` → **Log Reschedule**
- **Potential failures / edge cases:**
  - If `eventCategory` is anything else (or missing), **no route will match**, and the workflow effectively ends for that execution (no default output configured).

---

### Block 1.3 — New Booking (Created)
**Overview:**  
For new bookings, the workflow records the booking in Google Sheets, creates a Google Calendar event, alerts Slack, and sends a branded HTML confirmation email via Gmail.

**Nodes involved:**
- **Log to Google Sheets**
- **Create Calendar Event**
- **Slack - New Booking**
- **Prepare Confirmation Email**
- **Send Confirmation Email**
- **Done - Confirmation**

#### Node: Log to Google Sheets
- **Type / role:** `Google Sheets` append row (audit logging).
- **Key configuration:**
  - Operation: **Append**
  - Document: `YOUR_DOCUMENT_ID` (must be replaced)
  - Sheet (tab): `Bookings`
  - Columns mapped (examples):
    - Date: `{{ $json.formattedDate }}`
    - Name: `{{ $json.name }}`
    - Time: `{{ $json.formattedTime }}`
    - Email: `{{ $json.email }}`
    - Status: `"Confirmed"`
    - Event ID: `{{ $json.eventId }}`
    - Timezone, Date Logged (`processedAt`), Meeting URL, Meeting Type
- **Input / Output:**
  - Input ← Event Router (created branch)
  - Output → Create Calendar Event
- **Potential failures / edge cases:**
  - Auth/permissions to the spreadsheet.
  - Sheet tab “Bookings” must exist and column names should align with your header strategy (n8n appends by mapping; mismatches may create unexpected columns or fail depending on setup).
  - Rate limits / Google API quota.

#### Node: Create Calendar Event
- **Type / role:** `Google Calendar` create event in calendar.
- **Key configuration:**
  - Calendar: `primary`
  - Start: `={{ $('Parse Calendly Event').item.json.startTime }}`
  - End: `={{ $('Parse Calendly Event').item.json.endTime }}`
  - Summary: `={{ meetingName }} - {{ name }}`
  - Description includes attendee identity and join link.
- **Input / Output:**
  - Input ← Log to Google Sheets
  - Output → Slack - New Booking
- **Version-specific note:** node is `typeVersion: 1.2` (older calendar node variant); UI fields may differ slightly across n8n versions.
- **Potential failures / edge cases:**
  - Invalid/empty start/end times cause Google API rejection.
  - Timezone handling: times are passed as strings; ensure Calendly sends ISO strings with timezone offsets/Z.
  - Duplicates: no de-duplication; repeated webhook deliveries can create multiple calendar events.

#### Node: Slack - New Booking
- **Type / role:** `Slack` message post (team notification).
- **Key configuration:**
  - Sends text: `:calendar: *New Meeting Booked*`
  - `includeLinkToWorkflow: false`
  - Uses Slack credentials (in node it shows a `webhookId`, but actual credential is configured in n8n UI).
- **Input / Output:**
  - Input ← Create Calendar Event
  - Output → Prepare Confirmation Email
- **Potential failures / edge cases:**
  - Slack auth/webhook misconfiguration or revoked token.
  - Message lacks meeting details (only a headline). Teams may want to include name/time.

#### Node: Prepare Confirmation Email
- **Type / role:** `Code` node. Generates a branded HTML confirmation email.
- **Key configuration choices:**
  - Local `CONFIG` object (companyName/colors/logo/footer).
  - Uses data from: `$('Parse Calendly Event').first().json`
  - Builds:
    - `to = data.email`
    - `subject = Meeting Confirmed - <date> at <time>`
    - `htmlContent` (full HTML email)
- **Input / Output:**
  - Input ← Slack - New Booking
  - Output → Send Confirmation Email
- **Potential failures / edge cases:**
  - If `data.email` is empty, Gmail node will fail.
  - If `meetingUrl`/cancel/reschedule URLs are empty, sections/buttons are conditionally omitted (handled safely).
  - HTML size could be large but typically acceptable.

#### Node: Send Confirmation Email
- **Type / role:** `Gmail` send email.
- **Key configuration:**
  - To: `={{ $json.to }}`
  - Subject: `={{ $json.subject }}`
  - Message/body: `={{ $json.htmlContent }}`
  - Option: `appendAttribution: false`
- **Input / Output:**
  - Input ← Prepare Confirmation Email
  - Output → Done - Confirmation
- **Potential failures / edge cases:**
  - Gmail OAuth not connected, insufficient scopes, or sending limits.
  - Some Gmail configurations may treat HTML in “message” depending on node behavior/version; verify it sends as intended HTML.

#### Node: Done - Confirmation
- **Type / role:** `NoOp` terminator.
- **Input / Output:** Input ← Send Confirmation Email; no outputs.
- **Notes:** purely visual/end marker.

---

### Block 1.4 — Cancellation
**Overview:**  
For cancellations, the workflow appends a “Canceled” row to the same sheet, alerts Slack, and emails the attendee a cancellation notice including a link to book again.

**Nodes involved:**
- **Log Cancellation**
- **Slack - Cancellation**
- **Prepare Cancellation Email**
- **Send Cancellation Email**
- **Done - Cancellation**

#### Node: Log Cancellation
- **Type / role:** `Google Sheets` append row.
- **Key configuration:**
  - Same spreadsheet/tab: Document `YOUR_DOCUMENT_ID`, Sheet `Bookings`
  - Status: `"Canceled"`
  - Uses expressions referencing `$('Parse Calendly Event').item.json...` (not `$json`).
- **Input / Output:** Input ← Event Router (canceled) → Output → Slack - Cancellation
- **Potential failures / edge cases:** same as other Sheets nodes (auth, missing tab, quotas).

#### Node: Slack - Cancellation
- **Type / role:** Slack message post.
- **Key configuration:** `:x: *Meeting Canceled*`
- **Input / Output:** Input ← Log Cancellation → Output → Prepare Cancellation Email
- **Potential failures:** Slack auth/config.

#### Node: Prepare Cancellation Email
- **Type / role:** `Code` node generating HTML cancellation email.
- **Key configuration choices:**
  - CONFIG includes `calendlyUrl: 'https://calendly.com/your-username'` (must be customized).
  - Uses `$('Parse Calendly Event').first().json`
  - Output: `{ to, subject, htmlContent }`
- **Input / Output:** Input ← Slack - Cancellation → Output → Send Cancellation Email
- **Potential failures / edge cases:**
  - Missing email address.
  - Calendly booking link not updated → wrong CTA.

#### Node: Send Cancellation Email
- **Type / role:** Gmail send.
- **Key configuration:** to/subject/message from previous node; attribution disabled.
- **Input / Output:** Input ← Prepare Cancellation Email → Output → Done - Cancellation

#### Node: Done - Cancellation
- **Type / role:** NoOp terminator.

---

### Block 1.5 — Reschedule
**Overview:**  
For reschedules, the workflow appends a “Rescheduled” row, alerts Slack, and emails new meeting details.

**Nodes involved:**
- **Log Reschedule**
- **Slack - Rescheduled**
- **Prepare Reschedule Email**
- **Send Reschedule Email**
- **Done - Reschedule**

#### Node: Log Reschedule
- **Type / role:** Google Sheets append row.
- **Key configuration:** same spreadsheet/tab; Status: `"Rescheduled"`.
- **Input / Output:** Input ← Event Router (rescheduled) → Output → Slack - Rescheduled

#### Node: Slack - Rescheduled
- **Type / role:** Slack message post.
- **Key configuration:** `:arrows_counterclockwise: *Meeting Rescheduled*`
- **Input / Output:** Input ← Log Reschedule → Output → Prepare Reschedule Email

#### Node: Prepare Reschedule Email
- **Type / role:** Code node generating HTML reschedule email.
- **Key configuration:** CONFIG defines companyName and colors; uses parsed data.
- **Input / Output:** Input ← Slack - Rescheduled → Output → Send Reschedule Email
- **Edge cases:** missing start time/timezone leading to confusing formatted output; missing URLs are conditionally omitted.

#### Node: Send Reschedule Email
- **Type / role:** Gmail send.
- **Input / Output:** Input ← Prepare Reschedule Email → Output → Done - Reschedule

#### Node: Done - Reschedule
- **Type / role:** NoOp terminator.

---

### Block 1.6 — Error Handling (Not Active by Default)
**Overview:**  
Formats error information and posts a Slack alert. As provided, it is **not connected** to an error trigger, so it will not execute automatically.

**Nodes involved:**
- **Format Error1**
- **Slack - Error Alert1**

#### Node: Format Error1
- **Type / role:** Code node to normalize error payload into a Slack-friendly structure.
- **Key configuration:**
  - Reads error from `$input.item.json`
  - Outputs: `errorTime`, `errorMessage`, `errorNode`, `workflowName`, `severity`
- **Input / Output:** Output → Slack - Error Alert1
- **Edge cases:** error structure differs depending on trigger type; `error.node?.name` may be missing.

#### Node: Slack - Error Alert1
- **Type / role:** Slack message post for errors.
- **Key configuration:**
  - Text: `:rotating_light: *Workflow Error*`
  - `includeLinkToWorkflow: true`
- **Input / Output:** Input ← Format Error1; no further outputs.
- **Missing connection note:** to activate, add an **Error Trigger** node and connect it to Format Error1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Calendly Webhook | Webhook | Receives Calendly POST webhooks | — | Parse Calendly Event | ## Webhook & routing<br>Receives Calendly events and routes by type. |
| Parse Calendly Event | Code | Normalizes Calendly payload; derives eventCategory and fields | Calendly Webhook | Event Router | ## Webhook & routing<br>Receives Calendly events and routes by type. |
| Event Router | Switch | Routes to created/canceled/rescheduled branches | Parse Calendly Event | Log to Google Sheets; Log Cancellation; Log Reschedule | ## Webhook & routing<br>Receives Calendly events and routes by type. |
| Log to Google Sheets | Google Sheets | Append “Confirmed” booking row | Event Router (created) | Create Calendar Event | ## New booking<br>Logs, syncs calendar, notifies Slack, sends confirmation. |
| Create Calendar Event | Google Calendar | Create calendar event for new booking | Log to Google Sheets | Slack - New Booking | ## New booking<br>Logs, syncs calendar, notifies Slack, sends confirmation. |
| Slack - New Booking | Slack | Notify team of new booking | Create Calendar Event | Prepare Confirmation Email | ## New booking<br>Logs, syncs calendar, notifies Slack, sends confirmation. |
| Prepare Confirmation Email | Code | Generate branded HTML confirmation email | Slack - New Booking | Send Confirmation Email | ## New booking<br>Logs, syncs calendar, notifies Slack, sends confirmation. |
| Send Confirmation Email | Gmail | Send confirmation email | Prepare Confirmation Email | Done - Confirmation | ## New booking<br>Logs, syncs calendar, notifies Slack, sends confirmation. |
| Done - Confirmation | NoOp | End marker for booking path | Send Confirmation Email | — | ## New booking<br>Logs, syncs calendar, notifies Slack, sends confirmation. |
| Log Cancellation | Google Sheets | Append “Canceled” row | Event Router (canceled) | Slack - Cancellation | ## Cancellation<br>Logs cancellation, alerts team, notifies attendee. |
| Slack - Cancellation | Slack | Notify team of cancellation | Log Cancellation | Prepare Cancellation Email | ## Cancellation<br>Logs cancellation, alerts team, notifies attendee. |
| Prepare Cancellation Email | Code | Generate cancellation email with rebook CTA | Slack - Cancellation | Send Cancellation Email | ## Cancellation<br>Logs cancellation, alerts team, notifies attendee. |
| Send Cancellation Email | Gmail | Send cancellation email | Prepare Cancellation Email | Done - Cancellation | ## Cancellation<br>Logs cancellation, alerts team, notifies attendee. |
| Done - Cancellation | NoOp | End marker for cancellation path | Send Cancellation Email | — | ## Cancellation<br>Logs cancellation, alerts team, notifies attendee. |
| Log Reschedule | Google Sheets | Append “Rescheduled” row | Event Router (rescheduled) | Slack - Rescheduled | ## Reschedule<br>Logs changes, alerts team, sends updated details. |
| Slack - Rescheduled | Slack | Notify team of reschedule | Log Reschedule | Prepare Reschedule Email | ## Reschedule<br>Logs changes, alerts team, sends updated details. |
| Prepare Reschedule Email | Code | Generate reschedule email | Slack - Rescheduled | Send Reschedule Email | ## Reschedule<br>Logs changes, alerts team, sends updated details. |
| Send Reschedule Email | Gmail | Send reschedule email | Prepare Reschedule Email | Done - Reschedule | ## Reschedule<br>Logs changes, alerts team, sends updated details. |
| Done - Reschedule | NoOp | End marker for reschedule path | Send Reschedule Email | — | ## Reschedule<br>Logs changes, alerts team, sends updated details. |
| Format Error1 | Code | Format error details for alerting | (not connected) | Slack - Error Alert1 | ## Error handling<br>Catches errors and alerts #errors channel. |
| Slack - Error Alert1 | Slack | Post workflow error alert | Format Error1 | — | ## Error handling<br>Catches errors and alerts #errors channel. |
| Webhook | Sticky Note | Comment block | — | — |  |
| New Booking | Sticky Note | Comment block | — | — |  |
| Cancellation | Sticky Note | Comment block | — | — |  |
| Reschedule | Sticky Note | Comment block | — | — |  |
| Overview1 | Sticky Note | Comment block (overview + setup steps) | — | — |  |
| Errors1 | Sticky Note | Comment block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Webhook node**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `calendly-webhook`
   - Response: **All Entries**
   - Save and copy the **Production URL** (or Test URL for initial testing).

3. **Configure Calendly webhook**
   - In Calendly (developer/webhooks area), create a webhook pointing to the n8n webhook URL.
   - Subscribe to events for **created**, **canceled**, **rescheduled** (names depend on your Calendly webhook settings).
   - (Recommended) Enable signing/verification on Calendly side and validate signatures in n8n (not implemented in this workflow).

4. **Add Code node: “Parse Calendly Event”**
   - Paste logic that:
     - Reads request body (`$input.item.json.body || $input.item.json`)
     - Extracts invitee + event details
     - Derives `eventCategory` (`created` / `canceled` / `rescheduled`)
     - Outputs normalized fields: `email`, `name`, `startTime`, `endTime`, `timezone`, `meetingUrl`, `cancelUrl`, `rescheduleUrl`, `meetingName`, `formattedDate`, `formattedTime`, `processedAt`, `eventId`, `customAnswers`
   - Connect: **Webhook → Parse Calendly Event**

5. **Add Switch node: “Event Router”**
   - Value to evaluate: `{{ $json.eventCategory }}`
   - Create 3 rules (string equals):
     - `created`
     - `canceled`
     - `rescheduled`
   - Connect: **Parse Calendly Event → Event Router**

---

### Build the “created” branch (New booking)

6. **Add Google Sheets node: “Log to Google Sheets”**
   - Operation: **Append**
   - Credentials: Google Sheets OAuth2 (connect in n8n Credentials)
   - Document: select your spreadsheet (replace placeholder `YOUR_DOCUMENT_ID`)
   - Sheet/tab name: `Bookings` (must exist)
   - Map columns:
     - Date = `{{ $json.formattedDate }}`
     - Name = `{{ $json.name }}`
     - Time = `{{ $json.formattedTime }}`
     - Email = `{{ $json.email }}`
     - Status = `Confirmed`
     - Event ID = `{{ $json.eventId }}`
     - Timezone = `{{ $json.timezone }}`
     - Date Logged = `{{ $json.processedAt }}`
     - Meeting URL = `{{ $json.meetingUrl }}`
     - Meeting Type = `{{ $json.meetingName }}`
   - Connect: **Event Router (created output) → Log to Google Sheets**

7. **Add Google Calendar node: “Create Calendar Event”**
   - Operation: create event
   - Calendar: `primary`
   - Start: `{{ $('Parse Calendly Event').item.json.startTime }}`
   - End: `{{ $('Parse Calendly Event').item.json.endTime }}`
   - Summary: `{{ meetingName }} - {{ name }}`
   - Description: include attendee and join link
   - Credentials: Google Calendar OAuth2
   - Connect: **Log to Google Sheets → Create Calendar Event**

8. **Add Slack node: “Slack - New Booking”**
   - Operation: post message (channel configured in Slack credentials/node settings)
   - Text: `:calendar: *New Meeting Booked*`
   - Connect: **Create Calendar Event → Slack - New Booking**

9. **Add Code node: “Prepare Confirmation Email”**
   - Build HTML email using parsed data
   - Output JSON must include:
     - `to`
     - `subject`
     - `htmlContent`
   - Connect: **Slack - New Booking → Prepare Confirmation Email**

10. **Add Gmail node: “Send Confirmation Email”**
    - Operation: send
    - To: `{{ $json.to }}`
    - Subject: `{{ $json.subject }}`
    - Message: `{{ $json.htmlContent }}`
    - Disable attribution append (optional)
    - Credentials: Gmail OAuth2
    - Connect: **Prepare Confirmation Email → Send Confirmation Email**

11. **Add NoOp node: “Done - Confirmation”**
    - Connect: **Send Confirmation Email → Done - Confirmation**

---

### Build the “canceled” branch

12. **Add Google Sheets node: “Log Cancellation”**
    - Append row, same document/tab `Bookings`
    - Status = `Canceled`
    - Use expressions from Parse node (either `$json` if connected directly, or explicit `$('Parse Calendly Event')...`)
    - Connect: **Event Router (canceled output) → Log Cancellation**

13. **Add Slack node: “Slack - Cancellation”**
    - Text: `:x: *Meeting Canceled*`
    - Connect: **Log Cancellation → Slack - Cancellation**

14. **Add Code node: “Prepare Cancellation Email”**
    - Ensure `CONFIG.calendlyUrl` is your booking link
    - Output: `to`, `subject`, `htmlContent`
    - Connect: **Slack - Cancellation → Prepare Cancellation Email**

15. **Add Gmail node: “Send Cancellation Email”**
    - To/Subject/Message from previous node
    - Connect: **Prepare Cancellation Email → Send Cancellation Email**

16. **Add NoOp node: “Done - Cancellation”**
    - Connect: **Send Cancellation Email → Done - Cancellation**

---

### Build the “rescheduled” branch

17. **Add Google Sheets node: “Log Reschedule”**
    - Append row, Status = `Rescheduled`
    - Connect: **Event Router (rescheduled output) → Log Reschedule**

18. **Add Slack node: “Slack - Rescheduled”**
    - Text: `:arrows_counterclockwise: *Meeting Rescheduled*`
    - Connect: **Log Reschedule → Slack - Rescheduled**

19. **Add Code node: “Prepare Reschedule Email”**
    - Output: `to`, `subject`, `htmlContent`
    - Connect: **Slack - Rescheduled → Prepare Reschedule Email**

20. **Add Gmail node: “Send Reschedule Email”**
    - Connect: **Prepare Reschedule Email → Send Reschedule Email**

21. **Add NoOp node: “Done - Reschedule”**
    - Connect: **Send Reschedule Email → Done - Reschedule**

---

### Optional: activate error alerting properly

22. **Add “Error Trigger” node** (recommended)
    - Node type: **Error Trigger**
    - Connect: **Error Trigger → Format Error1 → Slack - Error Alert1**
    - Configure Slack to post in `#errors` (or desired channel).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automate Calendly bookings with Gmail, Google Calendar, Sheets and Slack. Syncs Calendly events to your calendar, logs to Sheets, sends confirmations, and alerts your team. | Sticky note “Overview1” |
| Setup steps: Calendly webhook → Connect Gmail/Calendar/Sheets/Slack → Create sheet tab “Bookings” → Replace YOUR_DOCUMENT_ID → Configure Slack channels (#bookings and #errors) → Test with a booking. | Sticky note “Overview1” |
| Webhook & routing: Receives Calendly events and routes by type. | Sticky note “Webhook” |
| New booking: Logs, syncs calendar, notifies Slack, sends confirmation. | Sticky note “New Booking” |
| Cancellation: Logs cancellation, alerts team, notifies attendee. | Sticky note “Cancellation” |
| Reschedule: Logs changes, alerts team, sends updated details. | Sticky note “Reschedule” |
| Error handling: Catches errors and alerts #errors channel. (Not connected by default—add Error Trigger.) | Sticky note “Errors1” |