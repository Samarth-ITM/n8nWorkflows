Schedule client meetings via web forms with Google Calendar, Zoom and multi‑channel notifications

https://n8nworkflows.xyz/workflows/schedule-client-meetings-via-web-forms-with-google-calendar--zoom-and-multi-channel-notifications-13579


# Schedule client meetings via web forms with Google Calendar, Zoom and multi‑channel notifications

## 1. Workflow Overview

**Purpose:** Automate client meeting scheduling from a web form. The workflow receives a booking request, formats/derives meeting times, checks Google Calendar availability, then either (a) books the meeting and notifies the client/team through multiple channels or (b) informs the client the slot is unavailable.

**Primary use cases:**
- Website “Book a meeting” form submissions.
- Real-time calendar conflict checking before confirming a meeting.
- Automatic Google Calendar event creation + optional Zoom meeting creation + multi-channel notifications (email, WhatsApp, Teams, Discord).

### 1.1 Input Reception & Date Formatting
Receives POST data via webhook and formats the submitted date for downstream messaging.

### 1.2 Meeting Defaults + Booking Window Computation
Sets defaults (title, duration, password) and computes ISO start/end timestamps from submitted date/time.

### 1.3 Calendar Availability Check
Queries Google Calendar events in the requested interval and branches based on whether conflicts exist.

### 1.4 Booking Creation (Calendar → Zoom) & Confirmation
Creates the Google Calendar event, then creates a Zoom meeting (configured but note edge cases), then emails the client confirmation.

### 1.5 Internal/Owner Notifications
After confirmation email, sends notifications via Teams chat message, Discord DM, WhatsApp via Rapiwa, and an additional Gmail notice.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Date Formatting
**Overview:** Accepts a booking request from a web form (POST) and formats the submitted date for consistent display in messages.

**Nodes involved:**
- **Web Contact Form**
- **Format Date & Time**

#### Node: Web Contact Form
- **Type / Role:** `Webhook` trigger; entry point.
- **Config choices:**
  - **HTTP Method:** POST
  - **Path:** `/forms`
  - Expects form fields in `body` (example pinned): `date`, `time`, `name`, `email`, `message`
- **Inputs/Outputs:**
  - **Input:** External HTTP request
  - **Output:** One item containing `json.body`, `json.headers`, etc.
- **Edge cases / failures:**
  - Missing `body.date` or `body.time` will later produce `null` booking timestamps.
  - If the form is `application/x-www-form-urlencoded` (as shown), ensure n8n parses it properly (it usually does).
  - Public webhook requires validation/anti-spam in real deployments (not present here).
- **Version notes:** Webhook node v2.

#### Node: Format Date & Time
- **Type / Role:** `Date & Time` formatter.
- **Config choices:**
  - **Operation:** Format Date
  - **Date input:** `{{$json.body.date}}`
- **Key outputs:**
  - Produces `formattedDate` used in emails/notifications.
- **Connections:**
  - **Input:** Web Contact Form
  - **Output:** Add Event name, Duration, Password
- **Edge cases / failures:**
  - If `body.date` is not a valid date string, formatting may fail or output an invalid value.
- **Version notes:** DateTime node v2.

---

### Block 1.2 — Meeting Defaults + Booking Window Computation
**Overview:** Sets default meeting metadata and calculates the meeting start/end timestamps (ISO strings) from the submitted date/time and configured duration.

**Nodes involved:**
- **Add Event name, Duration, Password**
- **Code (Configure Meeting Information)**

#### Node: Add Event name, Duration, Password
- **Type / Role:** `Set` node; defines defaults/constants.
- **Config choices (fields added):**
  - `event_title`: `"Web Form Meeting Booking"`
  - `meeting_time`: `40` (minutes)
  - `meeting_password`: `"add_your_password"`
- **Connections:**
  - **Input:** Format Date & Time
  - **Output:** Code (Configure Meeting Information)
- **Edge cases / failures:**
  - None typical; but “meeting_password” is not actually reused consistently (Zoom node uses a hardcoded placeholder).
- **Version notes:** Set node v3.4.

#### Node: Code (Configure Meeting Information)
- **Type / Role:** `Code` node; parses webhook body and computes booking window.
- **What it does (interpreted):**
  - Reads `body` from **Web Contact Form** directly: `$('Web Contact Form').first().json.body`
  - Uses `$input.first().json.meeting_time` (from the Set node) as duration fallback to 30 minutes if missing.
  - Builds start time using: `new Date(`${date}T${time}:00+06:00`)`
    - Hard-coded offset `+06:00` (Asia/Dhaka-style offset).
  - Computes:
    - `booking_start` (ISO string)
    - `booking_end` (ISO string = start + duration minutes)
    - `duration_time` (string like `"40 minutes"`)
  - Outputs fields:
    - `client_name`, `client_email`, `client_messate` (note spelling), `booking_date`, `booking_time`, `booking_start`, `booking_end`, `duration_time`
- **Connections:**
  - **Input:** Add Event name, Duration, Password
  - **Output:** Checks Google Calendar to see if the requested time slot is free
- **Key variables/expressions:**
  - Uses node cross-references: `$('Web Contact Form')...`
  - Uses `$input.first().json.meeting_time`
- **Edge cases / failures:**
  - If `date` or `time` missing → start/end remain `null`, causing downstream Google Calendar nodes to error.
  - If submitted time is not `HH:MM` → `new Date()` may be invalid.
  - Time zone handling: hard-coded `+06:00` may conflict with user-provided timezone assumptions.
  - Typo field `client_messate` is used later in Calendar event description; changing it will require updating expressions.
- **Version notes:** Code node v2.

---

### Block 1.3 — Calendar Availability Check
**Overview:** Queries Google Calendar for events overlapping the requested time window and branches to either booking flow or rejection email.

**Nodes involved:**
- **Checks Google Calendar to see if the requested time slot is free**
- **Check Meeting Slot Availability**
- **Send Notice for Unavailable Slot**

#### Node: Checks Google Calendar to see if the requested time slot is free
- **Type / Role:** `Google Calendar` node; checks conflicts by listing events in a time range.
- **Config choices:**
  - **Operation:** Get All events
  - **Calendar:** `user@example.com` (configured via resource locator list)
  - **returnAll:** true
  - **Time range parameters (as configured):**
    - `timeMax = {{$json.booking_start}}`
    - `timeMin = {{$json.booking_end}}`
- **Important logic note (potential bug):**
  - Typically **timeMin should be start** and **timeMax should be end**. Here they are reversed (`timeMin=end`, `timeMax=start`), which can yield empty results or API errors depending on Google behavior.
- **Connections:**
  - **Input:** Code (Configure Meeting Information)
  - **Output:** Check Meeting Slot Availability
- **Edge cases / failures:**
  - OAuth/permission errors (calendar scope, wrong account).
  - Invalid `timeMin/timeMax` (null or reversed).
  - Empty results are forced through because **alwaysOutputData=true**, so downstream IF must correctly interpret outputs.
- **Version notes:** Google Calendar node v1.3, `alwaysOutputData` enabled.

#### Node: Check Meeting Slot Availability
- **Type / Role:** `IF` node; decides whether slot is available.
- **Config choices:**
  - Condition checks whether the incoming `$json` object is **empty**:
    - `leftValue: {{$json}}` with operator “empty”
  - Interpreted intent: “If no events returned → slot free”.
- **Connections:**
  - **Input:** Checks Google Calendar to see if the requested time slot is free
  - **True output (slot available):** Create Event Using Web Submission Form...
  - **False output (slot unavailable):** Send Notice for Unavailable Slot
- **Edge cases / failures:**
  - Google Calendar “Get All” returns arrays/items; emptiness detection may not behave as intended if structure differs (e.g., always a non-empty object wrapper).
  - If `timeMin/timeMax` are reversed and returns empty erroneously → false negatives (double-bookings).
- **Version notes:** IF node v2.2.

#### Node: Send Notice for Unavailable Slot
- **Type / Role:** `Gmail` node; rejection email to client.
- **Config choices:**
  - **To:** `{{ $('Code (Configure Meeting Information)').item.json.client_email }}`
  - **Subject:** `Time slot unavailable.`
  - **Body:** includes formatted date/time, booking_time, client_name; uses `formattedDate.toLocaleString('ja-JP', { timeZone: 'Asia/Dhaka' })`
- **Connections:**
  - **Input:** IF (false branch)
  - **Output:** none
- **Edge cases / failures:**
  - Missing/invalid client email → Gmail send failure.
  - Locale/timezone formatting: `formattedDate` may be a string, not a Date object; calling `.toLocaleString()` can error if not a Date type.
  - Gmail API quota/auth issues.
- **Version notes:** Gmail node v2.1.

---

### Block 1.4 — Booking Creation (Calendar → Zoom) & Client Confirmation
**Overview:** If the slot is available, it creates the Google Calendar event, then creates a Zoom meeting, then sends a confirmation email to the client.

**Nodes involved:**
- **Create Event Using Web Submission Form (Add booking to Google Calendar with details & reminders)**
- **Create Zoom meeting with client info**
- **Send confirmation email to client**
- **Notify team via Microsoft Teams** (runs in parallel after Zoom creation, before confirmation notifications fan-out)

#### Node: Create Event Using Web Submission Form (Add booking to Google Calendar with details & reminders)
- **Type / Role:** `Google Calendar` node; creates the event.
- **Config choices:**
  - **Calendar:** `user@example.com`
  - **Start:** `{{ $('Code (Configure Meeting Information)').item.json.booking_start }}`
  - **End:** `{{ $('Code (Configure Meeting Information)').item.json.booking_end }}`
  - **Summary:** `{{ $('Add Event name, Duration, Password').item.json.event_title }}`
  - **Description:** includes client name and `client_messate`
  - **Reminders:** email 10 minutes; popup 5 minutes; default reminders disabled
  - **Color:** `"3"`
- **Connections:**
  - **Input:** IF (true branch)
  - **Output:** Create Zoom meeting with client info
- **Edge cases / failures:**
  - Invalid start/end (null) → Google rejects.
  - Permission issues for calendar write.
  - Double-booking risk if availability check logic is wrong.
- **Version notes:** Google Calendar node v1.3.

#### Node: Create Zoom meeting with client info
- **Type / Role:** `Zoom` node; creates a Zoom meeting.
- **Config choices:**
  - **Topic:** `Booking with <client_name>`
  - **Start time:** `booking_start`
  - **Time zone:** `Asia/Dhaka`
  - **Duration:** set to `duration_time` (string like `"40 minutes"`)
  - **Password:** hardcoded placeholder `"YOUR_CREDENTIAL_HERE"` (not using the Set node’s `meeting_password`)
  - **Meeting type:** `type: 1` (instant meeting in Zoom API terms; scheduled meetings are usually type 2)
- **Connections:**
  - **Input:** Create Event Using Web Submission Form...
  - **Outputs:** Send confirmation email to client **and** Notify team via Microsoft Teams (parallel)
- **Edge cases / failures:**
  - **Duration format mismatch:** Zoom typically expects duration as an integer number of minutes, not `"40 minutes"`.
  - **Type mismatch:** If you want scheduled meetings, type likely should be 2; otherwise `startTime` may be ignored.
  - Password placeholder will cause meeting creation failure unless replaced.
  - OAuth/JWT credential configuration errors.
- **Version notes:** Zoom node v1.

#### Node: Send confirmation email to client
- **Type / Role:** `Gmail` node; sends booking confirmation to the client.
- **Config choices:**
  - **To:** client email from Code node
  - **Subject:** includes formatted date/time
  - **Body:** includes date/time and `{{$json.join_url}}` from Zoom output
- **Connections:**
  - **Input:** Create Zoom meeting with client info
  - **Output:** Fans out to Create chat message, Send a Email, Send a message, Rapiwa ()WhatsApp Message
- **Edge cases / failures:**
  - If Zoom meeting fails or doesn’t output `join_url`, email will have a blank link or expression failure.
  - `.toLocaleString(...)` on `formattedDate` may fail depending on type (see earlier).
- **Version notes:** Gmail node v2.1.

#### Node: Notify team via Microsoft Teams
- **Type / Role:** `Microsoft Teams` channel message; internal notification.
- **Config choices:**
  - **Resource:** channelMessage
  - **Team ID / Channel ID:** currently empty (needs selection)
  - Message includes client name/date/time and Zoom `join_url`
- **Connections:**
  - **Input:** Create Zoom meeting with client info
  - **Output:** none
- **Edge cases / failures:**
  - Missing team/channel IDs → node will fail at runtime.
  - OAuth permission/scopes.
- **Version notes:** Microsoft Teams node v2.

---

### Block 1.5 — Internal/Owner Notifications (Post-confirmation fan-out)
**Overview:** After the client confirmation email is sent, the workflow notifies the owner/team across multiple channels (Teams chat, Gmail, Discord, WhatsApp).

**Nodes involved:**
- **Create chat message**
- **Send a Email**
- **Send a message**
- **Rapiwa ()WhatsApp Message**

#### Node: Create chat message
- **Type / Role:** `Microsoft Teams` chat message (not channel).
- **Config choices:**
  - **Resource:** chatMessage
  - **Chat ID:** empty (needs selection)
  - Message includes client/date/time and Zoom join URL
- **Connections:**
  - **Input:** Send confirmation email to client
  - **Output:** none
- **Edge cases / failures:**
  - Missing chatId → failure.
  - OAuth issues.
- **Version notes:** Microsoft Teams node v2.

#### Node: Send a Email
- **Type / Role:** `Gmail` node; sends an internal notification email.
- **Config choices:**
  - **To:** hardcoded `"your_mail_address"`
  - Body includes client/date/time and Zoom join URL
- **Connections:**
  - **Input:** Send confirmation email to client
  - **Output:** none
- **Edge cases / failures:**
  - Invalid recipient.
  - If Zoom did not create `join_url`, message is incomplete.
- **Version notes:** Gmail node v2.1.

#### Node: Send a message
- **Type / Role:** `Discord` node; sends a DM to a user.
- **Config choices:**
  - **sendTo:** user
  - **userId:** `"your_user_id"` (placeholder)
  - **guildId:** `"your_server_id"` (placeholder; not always required for DMs depending on implementation)
  - Content includes client/date/time and Zoom join URL
- **Connections:**
  - **Input:** Send confirmation email to client
  - **Output:** none
- **Edge cases / failures:**
  - Wrong userId/guildId, bot not permitted, DM closed.
  - Rate limits.
- **Version notes:** Discord node v2.

#### Node: Rapiwa ()WhatsApp Message
- **Type / Role:** `Rapiwa` community node; sends WhatsApp message.
- **Config choices:**
  - **number:** `"add_your_whatsapp_number"`
  - Message includes client/date/time and Zoom join URL
- **Connections:**
  - **Input:** Send confirmation email to client
  - **Output:** none
- **Edge cases / failures:**
  - Invalid number format, API key invalid, provider downtime.
- **Version notes:** Rapiwa node v1.

---

### Sticky Notes (documentation-only nodes)
Sticky notes do not execute but provide context. They “cover” groups of nodes visually; their content is replicated in the summary table where applicable.

- **Sticky Note1:** Full workflow overview, credentials, customization, useful links.
- **Sticky Note2:** Web form booking + calendar check explanation.
- **Sticky Note4:** “Notifications & Meeting” header for the booking/notification area.
- **Sticky Note3:** “notification after work is completed” (applies to the post-confirmation notification fan-out area).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Web Contact Form | Webhook | Receives web form booking submission | — | Format Date & Time | ## Overview … (see Sticky Note1 for full content); # Web Form Meeting Booking & Calendar Check… |
| Format Date & Time | Date & Time | Formats submitted date for messaging | Web Contact Form | Add Event name, Duration, Password | # Web Form Meeting Booking & Calendar Check… |
| Add Event name, Duration, Password | Set | Sets default title/duration/password | Format Date & Time | Code (Configure Meeting Information) | # Web Form Meeting Booking & Calendar Check… |
| Code (Configure Meeting Information) | Code | Extracts client info; computes booking start/end ISO timestamps | Add Event name, Duration, Password | Checks Google Calendar to see if the requested time slot is free | # Web Form Meeting Booking & Calendar Check… |
| Checks Google Calendar to see if the requested time slot is free | Google Calendar | Lists events in time range to detect conflicts | Code (Configure Meeting Information) | Check Meeting Slot Availability | # Web Form Meeting Booking & Calendar Check… |
| Check Meeting Slot Availability | IF | Branches based on whether returned events are empty | Checks Google Calendar to see if the requested time slot is free | (true) Create Event Using Web Submission Form…; (false) Send Notice for Unavailable Slot | # Web Form Meeting Booking & Calendar Check… |
| Send Notice for Unavailable Slot | Gmail | Emails client that slot is unavailable | Check Meeting Slot Availability (false) | — | # Web Form Meeting Booking & Calendar Check… |
| Create Event Using Web Submission Form (Add booking to Google Calendar with details & reminders) | Google Calendar | Creates the booking event in Google Calendar | Check Meeting Slot Availability (true) | Create Zoom meeting with client info | # Notifications & Meeting |
| Create Zoom meeting with client info | Zoom | Creates Zoom meeting and returns join_url | Create Event Using Web Submission Form… | Send confirmation email to client; Notify team via Microsoft Teams | # Notifications & Meeting |
| Notify team via Microsoft Teams | Microsoft Teams | Posts message to Teams channel | Create Zoom meeting with client info | — | # Notifications & Meeting |
| Send confirmation email to client | Gmail | Sends client confirmation with Zoom link | Create Zoom meeting with client info | Create chat message; Send a Email; Send a message; Rapiwa ()WhatsApp Message | # Notifications & Meeting |
| Create chat message | Microsoft Teams | Sends Teams chat message | Send confirmation email to client | — | **notification after work is completed** |
| Send a Email | Gmail | Sends internal email notification | Send confirmation email to client | — | **notification after work is completed** |
| Send a message | Discord | Sends Discord DM notification | Send confirmation email to client | — | **notification after work is completed** |
| Rapiwa ()WhatsApp Message | Rapiwa | Sends WhatsApp notification | Send confirmation email to client | — | **notification after work is completed** |
| Sticky Note1 | Sticky Note | Documentation | — | — |  |
| Sticky Note2 | Sticky Note | Documentation | — | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — |  |
| Sticky Note4 | Sticky Note | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Automated Client Meeting Scheduling & Instant Notifications**
   - Keep it inactive until credentials and IDs are configured.

2. **Add trigger: Webhook**
   - Node: **Webhook** named **Web Contact Form**
   - Method: **POST**
   - Path: **forms**
   - Decide whether you want “Test” or “Production” URL in your form action.

3. **Add Date formatting**
   - Node: **Date & Time** named **Format Date & Time**
   - Operation: **Format Date**
   - Date field expression: `{{$json.body.date}}`
   - Connect: **Web Contact Form → Format Date & Time**

4. **Set meeting defaults**
   - Node: **Set** named **Add Event name, Duration, Password**
   - Add fields:
     - `event_title` (string): `Web Form Meeting Booking`
     - `meeting_time` (number): `40`
     - `meeting_password` (string): set your password (or remove if not needed)
   - Connect: **Format Date & Time → Add Event name, Duration, Password**

5. **Compute booking start/end**
   - Node: **Code** named **Code (Configure Meeting Information)**
   - Paste logic equivalent to:
     - Read `date`, `time`, `name`, `email`, `message` from `Web Contact Form` body
     - Create `booking_start` and `booking_end` as ISO strings using a timezone offset (currently `+06:00`)
     - Output: `client_name`, `client_email`, `client_messate`, `booking_start`, `booking_end`, etc.
   - Connect: **Add Event name, Duration, Password → Code (Configure Meeting Information)**

6. **Check Google Calendar conflicts**
   - Node: **Google Calendar** named **Checks Google Calendar to see if the requested time slot is free**
   - Operation: **Get All**
   - Calendar: choose your calendar (e.g., your primary)
   - Set **TimeMin** to booking start and **TimeMax** to booking end (recommended):
     - `timeMin = {{$json.booking_start}}`
     - `timeMax = {{$json.booking_end}}`
   - Enable returning all if desired.
   - Credentials: **Google Calendar OAuth2**
   - Connect: **Code → Calendar Get All**

7. **Branch based on availability**
   - Node: **IF** named **Check Meeting Slot Availability**
   - Condition: check if the list of returned events is empty.
     - Configure the condition to correctly reflect the Google Calendar node’s output structure (commonly: check if number of items equals 0).
   - Connect: **Calendar Get All → IF**

8. **Unavailable branch: Email client**
   - Node: **Gmail** named **Send Notice for Unavailable Slot**
   - To: `{{ $('Code (Configure Meeting Information)').item.json.client_email }}`
   - Subject/body: inform requested slot unavailable (use formatted date/time + booking time).
   - Credentials: **Gmail OAuth2**
   - Connect: **IF false → Send Notice for Unavailable Slot**

9. **Available branch: Create Google Calendar event**
   - Node: **Google Calendar** named **Create Event Using Web Submission Form (Add booking to Google Calendar with details & reminders)**
   - Operation: **Create**
   - Calendar: same as above
   - Start: `{{ $('Code (Configure Meeting Information)').item.json.booking_start }}`
   - End: `{{ $('Code (Configure Meeting Information)').item.json.booking_end }}`
   - Summary: `{{ $('Add Event name, Duration, Password').item.json.event_title }}`
   - Description: include client name + message
   - Reminders: email 10 min, popup 5 min (disable default reminders)
   - Credentials: **Google Calendar OAuth2**
   - Connect: **IF true → Create Event**

10. **Create Zoom meeting**
   - Node: **Zoom** named **Create Zoom meeting with client info**
   - Topic: `Booking with {{ $('Code (Configure Meeting Information)').item.json.client_name }}`
   - Start time: booking_start
   - Time zone: Asia/Dhaka (or your choice)
   - Duration: set as an integer minutes (recommended: use the Set node’s `meeting_time`)
     - e.g. `{{ $('Add Event name, Duration, Password').item.json.meeting_time }}`
   - Password: use `meeting_password` or a Zoom-generated password
   - Credentials: **Zoom API** (OAuth or JWT depending on your n8n node support)
   - Connect: **Create Event → Create Zoom meeting**

11. **Send confirmation email to client**
   - Node: **Gmail** named **Send confirmation email to client**
   - To: client_email
   - Subject: include date/time
   - Body: include Zoom `{{$json.join_url}}` from the Zoom node output
   - Credentials: **Gmail OAuth2**
   - Connect: **Create Zoom meeting → Send confirmation email to client**

12. **(Optional) Notify Teams channel**
   - Node: **Microsoft Teams** named **Notify team via Microsoft Teams**
   - Resource: **channelMessage**
   - Select **Team ID** and **Channel ID**
   - Connect: **Create Zoom meeting → Notify team via Microsoft Teams**
   - Credentials: **Microsoft Teams OAuth2**

13. **Post-confirmation fan-out notifications**
   - Add and connect these nodes from **Send confirmation email to client** (parallel outputs):
   - **Microsoft Teams** named **Create chat message**
     - Resource: **chatMessage**
     - Set **chatId**
   - **Gmail** named **Send a Email**
     - To: your internal address
   - **Discord** named **Send a message**
     - sendTo: user
     - userId + bot credentials configured
   - **Rapiwa** named **Rapiwa ()WhatsApp Message**
     - number: your WhatsApp recipient
     - credentials: Rapiwa API token

14. **Credentials checklist**
   - Google Calendar OAuth2 (read + write calendar)
   - Gmail OAuth2 (send email)
   - Zoom API credential
   - Microsoft Teams OAuth2 (optional; requires choosing chat/channel IDs)
   - Discord Bot API (optional)
   - Rapiwa API (WhatsApp)

15. **Test with pinned data**
   - Use a sample payload with `date`, `time`, `name`, `email`, `message`.
   - Verify:
     - booking_start/end are correct timezone
     - Calendar availability check returns expected results
     - Event creation succeeds
     - Zoom returns `join_url`
     - Emails/notifications render correctly

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This intelligent workflow automates the meeting scheduling process from web form submissions…” + full credential/customization guidance | From Sticky Note1 |
| Google Calendar API Documentation | https://developers.google.com/calendar/api |
| Zoom API Documentation | https://marketplace.zoom.us/docs/api-reference/introduction |
| Gmail API Documentation | https://developers.google.com/gmail/api |
| Microsoft Teams API Documentation | https://docs.microsoft.com/en-us/microsoftteams/platform/overview |
| Discord API Documentation | https://discord.com/developers/docs/intro |
| Rapiwa API Documentation | https://rapiwa.com/docs |
| “# Web Form Meeting Booking & Calendar Check …” | From Sticky Note2 |
| “# Notifications & Meeting” | From Sticky Note4 |
| “notification after work is completed” | From Sticky Note3 |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.