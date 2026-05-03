Send AI pet grooming reminders and follow-ups with GPT-4o mini, Twilio, email, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/send-ai-pet-grooming-reminders-and-follow-ups-with-gpt-4o-mini--twilio--email--google-sheets-and-slack-12746


# Send AI pet grooming reminders and follow-ups with GPT-4o mini, Twilio, email, Google Sheets and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
Automate pet grooming appointment reminders and follow-ups using Google Sheets as the appointment source, GPT‑4o mini to generate a personalized SMS, Twilio for SMS delivery, email for prep instructions and follow-ups, and Slack for internal notifications and weekly reporting.

**Primary use cases:**
- Daily reminder outreach for tomorrow’s appointments
- Prep instruction emails before appointments
- Automated follow-up or reschedule email depending on confirmation status
- Internal daily schedule notification to groomers via Slack
- Logging interactions into Google Sheets and producing a weekly-style summary message in Slack (note: currently triggered per run, not weekly)

### Logical Blocks
**1.1 Trigger & Runtime Configuration**
- Runs every day at a fixed time and sets reusable parameters (days ahead, reschedule URL, grooming instructions).

**1.2 Appointment Retrieval & Normalization**
- Pulls appointments for a target date from Google Sheets and normalizes column names to consistent internal fields.

**1.3 AI Reminder Generation & Client Communications**
- Uses OpenAI (GPT‑4o mini) to generate an SMS reminder, sends via Twilio, then sends a prep email.

**1.4 Confirmation Branching & Follow-up**
- If confirmed: wait 1 day then send a thank-you email.
- If not confirmed: send reschedule link email immediately.

**1.5 Logging, Team Notification, and Summary Report**
- Logs interactions to a Google Sheet.
- Notifies groomers on Slack with the day’s appointment details.
- Merges “log” and “notification” paths and posts a summary to Slack (management channel).

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Runtime Configuration

**Overview:**  
Starts the workflow daily and defines configuration values used across downstream nodes (date window, reschedule URL, grooming instructions).

**Nodes Involved:**
- Daily Appointment Check
- Workflow Configuration

#### Node: Daily Appointment Check
- **Type / Role:** Schedule Trigger — initiates workflow on a schedule.
- **Configuration:** Triggers daily at **08:00** (server / instance timezone).
- **Inputs / Outputs:** Entry node → outputs to **Workflow Configuration**.
- **Version notes:** `typeVersion 1.3` (standard schedule trigger behavior).
- **Edge cases / failures:**
  - Timezone mismatch between business locale and n8n instance may cause “wrong day” lookups.
  - If n8n is down at trigger time, execution may be missed depending on hosting/restart behavior.

#### Node: Workflow Configuration
- **Type / Role:** Set node — defines variables/constants for the run.
- **Configuration choices:**
  - `daysAhead = 1` (targets tomorrow).
  - `rescheduleUrl = <placeholder>` (used in reschedule email).
  - `groomingInstructions` static text (used in prep email).
  - **Include other fields:** enabled (keeps upstream fields if any; here it mostly just outputs these assignments).
- **Key expressions/variables:** None beyond literal values.
- **Outputs:** → **Get Upcoming Appointments**
- **Edge cases / failures:**
  - Placeholder values must be replaced (reschedule URL), otherwise emails will contain placeholders.

Sticky note context (applies to this block): **“## Trigger & Config”** and the “Main / Setup” note describing general setup and contact info.

---

### 2.2 Appointment Retrieval & Normalization

**Overview:**  
Fetches appointments for a computed date from Google Sheets and standardizes the data fields to a consistent schema used by subsequent nodes.

**Nodes Involved:**
- Get Upcoming Appointments
- Normalize Client Data

#### Node: Get Upcoming Appointments
- **Type / Role:** Google Sheets — reads/filter rows for the target date.
- **Configuration choices (interpreted):**
  - Uses **Google Sheets OAuth2** credentials (“Google Sheets account 3”).
  - Targets a specific **Document ID** (placeholder) and **Sheet name** (placeholder).
  - Reads from range `A:F`.
  - Applies a filter: `date` column must equal **tomorrow’s date**.
- **Key expression:**
  - Lookup value:  
    `{{ $today.plus({ days: $('Workflow Configuration').first().json.daysAhead }).toFormat('yyyy-MM-dd') }}`
- **Inputs / Outputs:**
  - Input from **Workflow Configuration**.
  - Output rows → **Normalize Client Data** (each row becomes an item).
- **Version notes:** `typeVersion 4.7` (recent Sheets node with filter UI and range options).
- **Edge cases / failures:**
  - **Auth/permission errors** if the sheet is not shared with the OAuth account.
  - **Column mismatch:** filter expects a column literally named `date`. If the sheet uses `Date` or `Appointment Date`, filter may return nothing.
  - **Date format mismatch:** the sheet must store date strings matching `yyyy-MM-dd` or comparable normalized value; otherwise no matches.
  - If no appointments match, downstream nodes may not run (no items), which also affects Slack summaries/logging.

#### Node: Normalize Client Data
- **Type / Role:** Set node — normalizes possible alternative column names into consistent keys.
- **Configuration choices:**
  - Creates/overwrites fields:
    - `clientName = $json.clientName || $json['Client Name']`
    - `phone = $json.phone || $json['Phone']`
    - `email = $json.email || $json['Email']`
    - `petType = $json.petType || $json['Pet Type']`
    - `appointmentDate = $json.appointmentDate || $json['Appointment Date']`
    - `appointmentTime = $json.appointmentTime || $json['Appointment Time']`
    - `confirmed = $json.confirmed || $json['Confirmed'] || 'No'`
  - **Include other fields:** enabled.
- **Inputs / Outputs:**
  - Input from **Get Upcoming Appointments**.
  - Two parallel outputs:
    - → **Generate Personalized Reminder**
    - → **Notify Groomers of Daily Schedule**
- **Edge cases / failures:**
  - Missing/blank phone/email will later cause Twilio/email send failures.
  - `confirmed` defaults to `'No'`, which routes to reschedule email unless explicitly `Yes`.

Sticky note context: **“## Sheets & AI logic”** covers Sheets + AI generation area.

---

### 2.3 AI Reminder Generation & Client Communications

**Overview:**  
Generates a concise personalized reminder SMS via OpenAI, sends it through Twilio, and emails grooming preparation instructions to the client.

**Nodes Involved:**
- Generate Personalized Reminder
- Send SMS Reminder
- Send Grooming Prep Instructions

#### Node: Generate Personalized Reminder
- **Type / Role:** OpenAI (LangChain) node — generates SMS text.
- **Configuration choices:**
  - Model: **gpt‑4o-mini** (selected by id).
  - Prompt instructs: friendly, warm, professional, under 160 characters, includes client name, pet type, appointment date/time.
  - Uses built-in response field(s); output is expected as `message` downstream.
- **Key expression:** Prompt uses templating with:
  - `{{ $json.clientName }}`, `{{ $json.petType }}`, `{{ $json.appointmentDate }}`, `{{ $json.appointmentTime }}`
- **Inputs / Outputs:**
  - Input from **Normalize Client Data**.
  - Output → **Send SMS Reminder**
- **Version notes:** `typeVersion 2.1` of the OpenAI LangChain node.
- **Edge cases / failures:**
  - OpenAI credential/quota issues (401/429), model not available, or network timeouts.
  - Output field naming: Twilio node references `{{$json.message}}`. If this OpenAI node outputs content under a different key (e.g., `text`, `content`, or nested structure), SMS will be empty or expression will fail. Validate the node’s actual output structure in your n8n version.

#### Node: Send SMS Reminder
- **Type / Role:** Twilio — sends SMS to client.
- **Configuration choices:**
  - `to = {{$json.phone}}`
  - `from = <placeholder Twilio number>`
  - `message = {{$json.message}}`
- **Inputs / Outputs:** Input from **Generate Personalized Reminder** → output to **Send Grooming Prep Instructions**
- **Edge cases / failures:**
  - Invalid phone formatting (E.164 recommended, e.g., `+15551234567`).
  - Twilio “from” number not SMS-enabled or not in correct region.
  - If `message` is missing (see note above), Twilio may reject or send blank content.

#### Node: Send Grooming Prep Instructions
- **Type / Role:** Email Send — sends prep instructions to the client.
- **Configuration choices:**
  - To: `{{$json.email}}`
  - From: placeholder business email
  - Subject: `Grooming Prep Instructions for {{ $json.petType }} - Tomorrow!`
  - Text email body includes grooming instructions from configuration:
    - `{{ $('Workflow Configuration').first().json.groomingInstructions }}`
  - Format: plain text
- **Inputs / Outputs:** Input from **Send SMS Reminder** → output to **Check Confirmation Status**
- **Edge cases / failures:**
  - Missing/invalid email address.
  - Email node requires SMTP or configured email credentials in n8n environment; misconfiguration leads to send failures.
  - Content refers to “tomorrow” regardless of actual schedule logic—ensure `daysAhead` remains 1 or adjust text.

Sticky note context: **“## Communication”** covers Twilio + email nodes + IF/Wait.

---

### 2.4 Confirmation Branching & Follow-up

**Overview:**  
Routes clients based on confirmation status. Confirmed clients get a thank-you email after 1 day; unconfirmed clients receive a reschedule link email immediately.

**Nodes Involved:**
- Check Confirmation Status
- Wait for Follow-up Time
- Send Thank You Message
- Send Reschedule Link

#### Node: Check Confirmation Status
- **Type / Role:** IF node — conditional branch on `confirmed`.
- **Configuration choices:**
  - Condition: `{{$json.confirmed}}` **equals** `Yes` (case-insensitive enabled via options).
- **Inputs / Outputs:**
  - Input from **Send Grooming Prep Instructions**
  - **True** output (confirmed) → **Wait for Follow-up Time**
  - **False** output (not confirmed) → **Send Reschedule Link**
- **Edge cases / failures:**
  - If sheet uses values like `Y`, `TRUE`, `Confirmed`, etc., condition won’t match; normalize upstream or broaden condition (e.g., “contains”).
  - If `confirmed` is empty, it defaults to `'No'` and goes to reschedule path.

#### Node: Wait for Follow-up Time
- **Type / Role:** Wait node — delays execution for follow-up timing.
- **Configuration choices:** Wait **1 day**.
- **Inputs / Outputs:** From IF “true” branch → **Send Thank You Message**
- **Edge cases / failures:**
  - Wait node resumes later; if workflow is deactivated or execution data is pruned, resumed execution may not behave as expected depending on n8n settings.
  - “1 day” is relative to when the workflow ran, not appointment time; confirmed follow-up may not align with “after appointment” if appointment is more than 1 day away (if `daysAhead` changes).

#### Node: Send Thank You Message
- **Type / Role:** Email Send — sends post-visit thanks.
- **Configuration choices:** Plain text email to `{{$json.email}}` from placeholder business email; static subject “Thank You for Visiting Us!”
- **Inputs / Outputs:** From **Wait for Follow-up Time** → **Log Client Interactions**
- **Edge cases / failures:** Same email credential/address issues as above; may send even if appointment did not happen (because confirmation ≠ completion).

#### Node: Send Reschedule Link
- **Type / Role:** Email Send — sends rescheduling instructions.
- **Configuration choices:**
  - Includes `{{ $('Workflow Configuration').first().json.rescheduleUrl }}`
  - Plain text email to `{{$json.email}}`
- **Inputs / Outputs:** From IF “false” branch → **Log Client Interactions**
- **Edge cases / failures:**
  - If `rescheduleUrl` is not replaced, client receives placeholder text.
  - Triggered immediately upon “not confirmed” rather than after a missed appointment; adjust logic if you want “missed appointment” detection.

---

### 2.5 Logging, Team Notification, and Summary Report

**Overview:**  
Logs each follow-up/reschedule interaction to Google Sheets, sends Slack notifications to groomers for each appointment item, merges both paths, and sends a Slack “weekly summary” message based on merged item counts.

**Nodes Involved:**
- Log Client Interactions
- Notify Groomers of Daily Schedule
- Compile Weekly Summary Data
- Format Weekly Report
- Send Weekly Summary Report

#### Node: Log Client Interactions
- **Type / Role:** Google Sheets — appends or updates a log record.
- **Configuration choices:**
  - Operation: **appendOrUpdate**
  - Document ID and sheet name: placeholders (separate logging sheet).
  - Columns mapped:
    - email, phone, petType, timestamp (`{{$now.toISO()}}`), clientName, appointmentDate, interactionType = “Follow-up Sent”
  - Mapping mode: auto-map input data (but explicitly supplies `columns.value` object).
- **Inputs / Outputs:**
  - Input from **Send Thank You Message** and **Send Reschedule Link**
  - Output → **Compile Weekly Summary Data** (input index 0)
- **Edge cases / failures:**
  - appendOrUpdate typically requires a key/lookup column to determine updates; if not configured, it may behave like append or may error depending on node settings/version.
  - Sheet column headers must match expected names (`email`, `phone`, etc.) or auto-mapping may misbehave.
  - If multiple items run, this logs per-item, increasing counts.

#### Node: Notify Groomers of Daily Schedule
- **Type / Role:** Slack — posts daily schedule details per appointment.
- **Configuration choices:**
  - Sends a formatted message including appointment date/time, client, pet type, phone, and confirmation status with conditional expression:
    - `{{ $json.confirmed === "Yes" ? "✅ Confirmed" : "⏳ Pending" }}`
  - Channel: placeholder (by name)
- **Inputs / Outputs:**
  - Input from **Normalize Client Data**
  - Output → **Compile Weekly Summary Data** (input index 1)
- **Edge cases / failures:**
  - Slack credentials/webhook/token misconfiguration or missing permissions to post in channel.
  - Message uses emoji characters; if strict “no emoji” business requirement exists, edit text (this is Slack-only).
  - Posts one message per appointment item; could be noisy for many appointments.

#### Node: Compile Weekly Summary Data
- **Type / Role:** Merge — combines two streams (logging + groomer notifications).
- **Configuration choices:**
  - Mode: **combine**
  - Combine by: **position** (combineByPosition)
  - This means item #1 from log stream merges with item #1 from Slack-notify stream, etc.
- **Inputs / Outputs:**
  - Input 0 from **Log Client Interactions**
  - Input 1 from **Notify Groomers of Daily Schedule**
  - Output → **Format Weekly Report**
- **Edge cases / failures:**
  - If one branch produces fewer items (e.g., logging only happens after email branch, but Slack notify happens immediately), combine-by-position can misalign or drop data depending on arrival and counts.
  - Because logging happens only after follow-up/reschedule, while Slack notify happens for every appointment, the merge may not represent “weekly summary” reliably.

#### Node: Format Weekly Report
- **Type / Role:** Set — builds the report fields.
- **Configuration choices:**
  - `reportTitle = "Weekly Client Engagement Summary"`
  - `reportDate = {{$now.toFormat('yyyy-MM-dd')}}`
  - `totalInteractions = {{$input.all().length}}`
  - `summary = "This week we had X client interactions..."`
  - Include other fields enabled.
- **Inputs / Outputs:** From **Compile Weekly Summary Data** → **Send Weekly Summary Report**
- **Edge cases / failures:**
  - `totalInteractions` here equals the number of items at this node during the current execution, not a true weekly aggregate.
  - If there are no items, node may not execute (depending on upstream output), resulting in no report.

#### Node: Send Weekly Summary Report
- **Type / Role:** Slack — posts the formatted report.
- **Configuration choices:**
  - Channel: placeholder (by id)
  - Text uses report fields (title, date, summary, total interactions).
- **Inputs / Outputs:** From **Format Weekly Report** → (end)
- **Edge cases / failures:**
  - Same Slack auth/permissions issues.
  - Despite name “Weekly”, it will run whenever upstream runs and produces items (currently daily schedule run).

Sticky note context: **“## Log & Notify”** covers the logging + Slack reporting area.  
General sticky note “Main / Setup” applies broadly.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Appointment Check | Schedule Trigger | Daily workflow entry point (08:00) | — | Workflow Configuration | ## Trigger & Config |
| Workflow Configuration | Set | Defines constants (daysAhead, URLs, instructions) | Daily Appointment Check | Get Upcoming Appointments | ## Trigger & Config |
| Get Upcoming Appointments | Google Sheets | Fetch appointments for target date | Workflow Configuration | Normalize Client Data | ## Sheets & AI logic |
| Normalize Client Data | Set | Normalize sheet columns to standard fields | Get Upcoming Appointments | Generate Personalized Reminder; Notify Groomers of Daily Schedule | ## Sheets & AI logic |
| Generate Personalized Reminder | OpenAI (LangChain) | Generate personalized SMS reminder text | Normalize Client Data | Send SMS Reminder | ## Sheets & AI logic |
| Send SMS Reminder | Twilio | Send reminder SMS | Generate Personalized Reminder | Send Grooming Prep Instructions | ## Communication |
| Send Grooming Prep Instructions | Email Send | Send prep instructions email | Send SMS Reminder | Check Confirmation Status | ## Communication |
| Check Confirmation Status | IF | Branch based on confirmed == “Yes” | Send Grooming Prep Instructions | Wait for Follow-up Time (true); Send Reschedule Link (false) | ## Communication |
| Wait for Follow-up Time | Wait | Delay 1 day before thank-you | Check Confirmation Status (true) | Send Thank You Message | ## Communication |
| Send Thank You Message | Email Send | Send thank-you email | Wait for Follow-up Time | Log Client Interactions | ## Communication |
| Send Reschedule Link | Email Send | Send reschedule email with URL | Check Confirmation Status (false) | Log Client Interactions | ## Communication |
| Log Client Interactions | Google Sheets | Append/update interaction log row | Send Thank You Message; Send Reschedule Link | Compile Weekly Summary Data | ## Log & Notify |
| Notify Groomers of Daily Schedule | Slack | Post appointment details to groomer channel | Normalize Client Data | Compile Weekly Summary Data | ## Log & Notify |
| Compile Weekly Summary Data | Merge | Combine log + notify streams | Log Client Interactions; Notify Groomers of Daily Schedule | Format Weekly Report | ## Log & Notify |
| Format Weekly Report | Set | Create report fields based on item count | Compile Weekly Summary Data | Send Weekly Summary Report | ## Log & Notify |
| Send Weekly Summary Report | Slack | Post report to management channel | Format Weekly Report | — | ## Log & Notify |
| Sticky Note | Sticky Note | Comment: “## Trigger & Config” | — | — |  |
| Sticky Note1 | Sticky Note | Comment: “## Sheets & AI logic” | — | — |  |
| Sticky Note2 | Sticky Note | Comment: “## Communication” | — | — |  |
| Sticky Note3 | Sticky Note | Comment: “## Log & Notify” | — | — |  |
| Sticky Note4 | Sticky Note | Comment: Main description + Setup steps + contact: hyrum@quartersmart.com | — | — |  |

Additional sticky note content (applies broadly):  
- **“## Main …”** Automates reminders, engagement, team notifications; setup steps; contact Hyrum Hurst at **hyrum@quartersmart.com**.

---

## 4. Reproducing the Workflow from Scratch

1) **Create Schedule Trigger**
   - Add node: **Schedule Trigger**
   - Set to run **every day at 08:00**.
   - Connect to next node.

2) **Add configuration Set node**
   - Add node: **Set** named “Workflow Configuration”.
   - Add fields:
     - `daysAhead` (Number) = `1`
     - `rescheduleUrl` (String) = your online booking/reschedule link
     - `groomingInstructions` (String) = your standardized prep instructions
   - Enable **Include Other Fields**.
   - Connect from trigger → this node.

3) **Read upcoming appointments from Google Sheets**
   - Add node: **Google Sheets** named “Get Upcoming Appointments”.
   - Credentials: configure **Google Sheets OAuth2** (grant access to the appointment spreadsheet).
   - Document: select the spreadsheet containing appointments.
   - Sheet name: select the appointments sheet (e.g., “Appointments”).
   - Range: `A:F` (or adjust to include your used columns).
   - Add a filter:
     - Lookup Column: `date` (or adjust to match your header)
     - Lookup Value expression:  
       `{{ $today.plus({ days: $('Workflow Configuration').first().json.daysAhead }).toFormat('yyyy-MM-dd') }}`
   - Connect from “Workflow Configuration” → “Get Upcoming Appointments”.

4) **Normalize appointment/client fields**
   - Add node: **Set** named “Normalize Client Data”.
   - Enable **Include Other Fields**.
   - Add string fields with expressions:
     - `clientName = {{ $json.clientName || $json['Client Name'] }}`
     - `phone = {{ $json.phone || $json['Phone'] }}`
     - `email = {{ $json.email || $json['Email'] }}`
     - `petType = {{ $json.petType || $json['Pet Type'] }}`
     - `appointmentDate = {{ $json.appointmentDate || $json['Appointment Date'] }}`
     - `appointmentTime = {{ $json.appointmentTime || $json['Appointment Time'] }}`
     - `confirmed = {{ $json.confirmed || $json['Confirmed'] || 'No' }}`
   - Connect “Get Upcoming Appointments” → “Normalize Client Data”.

5) **Generate SMS reminder with OpenAI**
   - Add node: **OpenAI (LangChain)** named “Generate Personalized Reminder”.
   - Credentials: configure **OpenAI API** (or use provided credits if available).
   - Model: **gpt-4o-mini**.
   - Prompt/content:  
     “Generate a friendly, personalized appointment reminder SMS for {{ clientName }} … under 160 characters…”
   - Ensure the node outputs a field you can reference as `message` (you may need a small Set node if output key differs).
   - Connect “Normalize Client Data” → “Generate Personalized Reminder”.

6) **Send SMS via Twilio**
   - Add node: **Twilio** named “Send SMS Reminder”.
   - Credentials: configure Twilio account (Account SID/Auth Token).
   - From: your Twilio SMS-capable number.
   - To: `{{ $json.phone }}`
   - Message: `{{ $json.message }}` (adjust if OpenAI output key differs).
   - Connect “Generate Personalized Reminder” → “Send SMS Reminder”.

7) **Send prep instructions email**
   - Add node: **Email Send** named “Send Grooming Prep Instructions”.
   - Configure email sending (SMTP or n8n email credentials depending on your setup).
   - From: your business email.
   - To: `{{ $json.email }}`
   - Subject: `Grooming Prep Instructions for {{ $json.petType }} - Tomorrow!`
   - Body (text): include `{{ $('Workflow Configuration').first().json.groomingInstructions }}`
   - Connect “Send SMS Reminder” → “Send Grooming Prep Instructions”.

8) **Branch on confirmation**
   - Add node: **IF** named “Check Confirmation Status”.
   - Condition: String → `{{ $json.confirmed }}` equals `Yes` (case-insensitive).
   - Connect “Send Grooming Prep Instructions” → “Check Confirmation Status”.

9) **Confirmed path: wait then thank you email**
   - Add node: **Wait** named “Wait for Follow-up Time”.
     - Unit: days; Amount: 1
   - Add node: **Email Send** named “Send Thank You Message”.
     - To: `{{ $json.email }}`
     - From: your business email
     - Subject: “Thank You for Visiting Us!”
     - Text: thank-you message
   - Connect IF **true** → Wait → Thank You Email.

10) **Not confirmed path: reschedule email**
   - Add node: **Email Send** named “Send Reschedule Link”.
     - To: `{{ $json.email }}`
     - From: your business email
     - Subject: “We Missed You - Reschedule Your Grooming Appointment”
     - Text includes `{{ $('Workflow Configuration').first().json.rescheduleUrl }}`
   - Connect IF **false** → “Send Reschedule Link”.

11) **Log interactions to Google Sheets**
   - Add node: **Google Sheets** named “Log Client Interactions”.
   - Credentials: Google Sheets OAuth2 (logging spreadsheet).
   - Operation: **appendOrUpdate** (or use Append if you don’t need updates).
   - Map columns (ensure headers exist in the log sheet):
     - clientName, email, phone, petType, appointmentDate
     - timestamp = `{{ $now.toISO() }}`
     - interactionType = “Follow-up Sent”
   - Connect:
     - “Send Thank You Message” → “Log Client Interactions”
     - “Send Reschedule Link” → “Log Client Interactions”

12) **Notify groomers in Slack**
   - Add node: **Slack** named “Notify Groomers of Daily Schedule”.
   - Credentials: Slack OAuth2/bot token with permission to post messages.
   - Channel: your groomers channel (e.g., `#groomers`).
   - Message template includes date/time/client/pet/phone and confirmation status expression.
   - Connect “Normalize Client Data” → “Notify Groomers of Daily Schedule”.

13) **Merge data for reporting**
   - Add node: **Merge** named “Compile Weekly Summary Data”.
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - “Log Client Interactions” → Merge **Input 1**
     - “Notify Groomers of Daily Schedule” → Merge **Input 2**

14) **Format and send Slack report**
   - Add node: **Set** named “Format Weekly Report”:
     - reportTitle, reportDate, totalInteractions = `{{ $input.all().length }}`, summary string.
   - Add node: **Slack** named “Send Weekly Summary Report” to management channel.
   - Connect Merge → Format → Slack report.

15) **Replace all placeholders**
   - Google Sheets Document IDs and sheet names
   - Twilio from number
   - Email from address + SMTP/credential configuration
   - Slack channel IDs/names

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Main: Automates pet grooming appointment reminders, personalized engagement, and team notifications. Clients are reminded, groomers are updated, and follow-ups are automated.” | Sticky note (general workflow purpose) |
| “## Setup: 1. Connect Google Sheets… 2. Add SMS and Email… 3. Connect Slack… 4. Add OpenAI API key… 5. Customize messages…” | Sticky note (implementation guidance) |
| “For assistance, contact Hyrum Hurst at hyrum@quartersmart.com” | Sticky note (support contact) |