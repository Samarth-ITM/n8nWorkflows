Send WhatsApp appointment reminders from Google Calendar with MoltFlow

https://n8nworkflows.xyz/workflows/send-whatsapp-appointment-reminders-from-google-calendar-with-moltflow-13482


# Send WhatsApp appointment reminders from Google Calendar with MoltFlow

## 1. Workflow Overview

**Purpose:** This workflow automatically sends **WhatsApp appointment reminders** for events in **Google Calendar** happening within the next **24 hours**, using the **MoltFlow (WaiFlow) API**.

**Target use cases:**
- Clinics, salons, coaches, service providers who schedule appointments in Google Calendar and want automated reminder messages.
- Teams that store client phone numbers inside event descriptions.

### 1.1 Scheduling / Trigger
Runs on a fixed cadence (every hour) to check for upcoming events.

### 1.2 Calendar Retrieval
Pulls events from a specific Google Calendar within a time window: **now → next 24 hours**.

### 1.3 Reminder Extraction & Message Formatting
Parses each event to extract a client phone number (and optionally name), creates a personalized message, and prepares payloads for WhatsApp sending.

### 1.4 Filtering & Sending via MoltFlow
Only proceeds for items that contain a valid `chat_id` and sends them through MoltFlow’s HTTP API.

### 1.5 Result Logging
Aggregates send results and outputs a summary object (count + per-item results).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduling / Trigger
**Overview:** Starts the workflow every hour to ensure reminders are continuously sent for near-future events.  
**Nodes involved:** `Every Hour`

#### Node: Every Hour
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) – entry point.
- **Configuration (interpreted):**
  - Runs at an interval: **every 1 hour**.
- **Inputs / outputs:**
  - **Output:** Triggers `Get Upcoming Events`.
- **Version notes:** Node typeVersion `1.2` (no special constraints).
- **Edge cases / failures:**
  - None typical beyond instance scheduling being disabled or workflow not active.

---

### Block 2.2 — Calendar Retrieval
**Overview:** Fetches all Google Calendar events starting now and ending within the next 24 hours, expanded to single instances.  
**Nodes involved:** `Get Upcoming Events`

#### Node: Get Upcoming Events
- **Type / role:** Google Calendar (`n8n-nodes-base.googleCalendar`) – reads events.
- **Operation:** `getAll` (fetch multiple events)
- **Configuration choices:**
  - **Calendar ID:** `primary` (selects the primary calendar of the connected Google account)
  - **Time window:**
    - `timeMin = {{$now.toISO()}}` (now)
    - `timeMax = {{$now.plus({ hours: 24 }).toISO()}}` (now + 24h)
  - **singleEvents: true**: expands recurring events into individual occurrences.
  - **returnAll: true**: returns all matching events (beware large volumes).
- **Credentials:**
  - Requires `googleCalendarOAuth2Api` (“Google Calendar”).
- **Inputs / outputs:**
  - **Input:** Trigger from `Every Hour`
  - **Output:** Array of event items to `Format Reminders`
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - OAuth issues (expired token, missing consent scopes).
  - Wrong calendarId (not accessible to the account).
  - Large event volume could impact performance (since `returnAll`).
  - Timezone nuances: `$now` is n8n server time; calendar events may be in different time zones.

---

### Block 2.3 — Reminder Extraction & Message Formatting
**Overview:** Converts calendar events into WhatsApp reminder payloads by parsing phone/name from the description and composing a reminder message.  
**Nodes involved:** `Format Reminders`

#### Node: Format Reminders
- **Type / role:** Code node (`n8n-nodes-base.code`) – transforms and filters events.
- **Mode:** `runOnceForAllItems` (processes all fetched events in one execution)
- **Key configuration choices (logic):**
  - Hard-coded constant: `SESSION_ID = 'YOUR_SESSION_ID'`  
    - Must be replaced with your MoltFlow session identifier.
  - Iterates through all events:
    - `summary`: defaults to `"Appointment"` if missing
    - `description`: defaults to `""`
    - `startTime`: uses `event.start.dateTime` or `event.start.date` (all-day events)
  - **Phone extraction (regex):**
    - First tries labeled formats like: `phone: ...`, `tel: ...`, `mobile: ...`, `whatsapp: ...`
    - Fallback: any number-like token `([+]?[0-9]{7,15})`
    - Normalizes to digits only: `replace(/[^0-9]/g, '')`
    - Rejects too-short numbers: `< 7 digits`
  - **Client name extraction:**
    - Looks for `name:` / `client:` / `patient:` line
    - Else uses first attendee displayName/email
    - Else `"there"`
  - **Time formatting:**
    - Attempts `toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: true })`
    - If parsing fails, keeps raw `startTime`
  - **Message template:**
    - `"Hi ${clientName}! Just a reminder about your appointment: ${summary} tomorrow at ${timeStr}. Reply YES to confirm or let us know if you need to reschedule."`
  - Output per reminder:
    - `session_id`
    - `chat_id = phone + '@c.us'` (WhatsApp chat identifier format used by this API)
    - `message`
    - also includes `client_name`, `event_name`, `event_time` for downstream logging.
  - If no reminders created:
    - Returns a single item: `{ status: 'no_reminders', message: 'No upcoming appointments with phone numbers found' }`
- **Inputs / outputs:**
  - **Input:** Events from `Get Upcoming Events`
  - **Output:** Reminder items to `Has Reminder?`
- **Version notes:** Code node typeVersion `2`.
- **Edge cases / failures:**
  - **“Tomorrow” wording is not time-window aware:** the workflow checks “next 24 hours” but always says “tomorrow”, which can be wrong (e.g., an appointment in 2 hours).
  - All-day events (`start.date`) produce a date without time; `toLocaleTimeString` may yield unexpected results or fallback to raw string.
  - Phone extraction may capture unintended numbers in description (IDs, ZIP codes).
  - Country code handling: stripping non-digits removes `+`; depending on MoltFlow expectations, you may need to preserve country code explicitly.
  - If `YOUR_SESSION_ID` is not replaced, messages will likely fail at the API call stage.

---

### Block 2.4 — Filtering & Sending via MoltFlow
**Overview:** Ensures only properly formatted reminder items are sent, then calls the MoltFlow API endpoint to deliver WhatsApp messages.  
**Nodes involved:** `Has Reminder?`, `Send WhatsApp Reminder`

#### Node: Has Reminder?
- **Type / role:** IF node (`n8n-nodes-base.if`) – filters items.
- **Condition (interpreted):**
  - Checks whether `{{$json.chat_id}}` **exists**.
- **Inputs / outputs:**
  - **Input:** Items from `Format Reminders`
  - **True output:** Goes to `Send WhatsApp Reminder`
  - **False output:** Not connected (items are dropped)
- **Version notes:** IF node typeVersion `2`.
- **Edge cases / failures:**
  - If the “no_reminders” fallback item is produced, it has no `chat_id`, so it will go to the false branch and effectively disappear (no logging node attached to false output).

#### Node: Send WhatsApp Reminder
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – sends message via MoltFlow.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Body:** JSON (provided as a string expression)
    - `{{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
  - **Authentication:** Generic credential type → **HTTP Header Auth**
    - Expected header: `X-API-Key: <your key>` (per sticky note)
- **Credentials:**
  - `httpHeaderAuth` credential named “MoltFlow API Key”
- **Inputs / outputs:**
  - **Input:** Filtered reminder items from `Has Reminder?` (true branch)
  - **Output:** API responses to `Log Results`
- **Version notes:** HTTP Request node typeVersion `4.2`.
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid.
  - 400 if `session_id` is invalid or message/chat_id format rejected.
  - Rate limits / throttling if many events.
  - Network timeouts, DNS issues.
  - Because body is passed via `JSON.stringify(...)`, ensure node is truly expecting JSON; misconfiguration can result in double-encoded JSON in some setups (here it’s set to `specifyBody: json` and `sendBody: true`, but the value is still a JSON string).

---

### Block 2.5 — Result Logging
**Overview:** Produces a final single summary item describing how many reminders were sent and to whom.  
**Nodes involved:** `Log Results`

#### Node: Log Results
- **Type / role:** Code node (`n8n-nodes-base.code`) – aggregates results.
- **Mode:** `runOnceForAllItems`
- **Logic (interpreted):**
  - Maps all incoming items (each should correspond to a send result) into:
    - `status: 'sent'`
    - `client`: `item.json.client_name` OR fallback to `$('Format Reminders').first().json.client_name`
    - `event`: `$('Format Reminders').first().json.event_name`
  - Returns a single output item:
    - `summary: 'Sent X appointment reminder(s)'`
    - `results: [...]`
- **Inputs / outputs:**
  - **Input:** Responses from `Send WhatsApp Reminder`
  - **Output:** Final workflow output (no further nodes)
- **Version notes:** Code node typeVersion `2`.
- **Edge cases / failures:**
  - `Send WhatsApp Reminder` responses likely do **not** include `client_name`; the fallback uses **only the first** formatted reminder’s values, which can make logs inaccurate when multiple reminders are sent.
  - If MoltFlow returns an error payload, this node still labels status as `'sent'` (no error inspection).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / canvas annotation |  |  | ## Calendar → WhatsApp Reminders; Automatically send WhatsApp reminders for upcoming appointments from your Google Calendar using [MoltFlow](https://molt.waiflow.app).; How it works:; 1. Runs every hour and checks your Google Calendar; 2. Finds appointments happening in the next 24 hours; 3. Extracts client phone numbers from event descriptions; 4. Sends a personalized WhatsApp reminder via MoltFlow |
| Sticky Note1 | Sticky Note | Setup instructions / canvas annotation |  |  | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Connect Google Calendar OAuth2 credential; 3. Set your calendar ID (default: primary); 4. Set `YOUR_SESSION_ID` in the Format Reminders node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; Format: Add client phone to calendar event description:; `Phone: 1234567890` |
| Every Hour | Schedule Trigger | Periodic execution (hourly) |  | Get Upcoming Events |  |
| Get Upcoming Events | Google Calendar | Fetch events within next 24 hours | Every Hour | Format Reminders |  |
| Format Reminders | Code | Parse events → build reminder payloads | Get Upcoming Events | Has Reminder? |  |
| Has Reminder? | IF | Filter items that have `chat_id` | Format Reminders | Send WhatsApp Reminder (true) |  |
| Send WhatsApp Reminder | HTTP Request | Send WhatsApp message via MoltFlow API | Has Reminder? (true) | Log Results |  |
| Log Results | Code | Aggregate send results to a summary | Send WhatsApp Reminder |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Send WhatsApp Appointment Reminders from Google Calendar with MoltFlow**
   - (Optional) Add tags: `whatsapp`, `google calendar`, `appointments`, `moltflow`, `reminders`

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Name: **Every Hour**
   - Configure interval:
     - Interval unit: **Hours**
     - Every: **1**

3. **Add node: Google Calendar**
   - Node type: **Google Calendar**
   - Name: **Get Upcoming Events**
   - Credentials:
     - Create/select **Google Calendar OAuth2** credential
     - Ensure the connected Google account has access to the target calendar
   - Operation: **Get All**
   - Calendar: choose **primary** (or replace with your calendar ID)
   - Options:
     - **Return All:** enabled
     - **Single Events:** enabled
     - **Time Min:** expression `{{$now.toISO()}}`
     - **Time Max:** expression `{{$now.plus({ hours: 24 }).toISO()}}`
   - Connect: `Every Hour` → `Get Upcoming Events`

4. **Add node: Code**
   - Node type: **Code**
   - Name: **Format Reminders**
   - Mode: **Run Once for All Items**
   - Paste logic equivalent to:
     - Define `SESSION_ID` and set it to your MoltFlow session (replace `'YOUR_SESSION_ID'`)
     - For each event: extract `summary`, `description`, `startTime`
     - Regex extract phone (prefer labeled `Phone:` formats)
     - Normalize phone to digits; require length ≥ 7
     - Extract client name from description or attendees
     - Compose message text
     - Output items with fields: `session_id`, `chat_id` (`<digits>@c.us`), `message`, plus metadata
     - If none found: output a single “no_reminders” item
   - Connect: `Get Upcoming Events` → `Format Reminders`

5. **Add node: IF**
   - Node type: **IF**
   - Name: **Has Reminder?**
   - Condition:
     - Type: String
     - Value 1: expression `{{$json.chat_id}}`
     - Operation: **exists**
   - Connect: `Format Reminders` → `Has Reminder?`

6. **Add node: HTTP Request**
   - Node type: **HTTP Request**
   - Name: **Send WhatsApp Reminder**
   - Method: **POST**
   - URL: `https://apiv2.waiflow.app/api/v2/messages/send`
   - Body content type: **JSON**
   - Body parameters (logical):
     - `session_id`: `{{$json.session_id}}`
     - `chat_id`: `{{$json.chat_id}}`
     - `message`: `{{$json.message}}`
   - Authentication:
     - Select: **Generic Credential Type**
     - Type: **HTTP Header Auth**
     - Create credential “MoltFlow API Key” with:
       - Header name: `X-API-Key`
       - Header value: your MoltFlow API key
   - Connect: `Has Reminder?` (true output) → `Send WhatsApp Reminder`

7. **Add node: Code (logging/summary)**
   - Node type: **Code**
   - Name: **Log Results**
   - Mode: **Run Once for All Items**
   - Implement aggregation:
     - Count items
     - Output `{ summary: 'Sent X appointment reminder(s)', results: [...] }`
   - Connect: `Send WhatsApp Reminder` → `Log Results`

8. **(Optional but recommended) Handle “no reminders” path**
   - Connect `Has Reminder?` (false output) to a logging node (e.g., Set/Code) so you still get a clean run output like “No upcoming appointments...”.

9. **Activate workflow**
   - Ensure credentials are valid.
   - Turn workflow **Active**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow service used to send WhatsApp messages | https://molt.waiflow.app |
| MoltFlow message send endpoint | `https://apiv2.waiflow.app/api/v2/messages/send` |
| Required API auth header | `X-API-Key: <your MoltFlow API key>` |
| Expected event description format for phone extraction | Example: `Phone: 1234567890` |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.