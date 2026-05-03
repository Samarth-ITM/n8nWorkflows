Google Calendar automated notifications with email, SMS & analytics dashboard

https://n8nworkflows.xyz/workflows/google-calendar-automated-notifications-with-email--sms---analytics-dashboard-12083


# Google Calendar automated notifications with email, SMS & analytics dashboard

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Google Calendar automated notifications with email, SMS & analytics dashboard

**Purpose:**  
This workflow sends automated daily and weekly Google Calendar digests via **email** and **SMS**, plus near-real-time **15-minute reminder SMS** before upcoming events. Every run is **logged to Google Sheets**, and a “Daily Stats” sheet is **updated/aggregated** for analytics (intended to power a reporting webpage via Apps Script + a web frontend).

**Primary use cases**
- Individuals/teams who want a daily schedule delivered by email and SMS.
- Operations teams needing “event starting soon” reminders.
- Lightweight analytics: how many runs occurred, events processed, and run types per day.

### 1.1 Daily Email Summary (6 AM)
- Trigger at 06:00 daily
- Fetch today’s calendar events
- Build a branded HTML email digest
- Send email (SendGrid used; Mailchimp node exists but not connected)
- Log execution to Google Sheets + update daily stats

### 1.2 Daily SMS Summary (7 AM)
- Trigger at 07:00 daily
- Fetch today’s events
- Format a concise SMS agenda
- Send via Twilio
- Log execution to Google Sheets + update daily stats

### 1.3 15-minute Reminder SMS (every 5 minutes)
- Trigger every 5 minutes
- Fetch events starting within the next 20 minutes
- Format reminder SMS containing events and short descriptions
- Send via Twilio
- Log execution to Google Sheets + update daily stats

### 1.4 Weekly Preview SMS (Sunday 6 PM)
- Trigger Sunday at 18:00
- Fetch events for next 7 days (tomorrow through 7 days out)
- Group and format by day
- Send via Twilio
- Log execution to Google Sheets + update daily stats

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Email and SMS Summary
**Overview:** Runs two independent daily digests: an HTML email at 6 AM and an SMS agenda at 7 AM, both sourced from today’s Google Calendar events and logged into Google Sheets.

**Nodes involved:**
- 6 AM Daily Trigger
- Get Today’s Events
- Format HTML Email
- Send an email (SendGrid)
- Append to Execution Log5
- Update or append daily stats
- 7 AM Daily Trigger
- Get Today’s Events1
- Format SMS Message
- Twilio SMS
- Append to Execution Log7
- Update or append daily stats1
- Send Email - Mailchimp Transactional (Mandrill) *(present but not connected)*

#### 2.1 Node: 6 AM Daily Trigger
- **Type / role:** Schedule Trigger; entry point for daily email.
- **Config:** Cron `0 6 * * *` (06:00 every day).
- **Outputs:** To **Get Today’s Events**.
- **Version notes:** scheduleTrigger `typeVersion 1.2`.
- **Failure modes:** Timezone confusion (n8n instance timezone governs cron interpretation).

#### 2.2 Node: Get Today’s Events
- **Type / role:** Google Calendar “Get All” events for today.
- **Config choices:**
  - Operation: `getAll`, `returnAll: true`
  - `timeMin = {{$now.startOf('day').toISO()}}`
  - `timeMax = {{$now.endOf('day').toISO()}}`
  - `singleEvents: true` (expands recurring events into instances)
  - `orderBy: startTime`
  - Calendar ID is set to `Template_sheet_id` (placeholder).
- **Credentials:** Google Calendar OAuth2.
- **Input/Output:** From trigger → outputs list of event items to **Format HTML Email**.
- **Edge cases / failures:**
  - OAuth token expiration / missing scopes
  - Calendar ID invalid
  - Events with missing `summary`, `start`, or `end` (rare but possible)
  - Timezone differences if calendar events stored with different TZ; ISO comparisons may behave unexpectedly.

#### 2.3 Node: Format HTML Email
- **Type / role:** Code node (JS) to convert events into a styled HTML digest and metadata.
- **Key logic:**
  - `const events = $input.all();`
  - If no events: returns `emailBody` with “No Events Today”, `eventCount: 0`, `hasEvents: false`.
  - Otherwise builds HTML:
    - Uses `event.json.start.dateTime || event.json.start.date` to handle timed vs all-day events.
    - Creates `subject` like: `Your Schedule for <date> - N events`
    - Includes optional `location` and `description` (newlines converted to `<br>`).
- **Outputs:** Single item with `emailBody`, `eventCount`, `hasEvents`, `subject` → **Send an email (SendGrid)**.
- **Failure modes:**
  - If an event lacks `start` or `start.date/dateTime`, `new Date(undefined)` becomes invalid.
  - Descriptions with HTML could inject markup into the email (not sanitized).
- **Version notes:** code node `typeVersion 2`.

#### 2.4 Node: Send an email
- **Type / role:** SendGrid node; sends the daily HTML email.
- **Config:**
  - Subject: `={{ $json.subject }}`
  - `contentType: text/html`
  - Body: `={{ $json.emailBody }}`
  - To/From fields are placeholders (`Recipient_email`, `Your_email`, etc.)
- **Outputs:** To **Append to Execution Log5**.
- **Failure modes:**
  - SendGrid auth/key invalid
  - Sender domain not verified (SendGrid may reject)
  - Rate limits
  - Large HTML content (rare, but could exceed provider limits with many events)

#### 2.5 Node: Append to Execution Log5
- **Type / role:** Google Sheets append row into “Execution Log”.
- **Config:**
  - Spreadsheet: **Calendar Analytics** (document ID `1KN2xX...`)
  - Sheet: **Execution Log** (gid `2068938043`)
  - Header row: 3
  - Writes columns including:
    - `Workflow Name = 📧 Daily Email Summary`
    - `Events Processed = {{ $('Format HTML Email').item.json.eventCount }}`
    - `Execution ID = {{ $json.messageId || 'N\A'}}` (expects `messageId` from SendGrid output)
    - Start/End Time fields are filled with `new Date().toISOString()` and date-only variant
    - Status hard-coded `success`
- **Connections:** From **Send an email** → to **Update or append daily stats**.
- **Edge cases / failures:**
  - OAuth issues or missing write permission
  - Header row mismatch (if sheet layout differs)
  - `messageId` field name may differ from SendGrid output; could log `N\A` unexpectedly
  - Uses `matchingColumns: ["executionId"]` but the schema shows “Execution ID”. If mismatch exists in the underlying sheet, append still works, but “matching” is irrelevant for append.
- **Version notes:** googleSheets node `typeVersion 4.4`.

#### 2.6 Node: Update or append daily stats
- **Type / role:** Google Sheets “appendOrUpdate” into “Daily Stats”.
- **Config:**
  - Sheet: **Daily Stats** (gid `433157270`)
  - Header row: 3; first data row: 4
  - Matching column: `Date`
  - Sets `Date = {{ $json["End Time"] }}` (coming from the previous appended log row payload)
  - Other fields are Google Sheets formulas (COUNTIFS/SUMIFS) referencing the Execution Log sheet.
- **Connections:** From **Append to Execution Log5**.
- **Edge cases / failures:**
  - If the “End Time” value is not in the same format as “Date” column expects, matching may fail, causing duplicate rows.
  - Locale differences in formula separators (`,` vs `;`) depending on spreadsheet locale.
- **Version notes:** googleSheets node `typeVersion 4.7`.

#### 2.7 Node: 7 AM Daily Trigger
- **Type / role:** Schedule Trigger; entry point for daily SMS digest.
- **Config:** Cron `0 7 * * *`.
- **Output:** To **Get Today’s Events1**.

#### 2.8 Node: Get Today’s Events1
- **Type / role:** Same as “Get Today’s Events”, duplicated for the 7 AM path.
- **Config:** Identical time window expressions and calendar selection.
- **Output:** To **Format SMS Message**.
- **Edge cases:** Same as the email variant.

#### 2.9 Node: Format SMS Message
- **Type / role:** Code node to format a concise daily SMS agenda.
- **Key logic:**
  - If no events: `smsBody: '📅 No events scheduled for today...'`, `hasEvents: false`
  - Otherwise loops events and includes:
    - time (or “All Day”)
    - summary
    - optional location
  - Appends total event count.
- **Output:** To **Twilio SMS**.
- **Failure modes:**
  - SMS length: long days can exceed 1600+ char (carrier/Twilio segmentation applies); message may be split or rejected depending on settings.

#### 2.10 Node: Twilio SMS
- **Type / role:** Twilio; sends the daily SMS summary.
- **Config:**
  - To: `Recipient_number` (placeholder)
  - From: `Your_Twillo_number` (placeholder)
  - Message: `={{ $('Format SMS Message').item.json.smsBody }}`
- **Output:** To **Append to Execution Log7**.
- **Failure modes:** Twilio auth errors, invalid/from-number mismatch, unverified destination in trial accounts, regional restrictions.

#### 2.11 Node: Append to Execution Log7
- **Type / role:** Google Sheets append into Execution Log for daily SMS run.
- **Config differences:**
  - `Workflow Name = 📱 Daily SMS Summary`
  - `Events Processed = {{ $('Format SMS Message').item.json.eventCount }}`
  - Execution ID uses `{{ $json.messageId || 'N\A'}}` (Twilio typically returns `sid`, not `messageId`, so this may log `N\A` unless Twilio output includes messageId).
- **Output:** To **Update or append daily stats1**.

#### 2.12 Node: Update or append daily stats1
- **Type / role:** Same as “Update or append daily stats”, duplicated for this path.
- **Config:** Same formulas/matching.

#### 2.13 Node: Send Email - Mailchimp Transactional
- **Type / role:** Mailchimp node configured to “send” (intended Mandrill transactional).
- **Notes:** “For Mailchimp Transactional (Mandrill), you may need to use HTTP Request node instead”.
- **Connections:** **None** (not used in current workflow).
- **Failure modes:** Mailchimp Transactional isn’t always supported directly; may require Mandrill API endpoints via HTTP Request.

---

### Block 2 — 15-minute SMS Reminder before Events
**Overview:** Polls the calendar every 5 minutes to detect events starting soon (next 20 minutes) and sends a reminder SMS containing the upcoming events, then logs and aggregates stats.

**Nodes involved:**
- Check Every 5 Minutes1
- Get Events in Next 20 Min1
- Format Reminder SMS1
- Twilio SMS1
- Append to Execution Log8
- Update or append daily stats2

#### 2.14 Node: Check Every 5 Minutes1
- **Type / role:** Schedule Trigger; frequent polling entry point.
- **Config:** Interval “minutes” with no number shown; in n8n this commonly defaults to **every 1 minute** unless specified. The node name/notes indicate intended **every 5 minutes**.
- **Note:** “Checks every 5 minutes to catch events that need 15-min reminders”.
- **Output:** To **Get Events in Next 20 Min1**.
- **Edge case:** If interval misconfigured to 1 minute, you may send duplicate reminders and hit API/SMS rate limits.

#### 2.15 Node: Get Events in Next 20 Min1
- **Type / role:** Google Calendar getAll for a rolling window.
- **Config:**
  - `timeMin = {{$now.toISO()}}`
  - `timeMax = {{$now.plus({ minutes: 20 }).toISO()}}`
  - `singleEvents: true`, `orderBy: startTime`
- **Output:** To **Format Reminder SMS1**.
- **Edge cases:**
  - Overlapping polling windows will repeatedly include the same events; no deduplication exists in workflow.
  - All-day events can appear depending on how Google returns them relative to “now”; could generate unwanted reminders.

#### 2.16 Node: Format Reminder SMS1
- **Type / role:** Code node that formats reminder SMS and also outputs structured event details.
- **Key logic:**
  - If no events: outputs `eventCount: 0` and `smsBody: 'No events found for today.'`
  - Else builds:
    - `smsBody` listing time, title, location, short description (first 60 chars)
    - `events`: array of objects with id/title/start/end/location/status/isAllDay/htmlLink
    - `date` as `YYYY-MM-DD`
- **Output:** To **Twilio SMS1**.
- **Failure modes:**
  - The “No events found for today.” body is misleading for a rolling window (it’s “next 20 minutes”, not “today”).
  - Long SMS risk (multiple events + descriptions).

#### 2.17 Node: Twilio SMS1
- **Type / role:** Twilio; sends reminders.
- **Config:** Message from `={{ $('Format Reminder SMS1').item.json.smsBody }}`.
- **Output:** To **Append to Execution Log8**.

#### 2.18 Node: Append to Execution Log8
- **Type / role:** Google Sheets append for reminder runs.
- **Config:**
  - `Workflow Name = ⏰ 15-Minute Reminders`
  - `Events Processed = {{ $('Format Reminder SMS1').item.json.eventCount }}`
  - Execution ID uses `{{ $json.messageId || 'N\A' }}` (likely mismatch with Twilio `sid`).
- **Output:** To **Update or append daily stats2**.

#### 2.19 Node: Update or append daily stats2
- **Type / role:** Same “Daily Stats” updater, duplicated for this path.

---

### Block 3 — Next 7-day Weekly Sunday SMS Review
**Overview:** Every Sunday evening, gathers events for the upcoming week, groups them by day, sends a preview SMS, then logs and updates stats.

**Nodes involved:**
- Sunday 6 PM Trigger1
- Get Next 7 Days1
- Format Weekly Summary1
- Twilio SMS2
- Append to Execution Log9
- Update or append daily stats3

#### 2.20 Node: Sunday 6 PM Trigger1
- **Type / role:** Schedule Trigger; weekly entry point.
- **Config:** Cron `0 18 * * 0` (Sunday 18:00).
- **Output:** To **Get Next 7 Days1**.
- **Failure mode:** Cron day-of-week interpretation depends on n8n’s cron implementation (0 usually Sunday, but confirm in your environment).

#### 2.21 Node: Get Next 7 Days1
- **Type / role:** Google Calendar getAll for next week window.
- **Config:**
  - `timeMin = {{$now.plus({ days: 1 }).startOf('day').toISO()}}` (tomorrow 00:00)
  - `timeMax = {{$now.plus({ days: 8 }).startOf('day').toISO()}}` (8 days from now at 00:00; effectively 7 full days)
  - `singleEvents: true`, `orderBy: startTime`
- **Output:** To **Format Weekly Summary1**.

#### 2.22 Node: Format Weekly Summary1
- **Type / role:** Code node; groups events by day and formats a weekly preview SMS.
- **Key logic:**
  - If no events: outputs “No events scheduled for the upcoming week…”
  - Groups by `weekday, month short, day numeric`
  - Each event line: `• time - summary @ location`
  - Outputs `eventCount` and `dayCount`
- **Output:** To **Twilio SMS2**.
- **Edge cases:** SMS length can exceed limits for busy calendars.

#### 2.23 Node: Twilio SMS2
- **Type / role:** Twilio; sends weekly preview.
- **Config:** `from` is `+1234567890` (placeholder; inconsistent with other Twilio nodes).
- **Output:** To **Append to Execution Log9**.

#### 2.24 Node: Append to Execution Log9
- **Type / role:** Google Sheets append for weekly preview run.
- **Config:**
  - `Workflow Name = 📅 Weekly Preview`
  - `Events Processed = {{ $('Format Weekly Summary1').item.json.eventCount }}`
- **Output:** To **Update or append daily stats3**.

#### 2.25 Node: Update or append daily stats3
- **Type / role:** Same “Daily Stats” updater, duplicated for this path.

---

### Block 4 — Documentation / Canvas Notes
**Overview:** Sticky notes describe the intended structure, setup steps, and external resources (Google Sheet template, Apps Script, reporting webpage).

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

#### 2.26 Node: Sticky Note (## 1. Daily Email and SMS Summary)
- **Type / role:** Sticky note labeling block 1.
- **Content:** `## 1.  Daily Email and SMS Summary`
- **Applies to:** Daily email + daily SMS path nodes (section header on canvas).

#### 2.27 Node: Sticky Note1 (15-minute reminders)
- **Content:** `## 2. 15-minute SMS Reminder before Events`
- **Applies to:** Reminder block nodes.

#### 2.28 Node: Sticky Note2 (weekly preview)
- **Content:** `## 3. Next 7-day Weekly Sunday SMS Review`
- **Applies to:** Weekly preview block nodes.

#### 2.29 Node: Sticky Note3 (How it works / setup / links)
- **Content highlights:**
  - Explains node roles: Schedule Trigger, Google Calendar, Code, Mailchimp/SendGrid, Twilio, Google Sheets.
  - Setup steps referencing Google Sheet template + Apps Script + webpage.
  - Links:
    - Twilio console: https://console.twilio.com/
    - Google Cloud: https://console.cloud.google.com/
    - Get 3-sheet G/Sheet template: https://docs.google.com/spreadsheets/d/1S91eS0LV51DZ_9KkA20tfns27umyOU5V/edit?usp=drive_link&ouid=113513617635127854147&rtpof=true&sd=true
    - Get App Script Code: https://drive.google.com/file/d/1iamkTByQ00aNVsjkr7Nfz1mtn9_b03g9/view?usp=drive_link
    - Get the Report WebPage site: https://drive.google.com/file/d/1R0-v8nMEOjYDkaodhsFxo-eTXIByE6fP/view?usp=sharing

#### 2.30 Node: Sticky Note4 (Mailchimp note)
- **Content:** `## Can work with MailChimp by replacing with SendGrid - Standard plan or higher`
- **Applies to:** Email sending area (SendGrid/Mailchimp swap guidance).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 6 AM Daily Trigger | scheduleTrigger | Daily email entry point at 06:00 | — | Get Today's Events | ## 1.  Daily Email and SMS Summary |
| Get Today's Events | googleCalendar | Fetch today’s events for email digest | 6 AM Daily Trigger | Format HTML Email | ## 1.  Daily Email and SMS Summary |
| Format HTML Email | code | Build HTML email + subject + counts | Get Today's Events | Send an email | ## 1.  Daily Email and SMS Summary |
| Send an email | sendGrid | Send HTML email digest | Format HTML Email | Append to Execution Log5 | ## 1.  Daily Email and SMS Summary |
| Send Email - Mailchimp Transactional | mailchimp | Alternative email sender (unused) | — | — | ## Can work with MailChimp by replacing with SendGrid - Standard plan or higher |
| Append to Execution Log5 | googleSheets | Append email run result to Execution Log sheet | Send an email | Update or append daily stats | ## 1.  Daily Email and SMS Summary |
| Update or append daily stats | googleSheets | Aggregate/update Daily Stats based on Execution Log | Append to Execution Log5 | — | ## 1.  Daily Email and SMS Summary |
| 7 AM Daily Trigger | scheduleTrigger | Daily SMS entry point at 07:00 | — | Get Today's Events1 | ## 1.  Daily Email and SMS Summary |
| Get Today's Events1 | googleCalendar | Fetch today’s events for SMS digest | 7 AM Daily Trigger | Format SMS Message | ## 1.  Daily Email and SMS Summary |
| Format SMS Message | code | Build daily SMS agenda text | Get Today's Events1 | Twilio SMS | ## 1.  Daily Email and SMS Summary |
| Twilio SMS | twilio | Send daily SMS agenda | Format SMS Message | Append to Execution Log7 | ## 1.  Daily Email and SMS Summary |
| Append to Execution Log7 | googleSheets | Append SMS run result to Execution Log sheet | Twilio SMS | Update or append daily stats1 | ## 1.  Daily Email and SMS Summary |
| Update or append daily stats1 | googleSheets | Aggregate/update Daily Stats (dup node) | Append to Execution Log7 | — | ## 1.  Daily Email and SMS Summary |
| Check Every 5 Minutes1 | scheduleTrigger | Frequent polling for upcoming events | — | Get Events in Next 20 Min1 | ## 2. 15-minute SMS Reminder before Events |
| Get Events in Next 20 Min1 | googleCalendar | Fetch events starting within next 20 minutes | Check Every 5 Minutes1 | Format Reminder SMS1 | ## 2. 15-minute SMS Reminder before Events |
| Format Reminder SMS1 | code | Build reminder SMS + structured event array | Get Events in Next 20 Min1 | Twilio SMS1 | ## 2. 15-minute SMS Reminder before Events |
| Twilio SMS1 | twilio | Send 15-min reminder SMS | Format Reminder SMS1 | Append to Execution Log8 | ## 2. 15-minute SMS Reminder before Events |
| Append to Execution Log8 | googleSheets | Append reminder run result to Execution Log sheet | Twilio SMS1 | Update or append daily stats2 | ## 2. 15-minute SMS Reminder before Events |
| Update or append daily stats2 | googleSheets | Aggregate/update Daily Stats (dup node) | Append to Execution Log8 | — | ## 2. 15-minute SMS Reminder before Events |
| Sunday 6 PM Trigger1 | scheduleTrigger | Weekly preview entry point | — | Get Next 7 Days1 | ## 3. Next 7-day Weekly Sunday SMS Review |
| Get Next 7 Days1 | googleCalendar | Fetch upcoming week events | Sunday 6 PM Trigger1 | Format Weekly Summary1 | ## 3. Next 7-day Weekly Sunday SMS Review |
| Format Weekly Summary1 | code | Group events by day and format weekly SMS | Get Next 7 Days1 | Twilio SMS2 | ## 3. Next 7-day Weekly Sunday SMS Review |
| Twilio SMS2 | twilio | Send weekly preview SMS | Format Weekly Summary1 | Append to Execution Log9 | ## 3. Next 7-day Weekly Sunday SMS Review |
| Append to Execution Log9 | googleSheets | Append weekly run result to Execution Log sheet | Twilio SMS2 | Update or append daily stats3 | ## 3. Next 7-day Weekly Sunday SMS Review |
| Update or append daily stats3 | googleSheets | Aggregate/update Daily Stats (dup node) | Append to Execution Log9 | — | ## 3. Next 7-day Weekly Sunday SMS Review |
| Sticky Note | stickyNote | Canvas header for block 1 | — | — | ## Auto Calendar Notifications and Real-Time Reporting with n8n and App Script *(see Sticky Note3 for full content)* |
| Sticky Note1 | stickyNote | Canvas header for block 2 | — | — | ## Auto Calendar Notifications and Real-Time Reporting with n8n and App Script *(see Sticky Note3 for full content)* |
| Sticky Note2 | stickyNote | Canvas header for block 3 | — | — | ## Auto Calendar Notifications and Real-Time Reporting with n8n and App Script *(see Sticky Note3 for full content)* |
| Sticky Note3 | stickyNote | Setup/how-it-works + external resource links | — | — | ## Auto Calendar Notifications and Real-Time Reporting with n8n and App Script *(contains links listed in section 5)* |
| Sticky Note4 | stickyNote | Note about Mailchimp vs SendGrid | — | — | ## Can work with MailChimp by replacing with SendGrid - Standard plan or higher |

> Note: Sticky notes do not execute; they are included because the requirements say not to skip any nodes.

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Google Calendar OAuth2 credential (Google Cloud project; enable Google Calendar API; OAuth consent; scopes to read events).
   2. Google Sheets OAuth2 credential (scopes to read/write sheets).
   3. Twilio credential (Account SID + Auth Token).
   4. SendGrid API credential (API key with mail send permissions).
   5. (Optional) Mailchimp credential if you plan to use Mandrill/Mailchimp Transactional.

2. **Prepare Google Sheets analytics workbook**
   1. Create/duplicate the provided “Calendar Analytics” spreadsheet (or use the linked 3-sheet template in Notes).
   2. Ensure there is an **Execution Log** sheet and a **Daily Stats** sheet.
   3. Place headers so that:
      - Execution Log header row is **row 3**.
      - Daily Stats header row is **row 3**, and data starts at **row 4**.
   4. Confirm column names match what you will write: “Execution ID”, “Workflow Name”, “Status”, “Duration (s)”, “Events Processed”, “Start Time”, “End Time”, “Error Message”, plus Daily Stats fields.

3. **Daily Email path**
   1. Add **Schedule Trigger** node named **“6 AM Daily Trigger”** with cron: `0 6 * * *`.
   2. Add **Google Calendar** node named **“Get Today’s Events”**:
      - Operation: Get All
      - Calendar: select your calendar ID
      - Options: `singleEvents: true`, `orderBy: startTime`
      - `timeMin = {{$now.startOf('day').toISO()}}`
      - `timeMax = {{$now.endOf('day').toISO()}}`
      - Credentials: Google Calendar OAuth2
   3. Add **Code** node named **“Format HTML Email”** and paste the JS that:
      - reads `$input.all()`
      - builds `emailBody`, `eventCount`, `subject`
   4. Add **SendGrid** node named **“Send an email”**:
      - Resource: Mail
      - To: your recipient(s)
      - From email/name: configured sender
      - Subject: `={{ $json.subject }}`
      - Content type: `text/html`
      - Content: `={{ $json.emailBody }}`
   5. Add **Google Sheets** node named **“Append to Execution Log5”**:
      - Operation: Append
      - Document: your analytics spreadsheet
      - Sheet: Execution Log
      - Header row: 3
      - Map columns (examples):
        - Status: `success`
        - Start Time: `={{ new Date().toISOString() }}`
        - End Time: `={{ new Date().toISOString().split('T')[0] }}`
        - Workflow Name: `📧 Daily Email Summary`
        - Events Processed: `={{ $('Format HTML Email').item.json.eventCount }}`
        - Execution ID: `={{ $json.messageId || 'N\\A' }}`
        - Error Message: blank
   6. Add **Google Sheets** node named **“Update or append daily stats”**:
      - Operation: Append or Update
      - Sheet: Daily Stats
      - Header row: 3, first data row: 4
      - Match on column: `Date`
      - Date: `={{ $json["End Time"] }}`
      - Add the COUNTIFS/SUMIFS formulas as cell formulas in the mapped values (as in the workflow).

   7. Connect:  
      `6 AM Daily Trigger → Get Today’s Events → Format HTML Email → Send an email → Append to Execution Log5 → Update or append daily stats`

4. **Daily SMS path**
   1. Add **Schedule Trigger** named **“7 AM Daily Trigger”** with cron: `0 7 * * *`.
   2. Add **Google Calendar** node **“Get Today’s Events1”** with the same config as “Get Today’s Events”.
   3. Add **Code** node **“Format SMS Message”** and paste the SMS-building JS.
   4. Add **Twilio** node **“Twilio SMS”**:
      - To: recipient phone number
      - From: Twilio number
      - Message: `={{ $('Format SMS Message').item.json.smsBody }}`
   5. Add **Google Sheets** node **“Append to Execution Log7”** (Append to Execution Log) similar to email, but:
      - Workflow Name: `📱 Daily SMS Summary`
      - Events Processed: `={{ $('Format SMS Message').item.json.eventCount }}`
   6. Add **Google Sheets** node **“Update or append daily stats1”** same settings as the stats updater.
   7. Connect:  
      `7 AM Daily Trigger → Get Today’s Events1 → Format SMS Message → Twilio SMS → Append to Execution Log7 → Update or append daily stats1`

5. **15-minute reminder path**
   1. Add **Schedule Trigger** named **“Check Every 5 Minutes1”**:
      - Set interval to **every 5 minutes** (explicitly configure; do not rely on defaults).
   2. Add **Google Calendar** node **“Get Events in Next 20 Min1”**:
      - `timeMin = {{$now.toISO()}}`
      - `timeMax = {{$now.plus({ minutes: 20 }).toISO()}}`
      - `singleEvents: true`, `orderBy: startTime`
   3. Add **Code** node **“Format Reminder SMS1”** and paste the JS.
   4. Add **Twilio** node **“Twilio SMS1”** with message:
      - `={{ $('Format Reminder SMS1').item.json.smsBody }}`
   5. Add **Google Sheets** node **“Append to Execution Log8”**:
      - Workflow Name: `⏰ 15-Minute Reminders`
      - Events Processed: `={{ $('Format Reminder SMS1').item.json.eventCount }}`
   6. Add **Google Sheets** node **“Update or append daily stats2”** (same config).
   7. Connect:  
      `Check Every 5 Minutes1 → Get Events in Next 20 Min1 → Format Reminder SMS1 → Twilio SMS1 → Append to Execution Log8 → Update or append daily stats2`

6. **Weekly preview path**
   1. Add **Schedule Trigger** named **“Sunday 6 PM Trigger1”** with cron: `0 18 * * 0`.
   2. Add **Google Calendar** node **“Get Next 7 Days1”**:
      - `timeMin = {{$now.plus({ days: 1 }).startOf('day').toISO()}}`
      - `timeMax = {{$now.plus({ days: 8 }).startOf('day').toISO()}}`
      - `singleEvents: true`, `orderBy: startTime`
   3. Add **Code** node **“Format Weekly Summary1”** and paste the JS.
   4. Add **Twilio** node **“Twilio SMS2”** with message:
      - `={{ $('Format Weekly Summary1').item.json.smsBody }}`
   5. Add **Google Sheets** node **“Append to Execution Log9”**:
      - Workflow Name: `📅 Weekly Preview`
      - Events Processed: `={{ $('Format Weekly Summary1').item.json.eventCount }}`
   6. Add **Google Sheets** node **“Update or append daily stats3”** (same config).
   7. Connect:  
      `Sunday 6 PM Trigger1 → Get Next 7 Days1 → Format Weekly Summary1 → Twilio SMS2 → Append to Execution Log9 → Update or append daily stats3`

7. **(Optional) Add canvas documentation**
   - Add sticky notes with the same headings and setup links for maintainability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Can use MailChimp or SendGrid to send email to the team | Sticky Note3 |
| “Twillo node is used to send sms reminders to the team” | Sticky Note3 (typo in note; node is Twilio) |
| Twilio setup | https://console.twilio.com/ |
| Google Cloud setup (enable Google Calendar API) | https://console.cloud.google.com/ |
| Get 3-sheet Google Sheet template | https://docs.google.com/spreadsheets/d/1S91eS0LV51DZ_9KkA20tfns27umyOU5V/edit?usp=drive_link&ouid=113513617635127854147&rtpof=true&sd=true |
| Get App Script Code | https://drive.google.com/file/d/1iamkTByQ00aNVsjkr7Nfz1mtn9_b03g9/view?usp=drive_link |
| Get the Report WebPage site | https://drive.google.com/file/d/1R0-v8nMEOjYDkaodhsFxo-eTXIByE6fP/view?usp=sharing |
| “For Mailchimp Transactional (Mandrill), you may need to use HTTP Request node instead” | Node note on “Send Email - Mailchimp Transactional” |
| “Can work with MailChimp by replacing with SendGrid - Standard plan or higher” | Sticky Note4 |

