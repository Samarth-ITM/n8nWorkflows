Schedule and confirm revenue ops meetings with Pipedrive, Google Calendar and Slack

https://n8nworkflows.xyz/workflows/schedule-and-confirm-revenue-ops-meetings-with-pipedrive--google-calendar-and-slack-12970


# Schedule and confirm revenue ops meetings with Pipedrive, Google Calendar and Slack

## 1. Workflow Overview

**Title:** Schedule and confirm revenue ops meetings with Pipedrive, Google Calendar and Slack  
**Workflow name (in JSON):** Pipedrive ops Automation  
**Purpose:** Automate scheduling and confirmation of Revenue Ops/sales meetings when a Pipedrive deal reaches a specific stage. The workflow waits briefly for an SDR to add meeting details, then creates a Google Calendar event, emails the client via Gmail, and notifies the team in Slack.

### 1.1 Deal Change Reception (Pipedrive)
Listens for changes to **deals** in Pipedrive and starts the workflow on any deal change event.

### 1.2 Timing Guard + Stage Qualification
Adds a short delay to give SDRs time to enter activity/meeting details, then checks that the deal has entered the **Meeting Booking** stage (stage_id = 2).

### 1.3 Deal Enrichment (Pipedrive API fetch)
Fetches full deal details from Pipedrive to ensure reliable fields (person, org, next activity details).

### 1.4 Data Normalization (meeting link + time calculations)
Extracts and reshapes key fields; normalizes start/end timestamps in a specific timezone (**Asia/Dubai**) for calendar creation.

### 1.5 Meeting Creation + Notifications
Creates the meeting in **Google Calendar**, then:
- Sends a **Gmail** reminder/confirmation to the client
- Posts a **Slack** reminder to an internal channel

---

## 2. Block-by-Block Analysis

### Block 1 — Deal Change Reception (Pipedrive)

**Overview:** Triggers whenever a deal is changed in Pipedrive. Provides event payload containing current deal data and previous values (e.g., previous stage).  
**Nodes involved:**  
- **Pipedrive Trigger**

#### Node: Pipedrive Trigger
- **Type / role:** `n8n-nodes-base.pipedriveTrigger` — webhook trigger from Pipedrive.
- **Configuration (interpreted):**
  - Watches **entity:** `deal`
  - On **action:** `change`
- **Credentials:** Pipedrive API credential (“Pipedrive account”).
- **Input/Output:**
  - **Output →** Wait node (“Wait untill the SDR submit…”)
- **Key payload fields used downstream:**
  - `{{$json.data.stage_id}}` for stage gating
  - `{{$json.data.id}}` for fetching full deal
- **Version-specific notes:** Type version `1.1`.
- **Edge cases / failures:**
  - Webhook not registered or disabled in Pipedrive → no executions.
  - Credential revoked/invalid → trigger setup fails.
  - Payload may not include required fields depending on Pipedrive permissions or webhook configuration.

---

### Block 2 — Timing Guard + Stage Qualification

**Overview:** Introduces a fixed wait to let SDRs populate meeting info, then filters events so the workflow only continues if the deal is in stage_id = 2 (Meeting Booking).  
**Nodes involved:**  
- **Wait untill the SDR submit the google meeting link from Deals in pipedrive**  
- **If stage_id is 2 (Meeting Booking)**

#### Node: Wait untill the SDR submit the google meeting link from Deals in pipedrive
- **Type / role:** `n8n-nodes-base.wait` — pauses execution for a configured duration.
- **Configuration (interpreted):**
  - Wait **amount:** `55` (units depend on node settings; in many n8n setups this is minutes or seconds based on UI choice—verify in your instance)
- **Input/Output:**
  - **Input ←** Pipedrive Trigger
  - **Output →** IF stage check
- **Version-specific notes:** Type version `1.1`.
- **Edge cases / failures:**
  - If meeting details are added after the wait window, downstream nodes may still see missing `next_activity` fields.
  - If n8n restarts and the instance isn’t configured to persist waiting executions, the wait may be interrupted.
  - If multiple deal changes occur rapidly, you may queue multiple waits for the same deal.

#### Node: If stage_id is 2 (Meeting Booking)
- **Type / role:** `n8n-nodes-base.if` — conditional branching.
- **Configuration (interpreted):**
  - Condition: numeric equals  
    - **Left:** `={{ $json.data.stage_id }}`
    - **Right:** `2`
- **Input/Output:**
  - **Input ←** Wait node
  - **True output →** Extract deal info
  - **False output →** not connected (workflow ends silently)
- **Version-specific notes:** Type version `2.3` (IF node “version 3” condition model).
- **Edge cases / failures:**
  - Stage IDs differ per Pipedrive pipeline; if stage_id 2 is not “Meeting Booking” in your account, the workflow won’t run as expected.
  - If the trigger payload lacks `data.stage_id`, strict validation can cause the condition to fail (no match).

---

### Block 3 — Deal Enrichment (Pipedrive)

**Overview:** Fetches the full deal record from Pipedrive using the deal ID from the webhook payload to ensure all needed fields are available for scheduling.  
**Nodes involved:**  
- **Extract deal info**

#### Node: Extract deal info
- **Type / role:** `n8n-nodes-base.pipedrive` — Pipedrive API call.
- **Configuration (interpreted):**
  - Operation: **Get deal**
  - Deal ID: `={{ $json.data.id }}`
- **Credentials:** Pipedrive API credential (“Pipedrive account”).
- **Input/Output:**
  - **Input ←** IF stage check (true branch)
  - **Output →** Set/transform node
- **Version-specific notes:** Type version `1`.
- **Edge cases / failures:**
  - Deal ID missing/invalid → API error.
  - Pipedrive rate limits or temporary API failures.
  - If the deal lacks `next_activity` or the activity lacks conference link/time/duration, downstream time parsing will break unless guarded.

---

### Block 4 — Data Normalization (meeting link + start/end calculation)

**Overview:** Extracts relevant deal/contact/org fields and calculates ISO timestamps for start and end times in **Asia/Dubai** timezone.  
**Nodes involved:**  
- **Get meeting link URL, Start time and end time**

#### Node: Get meeting link URL, Start time and end time
- **Type / role:** `n8n-nodes-base.set` — mapping + transformation.
- **Configuration (interpreted):**
  - Creates/overwrites fields (selected highlights):
    - `person_id.name` = `{{$json.person_id.name}}`
    - `person_id.email[0].value` = `{{$json.person_id.email[0].value}}`
    - `next_activity.conference_meeting_url` = `{{$json.next_activity.conference_meeting_url}}`
    - `next_activity.due_date` = `{{$json.next_activity.due_date}}`
    - `next_activity.due_time` = `={{ `${$json.next_activity.due_date}T${$json.next_activity.due_time}:00` }}`
    - `StartTime` = expression that:
      - accepts either full ISO or HH:mm(/HH:mm:ss)
      - builds an ISO datetime from `due_date` + `due_time`
      - sets timezone to `Asia/Dubai`
    - `EndTime` = expression that:
      - computes start as above
      - adds duration from `next_activity.duration` (expected `HH:mm`)
    - `org_id.name` = `{{$json.org_id.name}}`
- **Key expressions / requirements:**
  - Uses `DateTime.fromISO(...)` and `.setZone('Asia/Dubai')`
  - Assumes Luxon `DateTime` is available in n8n expressions (standard in recent n8n versions).
- **Input/Output:**
  - **Input ←** Extract deal info
  - **Output →** Google Calendar event creation
- **Version-specific notes:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing `next_activity.due_date` or `next_activity.due_time` → invalid ISO string → expression error.
  - Missing/invalid `next_activity.duration` → fallback to `00:00` results in zero-length meetings.
  - Timezone mismatch: if SDR enters time in another timezone, created event may be offset.
  - The assignment name `next_activity.due_time` is being overwritten with a combined datetime string; later nodes should not assume it remains only HH:mm.

---

### Block 5 — Meeting Creation + Notifications (Google Calendar → Gmail → Slack)

**Overview:** Creates a calendar event using normalized timestamps and meeting URL, then sends client email and internal Slack reminder.  
**Nodes involved:**  
- **Create a meeting and add you recipient**  
- **Send Email reminder to the client**  
- **Send a reminder to SDR in specific channel**

#### Node: Create a meeting and add you recipient
- **Type / role:** `n8n-nodes-base.googleCalendar` — creates a Google Calendar event.
- **Configuration (interpreted):**
  - Calendar: `user@example.com` (selected from list)
  - Start: `={{ $json.StartTime }}`
  - End: `={{ $json.EndTime }}`
  - Additional fields:
    - Summary: `"Test Google calnder"` (typo preserved from node)
    - Location: `={{ $json.next_activity.conference_meeting_url }}`
    - Attendees: `={{ $json.person_id.email[0].value }}`
- **Credentials:** Google Calendar OAuth2 (“Google Calendar account”).
- **Input/Output:**
  - **Input ←** Set/transform node
  - **Output →** Gmail node
- **Version-specific notes:** Type version `1.3`.
- **Edge cases / failures:**
  - OAuth token expired / missing scopes (calendar events) → auth errors.
  - Invalid attendee email format → event creation may fail.
  - Start/End invalid or end <= start → API rejection.
  - Using `location` for meeting URL works but some teams prefer `description` for richer info.

#### Node: Send Email reminder to the client
- **Type / role:** `n8n-nodes-base.gmail` — sends an email through Gmail.
- **Configuration (interpreted):**
  - To: `={{ $('Get meeting link URL, Start time and end time').item.json.person_id.email[0].value }}`
  - Subject: `=Meeting Reminder:{{ $('Get meeting link URL, Start time and end time').item.json.next_activity.due_date }}`
  - Body includes:
    - Recipient name from Set node
    - Meeting time formatted from Google Calendar output:
      - `{{ DateTime.fromISO($json.start.dateTime).toFormat('dd LLL yyyy, hh:mm a') }}`
  - Attribution disabled: `appendAttribution: false`
- **Credentials:** Gmail OAuth2 (“Gmail account”).
- **Input/Output:**
  - **Input ←** Google Calendar node (note: uses `$json.start.dateTime` from created event)
  - **Output →** Slack node
- **Version-specific notes:** Type version `2.2`.
- **Edge cases / failures:**
  - If Google Calendar node fails or returns a different shape (e.g., `start` not present), the `DateTime.fromISO($json.start.dateTime)` expression fails.
  - Gmail send limits, blocked “From” identity, or missing permissions.
  - If `person_id.email[0].value` is missing, sendTo expression fails.

#### Node: Send a reminder to SDR in specific channel
- **Type / role:** `n8n-nodes-base.slack` — posts a channel message.
- **Configuration (interpreted):**
  - Authentication: OAuth2
  - Channel: `meetings-reminder` (ID `C0AAP1C16S2`)
  - Text includes:
    - Meeting time from Google Calendar output (`$json.start.dateTime`)
    - Meeting link pulled from the Google Calendar node’s `location` field via:
      - `{{ $('Create a meeting and add you recipient').item.json.location }}`
- **Credentials:** Slack OAuth2 (“Slack account”).
- **Input/Output:**
  - **Input ←** Gmail node
  - **Output:** none (end)
- **Version-specific notes:** Type version `2.4`.
- **Edge cases / failures:**
  - Bot/user not in channel → `not_in_channel` error.
  - Missing `chat:write` scope → auth error.
  - If calendar node didn’t set `location`, link will be empty.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Pipedrive Trigger | n8n-nodes-base.pipedriveTrigger | Entry point: receives deal change webhooks | — | Wait untill the SDR submit the google meeting link from Deals in pipedrive | ### Pipedrive Trigger<br>**Why it exists**<br>Listens for deal updates so automation runs only when sales activity actually happens.<br><br>**You can change**<br>- Pipeline ID<br>- Stage ID (for example, use a different stage like *Demo Scheduled*)<br>- Trigger entity (deal vs activity) |
| Wait untill the SDR submit the google meeting link from Deals in pipedrive | n8n-nodes-base.wait | Delay to allow SDR to fill meeting details | Pipedrive Trigger | If stage_id is 2 (Meeting Booking) | ### Wait Node<br>**Why it exists**<br>Prevents the workflow from running before the SDR finishes adding meeting details.<br><br>**You can change**<br>- Wait time duration<br>- Replace with a conditional check instead of time-based waiting<br>- Remove it if meeting data is always available immediately |
| If stage_id is 2 (Meeting Booking) | n8n-nodes-base.if | Gate execution to Meeting Booking stage | Wait untill the SDR submit the google meeting link from Deals in pipedrive | Extract deal info | ### IF Stage Check<br>**Why it exists**<br>Ensures the workflow only runs for qualified deals and avoids unwanted executions.<br><br>**You can change**<br>- Stage ID value<br>- Add multiple allowed stages<br>- Add extra conditions like deal value or owner |
| Extract deal info | n8n-nodes-base.pipedrive | Fetch full deal details from Pipedrive | If stage_id is 2 (Meeting Booking) | Get meeting link URL, Start time and end time | ### Extract Deal Info<br>**Why it exists**<br>Pulls complete and reliable data directly from Pipedrive.<br><br>**You can change**<br>- Fields extracted from the deal<br>- Add custom fields<br>- Include organization or owner data |
| Get meeting link URL, Start time and end time | n8n-nodes-base.set | Normalize fields; compute StartTime/EndTime | Extract deal info | Create a meeting and add you recipient | ### Set / Transform Data<br>**Why it exists**<br>Cleans and prepares data so downstream tools receive correct formats.<br><br>**You can change**<br>- Time zone<br>- Date and time calculations<br>- Field names for CRM or calendar compatibility |
| Create a meeting and add you recipient | n8n-nodes-base.googleCalendar | Create Google Calendar event with attendee | Get meeting link URL, Start time and end time | Send Email reminder to the client | ### Create Calendar Event<br>**Why it exists**<br>Creates a single source of truth for the meeting and avoids double booking.<br><br>**You can change**<br>- Calendar selection<br>- Meeting title and description<br>- Replace Google Calendar with Outlook |
| Send Email reminder to the client | n8n-nodes-base.gmail | Email client confirmation/reminder | Create a meeting and add you recipient | Send a reminder to SDR in specific channel | ### Send Email Reminder<br>**Why it exists**<br>Confirms the meeting with the client and reduces no-shows.<br><br>**You can change**<br>- Email wording<br>- Sender account<br>- Add attachments or CC internal users |
| Send a reminder to SDR in specific channel | n8n-nodes-base.slack | Notify team in Slack channel | Send Email reminder to the client | — | ### Send Slack Notification<br>**Why it exists**<br>Keeps the sales team informed without checking the CRM.<br><br>**You can change**<br>- Target channel<br>- Message format<br>- Replace Slack with Microsoft Teams or Discord |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ## 📌 Template Overview (Read Before Using)<br><br>### What is it<br>###### This template is a **Revenue Ops meeting automation** that connects Pipedrive, Google Calendar, Gmail, and Slack.<br><br>###### When a deal reaches the *Meeting Booking* stage in Pipedrive, the workflow automatically schedules the meeting, confirms it with the client, and notifies the internal team.<br><br>###### It is designed to eliminate manual coordination and reduce missed or forgotten meetings.<br><br>---<br><br>### How it works<br>1. The workflow listens for **deal stage changes** in Pipedrive.<br>2. It runs only when the deal enters the *Meeting Booking* stage.<br>3. A short wait is applied to allow the SDR to add meeting details.<br>4. Deal and activity data are extracted and time values are normalized.<br>5. A meeting is created in Google Calendar with the client as an attendee.<br>6. A confirmation email is sent to the client via Gmail.<br>7. A reminder is posted in a Slack channel for the sales team.<br><br>---<br><br>### Setup steps<br>1. Connect your **Pipedrive account** and confirm the correct stage ID for *Meeting Booking*.<br>2. Ensure meeting details (date, time, duration, meeting link) are saved in the deal activity.<br>3. Connect **Google Calendar** and select the calendar where meetings should be created.<br>4. Connect **Gmail** and review the email message content.<br>5. Connect **Slack** and choose the channel for internal reminders.<br>6. Run the workflow once with test data to confirm timings and notifications are correct.<br>7. Activate the workflow once validation is complete. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — |  |

> Note: Sticky note nodes are included as nodes (as required). Their content is primarily documentation and doesn’t affect execution.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (example): **Pipedrive ops Automation**
   - Keep it **inactive** until credentials and tests are done.

2. **Add trigger: Pipedrive Trigger**
   - Node type: **Pipedrive Trigger**
   - Event:
     - **Entity:** Deal
     - **Action:** Change
   - Credentials:
     - Create/connect **Pipedrive API** credential (API token or OAuth depending on your n8n setup).
   - Save so n8n can register the webhook in Pipedrive.

3. **Add a Wait node**
   - Node type: **Wait**
   - Configure a delay of **55** (verify unit in your n8n UI; set to the desired seconds/minutes).
   - Connect: **Pipedrive Trigger → Wait**

4. **Add an IF node for stage filtering**
   - Node type: **IF**
   - Condition:
     - Value 1 (Expression): `{{$json.data.stage_id}}`
     - Operation: **equals**
     - Value 2: `2`
   - Connect: **Wait → IF**
   - Ensure you connect only the **true** output forward (optional: connect false output to a “NoOp”/log node).

5. **Add Pipedrive “Get Deal” node**
   - Node type: **Pipedrive**
   - Operation: **Get**
   - Deal ID (Expression): `{{$json.data.id}}`
   - Credentials: same Pipedrive credential
   - Connect: **IF (true) → Pipedrive (Get Deal)**

6. **Add a Set node to normalize fields**
   - Node type: **Set**
   - Add fields (as expressions), including:
     - `person_id.name` = `{{$json.person_id.name}}`
     - `person_id.email[0].value` = `{{$json.person_id.email[0].value}}`
     - `next_activity.conference_meeting_url` = `{{$json.next_activity.conference_meeting_url}}`
     - `next_activity.due_date` = `{{$json.next_activity.due_date}}`
     - `StartTime` = compute ISO start in `Asia/Dubai` (use Luxon `DateTime`)
     - `EndTime` = start + duration (HH:mm) in ISO
     - `org_id.name` = `{{$json.org_id.name}}`
   - Connect: **Pipedrive (Get Deal) → Set**

7. **Add Google Calendar node (Create Event)**
   - Node type: **Google Calendar**
   - Operation: **Create Event** (or equivalent in your version)
   - Calendar: select target calendar (e.g., `user@example.com`)
   - Start (Expression): `{{$json.StartTime}}`
   - End (Expression): `{{$json.EndTime}}`
   - Additional fields:
     - Summary: e.g., “Test Google calnder” (replace with real meeting title)
     - Location: `{{$json.next_activity.conference_meeting_url}}`
     - Attendees: `{{$json.person_id.email[0].value}}`
   - Credentials:
     - Connect **Google Calendar OAuth2**
     - Ensure scopes allow creating events.
   - Connect: **Set → Google Calendar**

8. **Add Gmail node (Send Email)**
   - Node type: **Gmail**
   - Operation: **Send**
   - To (Expression): use Set node output  
     `{{$('Get meeting link URL, Start time and end time').item.json.person_id.email[0].value}}`
   - Subject (Expression):  
     `Meeting Reminder:{{$('Get meeting link URL, Start time and end time').item.json.next_activity.due_date}}`
   - Message: include meeting time from calendar output, e.g.:  
     `{{ DateTime.fromISO($json.start.dateTime).toFormat('dd LLL yyyy, hh:mm a') }}`
   - Credentials:
     - Connect **Gmail OAuth2** (ensure send scope).
   - Connect: **Google Calendar → Gmail**

9. **Add Slack node (Post Message)**
   - Node type: **Slack**
   - Operation: **Post message**
   - Select: **Channel**
   - Channel: pick your channel (example ID `C0AAP1C16S2`)
   - Text: include meeting time and link, e.g.:
     - Meeting time: `{{ DateTime.fromISO($json.start.dateTime).toFormat('dd LLL yyyy, hh:mm a') }}`
     - Link: `{{ $('Create a meeting and add you recipient').item.json.location }}`
   - Credentials:
     - Connect **Slack OAuth2**
     - Ensure scopes like `chat:write` and that the bot/user is in the channel.
   - Connect: **Gmail → Slack**

10. **(Optional) Add sticky notes**
   - Node type: **Sticky Note**
   - Paste the provided content for maintainability (overview, node explanations, video link).

11. **Test with real data**
   - Move a test deal into the “Meeting Booking” stage (confirm stage_id = 2).
   - Ensure the deal has a **next activity** containing:
     - due date, due time, duration, conference meeting URL, and contact email.
   - Verify:
     - Event created correctly (timezone and duration)
     - Email delivered and formatted
     - Slack message posts to the intended channel

12. **Activate the workflow**
   - Once validated, set workflow to **Active**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video link: https://youtu.be/Y1Odi7odMQQ | Referenced in a canvas note (“Video Tutorial”) |
| Embedded reference: `@[youtube](Y1Odi7odMQQ)` | Present in sticky note content (YouTube embed syntax used by some n8n templates) |
| Timezone used in date calculations: `Asia/Dubai` | Defined in Set node expressions; adjust to your operating timezone |
| Stage gating uses `stage_id = 2` | Must match your Pipedrive pipeline’s “Meeting Booking” stage ID |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.