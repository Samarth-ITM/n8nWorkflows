Send customer visit notifications from Google Calendar to Slack and Sheets

https://n8nworkflows.xyz/workflows/send-customer-visit-notifications-from-google-calendar-to-slack-and-sheets-13111


# Send customer visit notifications from Google Calendar to Slack and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send customer visit notifications from Google Calendar to Slack and Sheets  
**Workflow name (n8n):** Customer Visit Notification

This workflow runs hourly (06:00–18:00) to detect Google Calendar events that indicate **a customer will visit** (“to come”, “to arrive”, “visit”). It sends:
- **Partner-specific notifications one day in advance** (direct Slack messages to attendees mapped from a Google Sheet).
- **Company-wide announcements for today** (posts to a public Slack channel with an uploaded image + message).
- **Logging to Google Sheets** to avoid duplicate notifications and to record notification status.

### Logical blocks
1.1 **Scheduling + shared variables/time utilities**  
1.2 **Tomorrow: detect visit events + deduplicate vs Sheet**  
1.3 **Tomorrow: resolve attendees → Slack IDs + DM partners + log to Sheet**  
1.4 **Today (branch A): company-wide announcement + upload image + append/update Sheet**  
1.5 **Today (branch B): similar announcement flow + append/update Sheet** (appears duplicated; likely different calendar/sheet targets)

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling + shared variables/time utilities
**Overview:** Triggers the workflow every hour during business hours and computes date strings (today/tomorrow/last week) used by subsequent Google Calendar queries and folder naming.  
**Nodes involved:**
- Step 1: Activation schedule: Every 1 hour, from 6 AM to 6 PM.
- Step 2: Set Variables
- Step 3: Get datetime

#### Node details
**Step 1: Activation schedule…**  
- **Type/role:** Schedule Trigger; entrypoint.  
- **Config:** Cron expression `0 6-18/1 * * *` (at minute 0, every hour from 06 to 18 daily).  
- **Connections:** → Step 2.  
- **Edge cases:** Instance timezone affects when “6 AM” occurs; consider aligning n8n timezone with business timezone.

**Step 2: Set Variables**  
- **Type/role:** Set node; initializes shared constants.  
- **Config choices:** Raw JSON output with fields:  
  - `redmine6_Url`: empty (unused in this workflow)  
  - `Hour`: `{{ $json.Hour }}` (**problem:** upstream doesn’t provide `Hour`; it will be `undefined` unless n8n injects it elsewhere)  
  - `limit`: 100  
  - `channel_id`: empty (**must be set for company-wide Slack posting**)  
  - `calendar_event_web_search_read`: empty (unused)  
- **Connections:** → Step 3.  
- **Edge cases/failures:**  
  - `Hour` is likely intended to be “current hour”; but it isn’t computed. This breaks Steps 27/45 (>= 8 AM checks).

**Step 3: Get datetime**  
- **Type/role:** Code node; computes date strings in **UTC+7** (VN time) and reporting folder names.  
- **Key outputs:** `today`, `tomorrow`, `yesterday`, last-week start/end, plus folder naming strings.  
- **Connections:** → Step 4 (tomorrow events).  
- **Edge cases:**  
  - Uses manual UTC+7 shift; if your server timezone changes, this still forces VN time.  
  - `getDayOfWeek()` uses `date.getDay()` but the `days` array starts with Monday; JS `getDay()` returns Sunday=0. This mislabels day names (minor since not used later).

---

### 2.2 Tomorrow: detect visit events + deduplicate vs Sheet
**Overview:** Pulls tomorrow’s calendar events, filters to “customer visit” patterns, and removes those already marked “Sent” in a Google Sheet.  
**Nodes involved:**
- Step 4: Get the event information from Google Calendar for tomorrow.
- Step 5: Check if there is a record or not?
- Step 6: Filter out events…
- Step 7: Check if there is a record or not?
- Step 8: Read the Odoo Calendar Event notifications that have been sent.
- Step 9: Read Google Sheets member slack
- Step 10: Filter out records that have not been sent.
- Step 11: Check if there is a record or not?

#### Node details
**Step 4: Get the event information… tomorrow**  
- **Type/role:** Google Calendar “Get All”.  
- **Config:** time window: `tomorrow 00:05:00` → `tomorrow 23:55:00`, limit 100. Calendar selection is **blank** (`value: ""`).  
- **Connections:** → Step 5.  
- **Credentials:** Google Calendar OAuth2.  
- **Edge cases/failures:**  
  - Blank calendar ID will fail or default unpredictably; must set calendar.  
  - All-day events use `start.date` not `start.dateTime`; later code expects `dateTime` and may output `null`.

**Step 5: Check if there is a record or not?**  
- **Type/role:** IF; checks Calendar output not empty.  
- **True path:** → Step 6 (tomorrow partner notifications).  
- **False path:** → Step 38 (today announcement branch B).  
- **Edge cases:** Logic assumes “no tomorrow events” means run another branch; this coupling can be surprising.

**Step 6: Filter out events… (tomorrow)**  
- **Type/role:** Filter; selects events whose `summary` matches regex `(to come|to arrive|visit)` (case-insensitive).  
- **Connections:** → Step 7.  
- **Edge cases:**  
  - If your event naming differs, nothing matches.  
  - If `summary` is missing, filter may drop or error depending on strictness.

**Step 7: Check if there is a record or not?**  
- **Type/role:** IF; checks filtered events not empty.  
- **True path:** → Step 8  
- **False path:** ends (no notifications).

**Step 8: Read the Odoo Calendar Event notifications that have been sent.**  
- **Type/role:** Google Sheets read (operation not explicitly shown; defaults to “read/getAll” behavior).  
- **Config:** Document ID is **blank/expression `=`**, Sheet name blank.  
- **Connections:** → Step 9.  
- **Edge cases:** Must configure document + sheet; otherwise fails.

**Step 9: Read Google Sheets member slack**  
- **Type/role:** Google Sheets read; mapping table (Email → memberID).  
- **Config:** Document ID + sheet name blank.  
- **Connections:** → Step 10.

**Step 10: Filter out records that have not been sent.**  
- **Type/role:** Code; deduplicates tomorrow visit events.  
- **Logic:**  
  - Collect sent IDs where sheet column **“The day before” == "Sent"** and `ID` column stores event id.  
  - Map Calendar items to normalized fields:
    - `odoo_calendar_event_id` = event `id`
    - `odoo_calendar_event_name` = `summary` with bracket tags removed (`[ ... ]`)
    - `odoo_calendar_event_start/stop` = `start.dateTime`/`end.dateTime` formatted
    - `attendees` passed through
  - Filter out IDs already in sheet.  
- **Connections:** → Step 11.  
- **Edge cases:**  
  - If sheet column names differ (“The day before”, “ID”), dedupe fails and duplicates are sent.  
  - If event has no attendees, later partner loop won’t send anything.  
  - If time includes `Z` instead of `+07:00`, the regex removing timezone won’t match.

**Step 11: Check if there is a record or not?**  
- **Type/role:** IF; checks whether Step 10 output is empty.  
- **Important:** Condition uses operator **“empty”**.  
  - If **empty** → nothing to do.  
  - If **not empty** → sends to Step 12 (connected on the *false* branch).  
- **Connections:** False branch → Step 12.  
- **Edge cases:** Easy to misread; keep branch semantics clear.

---

### 2.3 Tomorrow: resolve attendees → Slack IDs + DM partners + log to Sheet
**Overview:** Iterates unsent events, maps each attendee email to a Slack member/channel ID from a Google Sheet, sends a DM per partner, then appends event info to a log sheet.  
**Nodes involved:**
- Step 12: Loop Over Items
- Step 13: Extract the email and member ID of the partners.
- Step 14: Loop Over Partners.
- Step 15: Wait 1s
- Step 16: Send a message to your Partners
- Step 17: Return output of step8
- Step 18: Loop Over Items
- Step 19: Fill in the Calendar Event information in the Google Sheet.

#### Node details
**Step 12: Loop Over Items**  
- **Type/role:** SplitInBatches; iterates each unsent event from Step 10.  
- **Connections:**  
  - Output 0 → Step 14 (partner loop)  
  - Output 1 → Step 13 (email→memberID mapping)  
- **Edge cases:** Default batch size behavior: ensure it processes all items; if batch size is default 1, it loops properly.

**Step 13: Extract the email and member ID of the partners.**  
- **Type/role:** Code; creates per-attendee notification targets.  
- **Logic:** For each `attendees[]` item, match attendee email to sheet records by:
  - exact email (score 100)
  - same local-part before `@` (score 80)
  - contains local-part (score 50)
  - keep matches with score ≥ 50  
- **Outputs:** array of `{ email, memberID, odoo_calendar_event_name, start, stop }`  
- **Connections:** → Step 12 (wired back) (in connections, Step 13 → Step 12).  
- **Edge cases:**  
  - If sheet column names are not exactly `Email` and `memberID`, mapping breaks.  
  - Same local-part across domains may mismatch.

**Step 14: Loop Over Partners.**  
- **Type/role:** SplitInBatches; iterates each partner entry (from Step 13 results).  
- **Connections:**  
  - Output 0 → Step 17 (to later log)  
  - Output 1 → Step 15 → Step 16 (send DM)  
- **Edge cases:** If Step 13 outputs zero items, no Slack sends occur.

**Step 15: Wait 1s**  
- **Type/role:** Wait; throttling for Slack API.  
- **Connections:** → Step 16.  
- **Edge cases:** For large attendee lists, 1s delay per message increases runtime.

**Step 16: Send a message to your Partners**  
- **Type/role:** HTTP Request to Slack `chat.postMessage`.  
- **Auth:** Bearer token (Slack Bot token) via `httpBearerAuth` credential. (A second credential `httpQueryAuth` is attached but not used by this request.)  
- **Body:**  
  - `channel` = partner `memberID` (could be a user ID for DM or a channel ID)  
  - `text` = formatted message with event name/start/stop  
- **Error handling:** `onError: continueRegularOutput` + retries.  
- **Connections:** → Step 14 (loop continuation).  
- **Edge cases/failures:**  
  - If `memberID` is not a valid Slack channel/user ID, Slack returns `channel_not_found`.  
  - If bot lacks permission to DM users, returns `not_in_channel`/`cannot_dm_user`.  
  - Slack rate limits (HTTP 429); retries help but may still fail—consider backoff using headers.

**Step 17: Return output of step8**  
- **Type/role:** Code; returns all items from Step 10 (despite node name referencing step 8).  
- **Connections:** → Step 18.  
- **Purpose:** Rehydrates the unsent event list to log it once partner loop completes.

**Step 18: Loop Over Items**  
- **Type/role:** SplitInBatches; iterates events for logging.  
- **Connections:**  
  - Output 0 → Step 20 (today flow A trigger)  
  - Output 1 → Step 19 (append log)  
- **Edge cases:** This creates a coupling: logging loop also triggers “today” flow.

**Step 19: Fill in the Calendar Event information in the Google Sheet.**  
- **Type/role:** Google Sheets Append.  
- **Config:** Document ID and sheet name are blank expressions; must be set.  
- **Connections:** → Step 18 (loop continues).  
- **Edge cases:** Column mapping not shown; append requires fields to match sheet columns.

---

### 2.4 Today (branch A): company-wide announcement + Slack file upload + append/update Sheet
**Overview:** Builds a single message for all unsent “today” visit events, uploads an image to Slack using the external upload flow, posts to a public channel, then updates/records sent status in Sheets.  
**Nodes involved:**
- Step 20: Get the event information from Google Calendar for today..
- Step 21: Check if there is a record or not?
- Step 22: Filter out events…
- Step 23: Check if there is a record or not?
- Step 24: Read the Odoo Calendar Event notifications that have been sent.
- Step 25: Filter out records that have not been sent.
- Step 26: Check if there is a record or not?
- Step 27: Check if it is currently >= 8 AM?
- Step 28: Convert to text
- Step 29: Download file image
- Step 30: Convert KB → bytes
- Step 31: Call Slack API’s getUploadURLExternal API
- Step 32: Choose branch 2
- Step 33: POST upload_url Slack
- Step 34: Call completeUploadExternal Slack API
- Step 35: Return output of step 25
- Step 36: Loop Over Items
- Step 37: Fill in the Calendar Event information in the Google Sheet.

#### Node details (key points)
**Step 20 (Google Calendar today)**  
- Same as Step 4 but for `today`. Calendar is blank; must set.

**Step 22 Filter (today)**  
- Same regex filter on `summary`.

**Step 24 Read sent-log sheet**  
- Uses column **“Today” == "Sent"** in Step 25.

**Step 25 Deduplicate (today)**  
- Same mapping logic as Step 10; checks “Today” column.

**Step 26 IF empty**  
- If Step 25 is empty → stop  
- Else → Step 27 (via false branch)

**Step 27 IF current hour >= 8**  
- Compares `Step 2: Set Variables`.json.Hour >= 8.  
- **Issue:** `Hour` is not computed anywhere; this likely prevents announcements or behaves inconsistently.  
- True → Step 28.

**Step 28 Convert to text**  
- Builds `<!channel> :mega:` message containing all events (name/from/to).  
- Replaces real newlines with literal `\n` for Slack JSON embedding.

**Steps 29–34 Slack image upload + post**  
- Step 29 downloads image from Google Drive (fileId blank).  
- Step 30 computes binary byte length; sets `fileName`.  
- Step 31 calls `files.getUploadURLExternal` (Slack) with filename + length.  
- Step 32 merge “chooseBranch” uses data of input 2 to ensure the binary data flows to Step 33 while still having `upload_url` from Step 31.  
- Step 33 POSTs binary to Slack-provided `upload_url` with `Content-Type: application/octet-stream`.  
- Step 34 completes upload and posts to `channel_id` from Step 2, with `initial_comment` from Step 28 message.  
- **Edge cases:**  
  - Slack external upload requires the right scopes (commonly `files:write`, `chat:write`).  
  - `channel_id` is blank in Step 2; must be set.  
  - Google Drive fileId blank; must be set.  
  - `alwaysOutputData: false` + `onError: continueRegularOutput`: failures may silently skip posting; consider logging Slack response `ok` flag.

**Step 35: Return output of step 25**  
- **Bug:** Code returns `$items("Step 43: Filter out records that have not been sent.")` (from the other branch), not Step 25.  
- This can break downstream logging (Step 36/37) or log wrong records.

**Step 36 Loop Over Items**  
- SplitInBatches; in connections, only output index 1 goes to Step 37, index 0 is unused. This is unusual; likely miswired.

**Step 37 AppendOrUpdate**  
- Google Sheets appendOrUpdate; documentId/sheetName blank.  
- Intended to mark events as “Sent” for today and/or store message metadata.

---

### 2.5 Today (branch B): duplicated company announcement flow + append/update Sheet
**Overview:** A second “today” path starting from Step 38, functionally similar to block 2.4, but with slightly different calendar selection mode and different node IDs. This likely supports a second calendar or sheet tab.  
**Nodes involved:**
- Step 38 → 55 (full chain): 38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55

#### Notable differences / issues
- **Step 38** uses calendar mode “id” (still blank value).  
- **Step 43** deduplicates against sheet column “Today”.  
- **Step 45** has the same broken `Hour` dependency.  
- **Steps 47+** download a Drive file by ID (blank).  
- **Step 53** correctly returns `$items("Step 43…")`.  
- **Step 55** appendOrUpdate similar to Step 37.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Step 1: Activation schedule: Every 1 hour, from 6 AM to 6 PM. | scheduleTrigger | Time-based entrypoint | — | Step 2 | # Customer Visit Notification (full overview + setup + links) |
| Step 2: Set Variables | set | Initialize shared variables | Step 1 | Step 3 | ## 1. Set Variables and Get datetime |
| Step 3: Get datetime | code | Compute today/tomorrow strings (UTC+7) | Step 2 | Step 4 | ## 1. Set Variables and Get datetime |
| Step 4: Get the event information from Google Calendar for tomorrow. | googleCalendar | Fetch tomorrow events | Step 3 | Step 5 | ## 2. The system sends a notification to partners one day in advance. |
| Step 5: Check if there is a record or not? | if | Branch: tomorrow events exist? | Step 4 | Step 6 / Step 38 | ## 2. The system sends a notification to partners one day in advance. |
| Step 6: Filter out events whose titles match the action of a customer visiting the company. | filter | Keep “visit” events | Step 5 | Step 7 | ## 2. The system sends a notification to partners one day in advance. |
| Step 7: Check if there is a record or not? | if | Ensure filtered list not empty | Step 6 | Step 8 | ## 2. The system sends a notification to partners one day in advance. |
| Step 8: Read the Odoo Calendar Event notifications that have been sent. | googleSheets | Read sent-log for “day before” | Step 7 | Step 9 | ## 2. The system sends a notification to partners one day in advance. |
| Step 9: Read Google Sheets member slack | googleSheets | Read email→Slack memberID mapping | Step 8 | Step 10 | ## 2. The system sends a notification to partners one day in advance. |
| Step 10: Filter out records that have not been sent. | code | Deduplicate tomorrow events vs sheet | Step 9 | Step 11 | ## 2. The system sends a notification to partners one day in advance. |
| Step 11: Check if there is a record or not? | if | Stop if nothing to send | Step 10 | Step 12 (false branch) | ## 2. The system sends a notification to partners one day in advance. |
| Step 12: Loop Over Items | splitInBatches | Loop through unsent tomorrow events | Step 11 | Step 14 / Step 13 | ## 2. The system sends a notification to partners one day in advance. |
| Step 13: Extract the email and member ID of the partners. | code | Map attendees emails to Slack IDs | Step 12 | Step 12 | ## 2. The system sends a notification to partners one day in advance. |
| Step 14: Loop Over Partners. | splitInBatches | Loop per partner target | Step 12 / Step 16 | Step 17 / Step 15 | ## 2. The system sends a notification to partners one day in advance. |
| Step 15: Wait 1s | wait | Slack throttling | Step 14 | Step 16 | ## 2. The system sends a notification to partners one day in advance. |
| Step 16: Send a message to your Partners | httpRequest | Slack chat.postMessage (DM) | Step 15 | Step 14 | ## 2. The system sends a notification to partners one day in advance. |
| Step 17: Return output of step8 | code | Restore event list for logging | Step 14 | Step 18 | ## 2. The system sends a notification to partners one day in advance. |
| Step 18: Loop Over Items | splitInBatches | Loop for sheet append | Step 17 / Step 19 | Step 20 / Step 19 | ## 3. Fill in the Calendar Event information in the Google Sheet. |
| Step 19: Fill in the Calendar Event information in the Google Sheet. | googleSheets | Append event to log sheet | Step 18 | Step 18 | ## 3. Fill in the Calendar Event information in the Google Sheet. |
| Step 20: Get the event information from Google Calendar for today.. | googleCalendar | Fetch today events (branch A) | Step 18 | Step 21 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 21: Check if there is a record or not? | if | Ensure today events exist | Step 20 | Step 22 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 22: Filter out events… | filter | Keep “visit” events (today) | Step 21 | Step 23 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 23: Check if there is a record or not? | if | Ensure filtered list not empty | Step 22 | Step 24 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 24: Read the Odoo Calendar Event notifications that have been sent. | googleSheets | Read sent-log (today) | Step 23 | Step 25 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 25: Filter out records that have not been sent. | code | Deduplicate today events vs sheet | Step 24 | Step 26 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 26: Check if there is a record or not? | if | Stop if nothing new | Step 25 | Step 27 (false branch) | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 27: Check if it is currently >= 8 AM? | if | Time gate for announcements | Step 26 | Step 28 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 28: Convert to text | code | Build Slack announcement text | Step 27 | Step 29 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 29: Download file image | googleDrive | Download image for Slack post | Step 28 | Step 30 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 30: Convert KB → bytes | code | Compute binary size + filename | Step 29 | Step 31 / Step 32 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 31: Call Slack API's getUploadURLExternal API | httpRequest | Slack external upload init | Step 30 | Step 32 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 32: Choose branch 2 | merge | Keep binary branch for upload | Step 30 / Step 31 | Step 33 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 33: POST upload_url Slack | httpRequest | Upload binary to Slack URL | Step 32 | Step 34 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 34: Call the completeUploadExternal Slack API to complete the file upload. | httpRequest | Finalize upload + post to channel | Step 33 | Step 35 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 35: Return output of step 25 | code | (Intended) restore today list; (actual) references Step 43 | Step 34 | Step 36 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 36: Loop Over Items | splitInBatches | Loop for sheet update | Step 35 / Step 37 | Step 37 (index 1 only) | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 37: Fill in the Calendar Event information in the Google Sheet. | googleSheets | AppendOrUpdate “sent today” | Step 36 | Step 36 | ## 4. The system sends a company-wide announcement to a public channel today and updates a note in the Google Sheet. |
| Step 38: Get the event information from Google Calendar for today. | googleCalendar | Fetch today events (branch B) | Step 5 | Step 39 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 39: Check if there is a record or not? | if | Ensure events exist | Step 38 | Step 40 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 40: Filter out events… | filter | Keep “visit” events | Step 39 | Step 41 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 41: Check if there is a record or not? | if | Ensure filtered list not empty | Step 40 | Step 42 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 42: Read the Odoo Calendar Event notifications that have been sent. | googleSheets | Read sent-log (today) | Step 41 | Step 43 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 43: Filter out records that have not been sent. | code | Deduplicate today events | Step 42 | Step 44 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 44: Check if there is a record or not? | if | Stop if none | Step 43 | Step 45 (false branch) | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 45: Check if it is currently >= 8 AM? | if | Time gate | Step 44 | Step 46 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 46: Convert to text | code | Build Slack announcement text | Step 45 | Step 47 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 47: Download file image | googleDrive | Download image | Step 46 | Step 48 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 48: Convert KB → bytes | code | Compute binary size + filename | Step 47 | Step 49 / Step 50 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 49: Call Slack API's getUploadURLExternal API | httpRequest | Slack external upload init | Step 48 | Step 50 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 50: Choose branch 2 | merge | Keep binary branch | Step 48 / Step 49 | Step 51 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 51: POST upload_url Slack | httpRequest | Upload binary | Step 50 | Step 52 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 52: Call the completeUploadExternal Slack API to complete the file upload. | httpRequest | Finalize upload + post | Step 51 | Step 53 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 53: Return output of step 43 | code | Restore today list for logging | Step 52 | Step 54 | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 54: Loop Over Items | splitInBatches | Loop for sheet update | Step 53 / Step 55 | Step 55 (index 1 only) | ## 5. The system sends a company-wide announcement to a public channel today |
| Step 55: Fill in the Calendar Event information in the Google Sheet. | googleSheets | AppendOrUpdate “sent today” | Step 54 | Step 54 | ## 5. The system sends a company-wide announcement to a public channel today |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow** named “Customer Visit Notification”.

2) **Add Trigger**
   1. Add **Schedule Trigger**  
      - Interval: Cron  
      - Cron: `0 6-18/1 * * *`

3) **Add shared variables**
   1. Add **Set** node “Step 2: Set Variables” (Mode: Raw JSON) and set:
      - `limit`: 100  
      - `channel_id`: `<YOUR_PUBLIC_SLACK_CHANNEL_ID>`  
      - (Optional) remove unused fields (`redmine6_Url`, `calendar_event_web_search_read`)  
      - **Fix Hour**: set `Hour` properly, e.g. by computing in code (recommended), not `{{$json.Hour}}`.

4) **Add date utility code**
   1. Add **Code** node “Step 3: Get datetime” with the provided VN-time logic (or adapt to your timezone).  
   2. Connect: Schedule Trigger → Set Variables → Get datetime.

5) **Tomorrow partner notification branch**
   1. Add **Google Calendar** node “Step 4…” (Operation: Get All)  
      - Calendar: choose your calendar  
      - TimeMin: `{{$node["Step 3: Get datetime"].json.tomorrow}} 00:05:00`  
      - TimeMax: `{{$node["Step 3: Get datetime"].json.tomorrow}} 23:55:00`  
      - Limit: `100`
      - Credentials: **Google Calendar OAuth2** (configure client/app + connect account)
   2. Add **IF** “Step 5…” to check not empty (value: Step 4 json).  
   3. Add **Filter** “Step 6…”:
      - Field: `summary`
      - Regex: `(to come|to arrive|visit)`; ignore case enabled
   4. Add **IF** “Step 7…” to check filtered not empty.
   5. Add **Google Sheets** “Step 8…” to read your notification log sheet:
      - Credentials: **Google Sheets OAuth2**
      - Document: select spreadsheet
      - Sheet: select tab (must contain columns like `ID`, `The day before`, `Today`)
   6. Add **Google Sheets** “Step 9…” to read the member mapping:
      - Spreadsheet/tab containing columns: `Email`, `memberID`
   7. Add **Code** “Step 10…” with dedupe logic (sent where “The day before” == Sent).
   8. Add **IF** “Step 11…” to stop when Step 10 returns empty.
   9. Add **SplitInBatches** “Step 12…” (batch size 1 recommended).
   10. Add **Code** “Step 13…” to map attendees emails → memberID from Step 9.
   11. Add **SplitInBatches** “Step 14…” to loop partners.
   12. Add **Wait** “Step 15…” amount 1 second.
   13. Add **HTTP Request** “Step 16…”:
       - POST `https://slack.com/api/chat.postMessage`
       - Auth: **Bearer token** (Slack Bot User OAuth Token)
       - Body params:
         - `channel` = `{{$node["Step 14: Loop Over Partners."].json.memberID}}`
         - `text` = formatted notification text using event fields
       - Slack app scopes: at least `chat:write` (and DM-related permissions depending on usage)
   14. Add **Code** “Step 17…” to output Step 10 items for logging.
   15. Add **SplitInBatches** “Step 18…” then **Google Sheets Append** “Step 19…” to log the “day before sent” status.
   16. Wire loops exactly:
       - Step 14 → Wait → HTTP → Step 14 (loop)
       - Step 14 (other output) → Step 17 → Step 18 → Step 19 → Step 18 (loop)

6) **Today company-wide announcement**
   - Implement **one** of the two “today” branches (they’re near-duplicates). Recommended: keep a single clean branch.
   1. Add **Google Calendar Get All** for today (timeMin/timeMax same pattern but with `today`).
   2. Filter titles with the same regex.
   3. Read the sent-log Google Sheet.
   4. Code node to dedupe where column “Today” == “Sent”.
   5. IF: stop if empty.
   6. IF time gate:
      - **Fix Hour** by computing current hour (in the same datetime code node, add `hour: now.getHours()` and reference it).
   7. Code: build `<!channel>` message text.
   8. Google Drive Download:
      - Set `fileId` to the image file ID to attach
      - Credentials: Google Drive OAuth2
   9. Code: compute binary size + filename.
   10. HTTP Request: Slack `files.getUploadURLExternal` (Bearer auth).
   11. Merge (chooseBranch) to pass the binary to upload step.
   12. HTTP Request: POST binary to `upload_url`.
   13. HTTP Request: Slack `files.completeUploadExternal`:
       - `channel_id`: from your Set Variables
       - `initial_comment`: from Convert-to-text code
   14. Google Sheets appendOrUpdate to mark events as “Sent” for Today.
   15. **Fix known bug:** ensure the “Return output…” code node references the correct dedupe node (Step 25, not Step 43).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| N8n Version 2.4.6; requires Google Calendar OAuth2, Google Sheets OAuth2, Slack Bot token; logs all notified events in Google Sheets | From sticky note overview |
| Intended audience: Reception/Admin, Sales/Account owners, Management/Team leads, Security/IT/Logistics | From sticky note overview |
| Customization/consulting contact | [Linkedin](https://www.linkedin.com/company/bac-ha-software/posts/?feedView=all) / [Website](https://bachasoftware.com/bhsoft-contacts) |
| Sticky block labels: “Set Variables and Get datetime”; “Notify partners one day in advance”; “Fill in Calendar Event information in Google Sheet”; “Company-wide announcement today (and update sheet)” | Sticky notes in workflow |

**Important implementation warnings (based on the current JSON):**
- Several required fields are blank: **calendar ID**, **Sheets documentId/sheetName**, **Drive fileId**, **Slack channel_id**.
- The “Hour” variable is not computed; time gating likely doesn’t work.
- Step 35 references Step 43 items (wrong branch), which can corrupt logging in the “today” flow.