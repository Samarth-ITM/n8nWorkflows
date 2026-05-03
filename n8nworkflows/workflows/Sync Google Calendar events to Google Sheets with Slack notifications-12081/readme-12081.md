Sync Google Calendar events to Google Sheets with Slack notifications

https://n8nworkflows.xyz/workflows/sync-google-calendar-events-to-google-sheets-with-slack-notifications-12081


# Sync Google Calendar events to Google Sheets with Slack notifications

## 1. Workflow Overview

**Purpose:** Monitor a Google Calendar for newly created or updated events, normalize and enrich event data, log events and cancellations into Google Sheets, send Slack notifications, optionally email attendees a confirmation, and produce a daily 8 AM Slack + Sheets summary. It also includes centralized error alerting to Slack.

**Primary use cases:**
- Operational visibility: “What meetings were created/updated/canceled?”
- Simple audit trail in Google Sheets (“Events”, “Cancellations”, “Statistics” tabs)
- Team notifications in Slack (new event, cancellation, daily summary, errors)
- Automatic attendee confirmation email (Gmail)

### 1.1 Calendar Event Intake (Triggers)
Two polling triggers check the primary calendar every minute for **created** and **updated** events.

### 1.2 Event Normalization & Routing
A Code node extracts/derives standardized fields (date/time formatting, duration, platform detection, category tagging, cancellation flag). An IF node routes canceled vs active events.

### 1.3 Logging & Slack Alerts (Per Event)
Active events are appended to a Google Sheet and posted to Slack. Cancellations are logged to a dedicated sheet and posted to Slack.

### 1.4 Optional Confirmation Email (Per Event)
If an event has attendees, the workflow prepares a styled HTML email and sends it via Gmail to the attendee list.

### 1.5 Daily Summary (8 AM)
A schedule trigger fetches today’s events, calculates meeting statistics, posts a Slack daily summary, and logs statistics to Google Sheets.

### 1.6 Error Handling
An Error Trigger catches workflow failures, formats the error payload, and alerts Slack (including link back to the workflow run).

---

## 2. Block-by-Block Analysis

### Block 1 — Calendar Event Intake (Triggers)
**Overview:** Starts the workflow when Google Calendar detects a new event or an updated event, using minute-level polling.

**Nodes involved:**
- `New Event Trigger`
- `Updated Event Trigger`

#### Node: New Event Trigger
- **Type / role:** Google Calendar Trigger (`googleCalendarTrigger`) — polls for `eventCreated`.
- **Key config:**
  - Calendar: `primary`
  - Trigger: `eventCreated`
  - Polling: every minute
- **Outputs to:** `Process Event`
- **Failure/edge cases:**
  - Google auth/consent issues or revoked token
  - Polling delays and potential missed/duplicated triggers depending on Google API behavior and n8n polling window
  - Events created without certain fields (summary, attendees, location, etc.)

#### Node: Updated Event Trigger
- **Type / role:** Google Calendar Trigger — polls for `eventUpdated`.
- **Key config:**
  - Calendar: `primary`
  - Trigger: `eventUpdated`
  - Polling: every minute
- **Outputs to:** `Process Event`
- **Failure/edge cases:**
  - High update frequency can create multiple logs/notifications for the same event unless you add deduplication (not present here)
  - Same auth/polling concerns as above

**Sticky note (applies to this block):**
- “Step 1: Calendar Triggers … New event trigger / Updated event trigger / Monitors your calendar”

---

### Block 2 — Event Normalization & Routing
**Overview:** Converts raw Google Calendar event payload into a stable schema for Sheets/Slack/Email, and routes cancellations separately.

**Nodes involved:**
- `Process Event`
- `Is Canceled?`

#### Node: Process Event
- **Type / role:** Code node — data enrichment and normalization.
- **What it derives (output fields):**
  - IDs: `eventId`, `recurringEventId`
  - Basic: `title`, `description`, `category`
  - Date/time: `date`, `dayOfWeek`, `startTime`, `endTime`, `durationMinutes`, `durationText`, `allDay`, `isRecurring`
  - Location/platform: `platform`, `meetingLink`, `location`
  - Attendees: `organizer`, `creator`, `attendeeCount`, `attendees` (comma-separated emails), `accepted`, `declined`, `pending`
  - Status: `status` (human-readable), `cancelled` (boolean)
  - Metadata: `syncedAt` (ISO), `syncedAtFormatted` (localized)
- **Key logic notes:**
  - All-day detection: `isAllDay = !event.start?.dateTime`
  - Platform detection uses `hangoutLink`, then `conferenceData.entryPoints`, then checks `location` for Zoom/Teams/Meet patterns
  - Category tagging uses title keyword matching (meeting/call/sync/1:1/interview/lunch/demo/focus)
- **Outputs to:** `Is Canceled?`
- **Failure/edge cases:**
  - Locale/timezone: `toLocaleDateString` / `toLocaleTimeString` uses the n8n server’s locale/timezone unless otherwise configured; can differ from calendar timezone
  - All-day events: `endTime` often empty; duration defaults to “1 hour” unless start/end present and not all-day
  - `attendees` is converted to a comma-separated string; Gmail “Send To” may accept it, but formatting issues can occur if empty or contains invalid emails

#### Node: Is Canceled?
- **Type / role:** IF node — routes cancellations vs normal events.
- **Condition:** `{{ $json.cancelled }} == true`
- **True output →** `Log Cancellation`
- **False output →** `Log Event`
- **Failure/edge cases:**
  - If upstream event payload marks cancellation differently, `cancelled` might not match expectations (here it is derived strictly from `event.status === 'cancelled'`)

**Sticky note (applies to this block):**
- “Step 2: Event Processing … Extract event details / Check if cancelled / Route accordingly”

---

### Block 3 — Logging & Slack Alerts (Per Event / Cancellation)
**Overview:** Persists event records to Google Sheets and sends Slack notifications for visibility.

**Nodes involved:**
- `Log Event`
- `Slack - New Event`
- `Log Cancellation`
- `Slack - Cancellation`
- `Done - Cancellation`

#### Node: Log Event
- **Type / role:** Google Sheets — append event row to the “Events” tab.
- **Key config:**
  - Operation: Append
  - Document: `YOUR_DOCUMENT_ID` (must be replaced)
  - Sheet/tab name: `Events`
  - Column mapping (selected):
    - `ID` = `{{ $json.eventId }}`
    - `Day` = `{{ $json.dayOfWeek }}`
    - `Date` = `{{ $json.date }}`
    - `Title` = `{{ $json.title }}`
    - `Status` = `{{ $json.status }}`
    - `Category` = `{{ $json.category }}`
    - `Duration` = `{{ $json.durationText }}`
    - `Start Time` / `End Time`
    - `Platform`, `Meeting Link`
    - `Attendees` = `{{ $json.attendeeCount }}`
    - `Synced At` = `{{ $json.syncedAtFormatted }}`
- **Input:** `Is Canceled?` (false branch)
- **Output →** `Slack - New Event`
- **Failure/edge cases:**
  - Missing sheet/tab or mismatched headers causes append/mapping errors
  - Google Sheets API quotas / rate limits
  - Document ID not replaced

#### Node: Slack - New Event
- **Type / role:** Slack node — sends a message via configured Slack credentials/webhook.
- **Key config:**
  - Text: `:calendar: *New calendar event*`
  - `includeLinkToWorkflow: false`
  - WebhookId placeholder: `YOUR_WEBHOOK_ID`
- **Input:** `Log Event`
- **Output →** `Has Attendees?`
- **Failure/edge cases:**
  - Invalid webhook/Slack auth
  - Message is generic (no event details). If you expect details, you must expand the text with expressions.

#### Node: Log Cancellation
- **Type / role:** Google Sheets — append cancellation row to “Cancellations” tab.
- **Key config:**
  - Operation: Append
  - Document: `YOUR_DOCUMENT_ID`
  - Sheet/tab: `Cancellations`
  - Column mapping:
    - `ID`, `Title`, `Canceled At`, `Original Date`, `Original Time`, `Affected Attendees`
- **Input:** `Is Canceled?` (true branch)
- **Output →** `Slack - Cancellation`
- **Failure/edge cases:** same as `Log Event` plus ensuring “Cancellations” tab exists.

#### Node: Slack - Cancellation
- **Type / role:** Slack node — cancellation notification.
- **Key config:**
  - Text: `:x: *Event canceled*`
  - `includeLinkToWorkflow: false`
- **Input:** `Log Cancellation`
- **Output →** `Done - Cancellation`
- **Failure/edge cases:** Slack auth/webhook errors.

#### Node: Done - Cancellation
- **Type / role:** NoOp — explicit endpoint marker for the cancellation path.
- **Input:** `Slack - Cancellation`
- **Output:** none

**Sticky note (applies to this block):**
- “Step 3: Logging & Alerts … Log to Sheets / Slack notifications / Track changes”

---

### Block 4 — Optional Confirmation Email
**Overview:** After logging and Slack notification for a normal event, checks if the event has attendees; if yes, sends a styled HTML confirmation email to them via Gmail.

**Nodes involved:**
- `Has Attendees?`
- `Prepare Confirmation Email`
- `Send Email`
- `Done - Event`

#### Node: Has Attendees?
- **Type / role:** IF node — checks attendee count.
- **Condition:** number `{{ $('Process Event').item.json.attendeeCount }}` **larger than** (implicit) `0`
  - In n8n, “larger” without a second value typically implies comparing against a default; to be safe, set comparison value explicitly to `0`.
- **True output →** `Prepare Confirmation Email`
- **False output →** `Done - Event`
- **Failure/edge cases:**
  - Uses cross-node reference `$('Process Event')...` instead of `$json...`; works as long as node execution context is consistent, but can be brittle if multiple items are processed in parallel or if node name changes.
  - If attendeeCount is undefined, condition evaluation may fail or behave unexpectedly.

#### Node: Prepare Confirmation Email
- **Type / role:** Code node — builds HTML email content.
- **Key inputs:** `const data = $('Process Event').first().json;`
- **Output fields:**
  - `to`: `data.attendees` (comma-separated email list)
  - `subject`: `Event: ${data.title} - ${data.date} ${data.startTime}`
  - `htmlContent`: full HTML template
- **Notable configuration:**
  - Inline “CONFIG” with `companyName` and `primaryColor`
  - Includes “Join Meeting” button only if `data.meetingLink` exists
  - Includes description section only if present
- **Output →** `Send Email`
- **Failure/edge cases:**
  - `data.attendees` is a single string of emails; Gmail node generally accepts comma-separated recipients, but invalid formatting can cause send failures.
  - The HTML contains unsanitized `data.description` (from calendar). If description includes HTML, it will render; if you want plain text, you should escape it.

#### Node: Send Email
- **Type / role:** Gmail node — sends the email.
- **Key config:**
  - To: `{{ $json.to }}`
  - Subject: `{{ $json.subject }}`
  - Message: `{{ $json.htmlContent }}`
  - `appendAttribution: false`
- **Input:** `Prepare Confirmation Email`
- **Output →** `Done - Event`
- **Failure/edge cases:**
  - Gmail OAuth permission issues (send scope)
  - Recipient limits / anti-spam enforcement
  - Large HTML content or blocked content policies

#### Node: Done - Event
- **Type / role:** NoOp — explicit endpoint marker for the event path.
- **Inputs:** from `Send Email` (attendee path) and from `Has Attendees?` (no-attendee path)

**Sticky note (applies to this block):**
- “Step 4: Confirmation Email … Prepare email / Send to attendees / Confirmation flow”

---

### Block 5 — Daily Summary (8 AM)
**Overview:** Every day at 08:00, pulls today’s calendar events, computes meeting load metrics, posts a Slack summary, and logs stats to Google Sheets.

**Nodes involved:**
- `Daily Summary Trigger`
- `Get Today Events`
- `Calculate Statistics`
- `Slack - Daily Summary`
- `Log Statistics`
- `Done - Summary`

#### Node: Daily Summary Trigger
- **Type / role:** Schedule Trigger — cron-based execution.
- **Key config:** Cron expression `0 8 * * *` (8:00 every day)
- **Output →** `Get Today Events`
- **Failure/edge cases:**
  - Timezone is n8n instance timezone unless schedule configured otherwise; “8 AM” may not be your local time if server differs.

#### Node: Get Today Events
- **Type / role:** Google Calendar node — fetch today’s events.
- **Key config:**
  - Operation: `getAll`
  - Calendar: `primary`
  - timeMin: `{{ $now.startOf('day').toISO() }}`
  - timeMax: `{{ $now.endOf('day').toISO() }}`
- **Output →** `Calculate Statistics`
- **Failure/edge cases:**
  - `$now` uses n8n timezone; day boundaries may not match calendar locale/timezone.
  - Pagination / large calendars (getAll may fetch many events; API quota concerns)

#### Node: Calculate Statistics
- **Type / role:** Code node — aggregates the fetched events.
- **Key logic:**
  - Filters out canceled: `events.filter(e => e.json.status !== 'cancelled')`
  - Sums duration minutes for non–all-day events (intended), but the condition is:
    - `if (!event.start?.date) { totalMinutes += duration; }`
    - This is logically inverted: timed events usually have `start.dateTime`, all-day have `start.date`. The condition works to exclude all-day events, but it’s a bit indirect; ensure it behaves with your event shapes.
  - Counts virtual vs in-person based on `hangoutLink` or `conferenceData.entryPoints`
  - Builds `eventList` as bullet lines with time + summary
  - Busy % computed using 480 minutes (8-hour day)
- **Outputs:** a single JSON with `date`, `totalEvents`, `meetingTime`, `busyPercent`, `dayStatus`, `virtualMeetings`, `inPersonMeetings`, `eventList`, etc.
- **Output →** `Slack - Daily Summary`
- **Failure/edge cases:**
  - All-day events: duration computed from date-to-date may be large; they are excluded from totalMinutes by the condition, but still appear in `eventList`
  - Missing `end` times could produce `NaN` duration
  - Event time formatting depends on server locale/timezone

#### Node: Slack - Daily Summary
- **Type / role:** Slack message.
- **Key config:**
  - Text: `:chart_with_upwards_trend: *Daily Summary*` (generic)
  - `includeLinkToWorkflow: false`
- **Input:** `Calculate Statistics`
- **Output →** `Log Statistics`
- **Failure/edge cases:** Slack auth/webhook errors; message contains no stats unless expanded with expressions.

#### Node: Log Statistics
- **Type / role:** Google Sheets — append a row to “Statistics”.
- **Key config:**
  - Document: `YOUR_DOCUMENT_ID`
  - Sheet/tab: `Statistics`
  - Columns include Date, Busy %, Status, Minutes, Virtual, In-Person, Meeting Time, Total Events
- **Input:** `Slack - Daily Summary`
- **Output →** `Done - Summary`
- **Failure/edge cases:** sheet/tab must exist; quotas; documentId placeholder.

#### Node: Done - Summary
- **Type / role:** NoOp — explicit endpoint marker.

**Sticky note (applies to this block):**
- “Step 5: Daily Summary … 8 AM trigger / Calculate statistics / Slack report”

---

### Block 6 — Error Handling
**Overview:** Catches any workflow execution error and notifies Slack with formatted details.

**Nodes involved:**
- `Error Trigger`
- `Format Error`
- `Slack - Error Alert`

#### Node: Error Trigger
- **Type / role:** Error Trigger — runs when the workflow errors.
- **Output →** `Format Error`
- **Failure/edge cases:**
  - Only triggers on workflow errors; logical/data issues that do not throw may not alert.

#### Node: Format Error
- **Type / role:** Code node — standardizes error payload.
- **Input:** `$input.item.json` (n8n error object)
- **Output fields:** `errorTime`, `errorMessage`, `errorNode`, `workflow: 'Google Calendar Sync'`
- **Output →** `Slack - Error Alert`
- **Failure/edge cases:**
  - Error object shape can vary across n8n versions; some properties may be undefined.

#### Node: Slack - Error Alert
- **Type / role:** Slack message — error channel notification.
- **Key config:**
  - Text: `:rotating_light: *Workflow Error*`
  - `includeLinkToWorkflow: true` (adds run link in Slack message)
- **Failure/edge cases:** Slack auth/webhook errors.

**Sticky note (applies to this block):**
- “Step 6: Error Handling … Catch errors / Format details / Alert #errors”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Event Trigger | Google Calendar Trigger | Polls for newly created events | — | Process Event | Step 1: Calendar Triggers — New event trigger / Updated event trigger / Monitors your calendar |
| Updated Event Trigger | Google Calendar Trigger | Polls for updated events | — | Process Event | Step 1: Calendar Triggers — New event trigger / Updated event trigger / Monitors your calendar |
| Process Event | Code | Normalize/enrich event into stable schema | New Event Trigger; Updated Event Trigger | Is Canceled? | Step 2: Event Processing — Extract event details / Check if cancelled / Route accordingly |
| Is Canceled? | IF | Route canceled vs active | Process Event | Log Cancellation (true); Log Event (false) | Step 2: Event Processing — Extract event details / Check if cancelled / Route accordingly |
| Log Event | Google Sheets | Append event row to “Events” | Is Canceled? (false) | Slack - New Event | Step 3: Logging & Alerts — Log to Sheets / Slack notifications / Track changes |
| Slack - New Event | Slack | Post “new event” notification | Log Event | Has Attendees? | Step 3: Logging & Alerts — Log to Sheets / Slack notifications / Track changes |
| Has Attendees? | IF | Decide whether to email attendees | Slack - New Event | Prepare Confirmation Email (true); Done - Event (false) | Step 4: Confirmation Email — Prepare email / Send to attendees / Confirmation flow |
| Prepare Confirmation Email | Code | Build HTML email + subject/to | Has Attendees? (true) | Send Email | Step 4: Confirmation Email — Prepare email / Send to attendees / Confirmation flow |
| Send Email | Gmail | Send confirmation email | Prepare Confirmation Email | Done - Event | Step 4: Confirmation Email — Prepare email / Send to attendees / Confirmation flow |
| Done - Event | NoOp | End marker for event path | Send Email; Has Attendees? (false) | — | Step 4: Confirmation Email — Prepare email / Send to attendees / Confirmation flow |
| Log Cancellation | Google Sheets | Append cancellation row to “Cancellations” | Is Canceled? (true) | Slack - Cancellation | Step 3: Logging & Alerts — Log to Sheets / Slack notifications / Track changes |
| Slack - Cancellation | Slack | Post cancellation notification | Log Cancellation | Done - Cancellation | Step 3: Logging & Alerts — Log to Sheets / Slack notifications / Track changes |
| Done - Cancellation | NoOp | End marker for cancellation path | Slack - Cancellation | — | Step 3: Logging & Alerts — Log to Sheets / Slack notifications / Track changes |
| Daily Summary Trigger | Schedule Trigger | Runs daily at 08:00 | — | Get Today Events | Step 5: Daily Summary — 8 AM trigger / Calculate statistics / Slack report |
| Get Today Events | Google Calendar | Fetch all events for today | Daily Summary Trigger | Calculate Statistics | Step 5: Daily Summary — 8 AM trigger / Calculate statistics / Slack report |
| Calculate Statistics | Code | Compute totals, meeting time, busy %, list | Get Today Events | Slack - Daily Summary | Step 5: Daily Summary — 8 AM trigger / Calculate statistics / Slack report |
| Slack - Daily Summary | Slack | Post daily summary header | Calculate Statistics | Log Statistics | Step 5: Daily Summary — 8 AM trigger / Calculate statistics / Slack report |
| Log Statistics | Google Sheets | Append daily stats to “Statistics” | Slack - Daily Summary | Done - Summary | Step 5: Daily Summary — 8 AM trigger / Calculate statistics / Slack report |
| Done - Summary | NoOp | End marker for summary path | Log Statistics | — | Step 5: Daily Summary — 8 AM trigger / Calculate statistics / Slack report |
| Error Trigger | Error Trigger | Entry point on workflow error | — | Format Error | Step 6: Error Handling — Catch errors / Format details / Alert #errors |
| Format Error | Code | Format error details to a clean schema | Error Trigger | Slack - Error Alert | Step 6: Error Handling — Catch errors / Format details / Alert #errors |
| Slack - Error Alert | Slack | Notify Slack with error + workflow link | Format Error | — | Step 6: Error Handling — Catch errors / Format details / Alert #errors |
| Overview | Sticky Note | Documentation/comment | — | — | ## Sync Google Calendar events to Sheets with Slack notifications … Setup steps … |
| Step 1 Calendar Triggers | Sticky Note | Documentation/comment | — | — | ## Step 1: Calendar Triggers … |
| Step 2 Event Processing | Sticky Note | Documentation/comment | — | — | ## Step 2: Event Processing … |
| Step 3 Logging | Sticky Note | Documentation/comment | — | — | ## Step 3: Logging & Alerts … |
| Step 4 Email | Sticky Note | Documentation/comment | — | — | ## Step 4: Confirmation Email … |
| Step 5 Daily Summary | Sticky Note | Documentation/comment | — | — | ## Step 5: Daily Summary … |
| Step 6 Error Handling | Sticky Note | Documentation/comment | — | — | ## Step 6: Error Handling … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Sync Google Calendar events to Google Sheets with Slack notifications*.

2. **Add credentials (required):**
   - **Google Calendar OAuth2** credential (scopes to read calendar events; triggers require access).
   - **Google Sheets OAuth2** credential (edit access to append rows).
   - **Slack**: either Slack API credential or Incoming Webhook-based Slack node configuration (match your node type usage). Replace placeholders like `YOUR_WEBHOOK_ID`.
   - **Gmail OAuth2** credential with send permissions.

3. **Create Google Sheets document** with three tabs (exact names matter):
   - `Events`
   - `Cancellations`
   - `Statistics`
   Ensure column headers match the fields you map (e.g., “ID”, “Day”, “Date”, “Title”, etc.). Copy the **Document ID**.

### Event path (new/updated)
4. **Add node:** *Google Calendar Trigger* → name it `New Event Trigger`
   - Calendar: `primary`
   - Trigger on: `eventCreated`
   - Polling: every minute

5. **Add node:** *Google Calendar Trigger* → name it `Updated Event Trigger`
   - Calendar: `primary`
   - Trigger on: `eventUpdated`
   - Polling: every minute

6. **Add node:** *Code* → name it `Process Event`
   - Paste the workflow’s JS logic that:
     - Extracts core fields
     - Computes date/time display fields, duration, category, platform/link, attendee stats
     - Sets `cancelled` boolean and `statusText`
   - Connect:
     - `New Event Trigger` → `Process Event`
     - `Updated Event Trigger` → `Process Event`

7. **Add node:** *IF* → name it `Is Canceled?`
   - Boolean condition: `{{ $json.cancelled }}` equals `true`
   - Connect `Process Event` → `Is Canceled?`

#### Active event branch
8. **Add node:** *Google Sheets* → name it `Log Event`
   - Operation: Append
   - Document ID: your Sheet document ID
   - Sheet name: `Events`
   - Map columns using expressions from `Process Event` output (ID, Day, Date, Title, etc.)
   - Connect `Is Canceled?` (false) → `Log Event`

9. **Add node:** *Slack* → name it `Slack - New Event`
   - Message text: `:calendar: *New calendar event*`
   - Disable “include link to workflow” (optional)
   - Connect `Log Event` → `Slack - New Event`

10. **Add node:** *IF* → name it `Has Attendees?`
   - Condition: attendeeCount > 0  
     Recommended robust setup:
     - Number condition: `{{ $json.attendeeCount }}` **larger than** `0`
     - (Instead of referencing `$('Process Event')...`.)
   - Connect `Slack - New Event` → `Has Attendees?`

11. **Add node:** *Code* → name it `Prepare Confirmation Email`
   - Build HTML email template, and output:
     - `to` (comma-separated)
     - `subject`
     - `htmlContent`
   - Connect `Has Attendees?` (true) → `Prepare Confirmation Email`

12. **Add node:** *Gmail* → name it `Send Email`
   - To: `{{ $json.to }}`
   - Subject: `{{ $json.subject }}`
   - Message: `{{ $json.htmlContent }}`
   - Append attribution: false
   - Connect `Prepare Confirmation Email` → `Send Email`

13. **Add node:** *NoOp* → name it `Done - Event`
   - Connect:
     - `Send Email` → `Done - Event`
     - `Has Attendees?` (false) → `Done - Event`

#### Cancellation branch
14. **Add node:** *Google Sheets* → name it `Log Cancellation`
   - Operation: Append
   - Document ID: your Sheet document ID
   - Sheet name: `Cancellations`
   - Map cancellation columns (ID, Title, Canceled At, etc.)
   - Connect `Is Canceled?` (true) → `Log Cancellation`

15. **Add node:** *Slack* → name it `Slack - Cancellation`
   - Text: `:x: *Event canceled*`
   - Connect `Log Cancellation` → `Slack - Cancellation`

16. **Add node:** *NoOp* → name it `Done - Cancellation`
   - Connect `Slack - Cancellation` → `Done - Cancellation`

### Daily summary path
17. **Add node:** *Schedule Trigger* → name it `Daily Summary Trigger`
   - Cron: `0 8 * * *` (confirm timezone)

18. **Add node:** *Google Calendar* → name it `Get Today Events`
   - Operation: Get All
   - Calendar: `primary`
   - timeMin: `{{ $now.startOf('day').toISO() }}`
   - timeMax: `{{ $now.endOf('day').toISO() }}`
   - Connect `Daily Summary Trigger` → `Get Today Events`

19. **Add node:** *Code* → name it `Calculate Statistics`
   - Aggregate events, compute:
     - `totalEvents`, `totalMinutes`, `meetingTime`, `busyPercent`, `dayStatus`, `virtualMeetings`, `inPersonMeetings`, `eventList`
   - Connect `Get Today Events` → `Calculate Statistics`

20. **Add node:** *Slack* → name it `Slack - Daily Summary`
   - Text: `:chart_with_upwards_trend: *Daily Summary*`
   - Connect `Calculate Statistics` → `Slack - Daily Summary`

21. **Add node:** *Google Sheets* → name it `Log Statistics`
   - Operation: Append
   - Document ID: your Sheet document ID
   - Sheet name: `Statistics`
   - Map Date, Busy %, Status, Minutes, Virtual, In-Person, Meeting Time, Total Events
   - Connect `Slack - Daily Summary` → `Log Statistics`

22. **Add node:** *NoOp* → name it `Done - Summary`
   - Connect `Log Statistics` → `Done - Summary`

### Error handling path
23. **Add node:** *Error Trigger* → name it `Error Trigger`

24. **Add node:** *Code* → name it `Format Error`
   - Output fields: time, message, node name, workflow label
   - Connect `Error Trigger` → `Format Error`

25. **Add node:** *Slack* → name it `Slack - Error Alert`
   - Text: `:rotating_light: *Workflow Error*`
   - Enable “include link to workflow”
   - Connect `Format Error` → `Slack - Error Alert`

26. **Replace placeholders** everywhere:
   - `YOUR_DOCUMENT_ID` in all Sheets nodes
   - `YOUR_WEBHOOK_ID` / Slack credential config
   - Ensure Google credentials are selected in Calendar/Sheets/Gmail nodes

27. **Test sequence:**
   - Create a timed event with attendees + Meet/Zoom link → verify Events row, Slack new event, email send
   - Cancel an event → verify Cancellations row, Slack cancellation
   - Run daily summary manually → verify Slack message + Statistics row
   - Force an error (e.g., wrong sheet name) → verify Slack error alert

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automatically track all calendar events, log them to Google Sheets, and send Slack notifications… Daily 8 AM summary with stats.” | Sticky note: “Overview” |
| Setup steps: connect Google Calendar/Sheets/Slack/Gmail; create “Events” tab; replace `YOUR_DOCUMENT_ID`; configure Slack channels `#calendar` and `#errors`; test by creating a calendar event | Sticky note: “Overview” |
| Disclaimer (FR): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Provided by user (applies globally) |