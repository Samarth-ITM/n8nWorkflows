Nurture landscaping leads and book calls with GoHighLevel, OpenAI and Slack

https://n8nworkflows.xyz/workflows/nurture-landscaping-leads-and-book-calls-with-gohighlevel--openai-and-slack-12745


# Nurture landscaping leads and book calls with GoHighLevel, OpenAI and Slack

## 1. Workflow Overview

**Purpose:** Automate immediate follow-up and internal notification for new landscaping leads captured in GoHighLevel (GHL). The workflow generates an AI-personalized message, sends it via SMS and email through GHL, checks email engagement, either schedules a follow-up call (Google Calendar) or sends a reminder SMS, logs outcomes to Google Sheets, and posts notifications to Slack. A second entry point posts a weekly summary report to Slack from the Google Sheet.

**Primary use cases:**
- Speed-to-lead response automation (SMS + email).
- Light lead nurturing based on whether an email was opened.
- Operational visibility (Slack notifications + Google Sheets logging).
- Weekly performance reporting to Slack.

### 1.1 Trigger & Configuration
Receives a new lead payload (webhook), then sets reusable configuration values and normalizes lead fields.

### 1.2 AI Message Generation & GHL Outreach
Uses OpenAI to generate a personalized follow-up message, then sends SMS and email via GHL’s API.

### 1.3 Engagement Check & Conditional Follow-up
Fetches message status from GHL, decides if the email was opened, then either schedules a calendar follow-up or sends a reminder SMS.

### 1.4 Log & Notify
Merges branch results, writes/updates a row in Google Sheets, then notifies a Slack channel.

### 1.5 Summary Reporting (Weekly)
On a weekly schedule, reads the Google Sheet and posts aggregate metrics to Slack.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Trigger & Configuration
**Overview:** Accepts inbound lead events and prepares normalized lead fields plus shared configuration (API base URL, API key, Slack channel).  
**Nodes involved:**  
- GoHighLevel New Lead Trigger  
- Workflow Configuration  
- Normalize Lead Information  

#### Node: GoHighLevel New Lead Trigger
- **Type / role:** Webhook (entry point) for inbound lead events (from GHL or a middleware).
- **Key configuration:**
  - HTTP Method: `POST`
  - Path: `gohighlevel-lead-webhook`
- **Inputs/outputs:**
  - **Input:** External HTTP request.
  - **Output:** A single item containing request data (expected to include `body.contact.*`).
- **Failure/edge cases:**
  - Missing/invalid payload structure (e.g., `body.contact` absent) will break later expressions.
  - If used with GHL, ensure the sender posts JSON with the expected schema.

#### Node: Workflow Configuration
- **Type / role:** Set node to store configuration values for downstream expressions.
- **Configuration choices:**
  - Adds fields:
    - `ghlApiUrl` (placeholder)
    - `ghlApiKey` (placeholder)
    - `slackChannel` (placeholder)
  - **Include other fields:** enabled (passes through webhook payload).
- **Key expressions/variables:** none (static placeholders).
- **Inputs/outputs:**
  - Input: from webhook trigger.
  - Output: payload + config fields.
- **Failure/edge cases:**
  - Leaving placeholders unresolved will cause HTTP request failures (invalid URL/auth) and Slack posting errors.

#### Node: Normalize Lead Information
- **Type / role:** Set node to standardize lead fields used throughout the workflow.
- **Configuration choices / mapping:**
  - `leadName` = `body.contact.name` OR `firstName + lastName`
  - `leadPhone` = `body.contact.phone`
  - `leadEmail` = `body.contact.email`
  - `serviceType` = `body.contact.customFields?.serviceType` OR `'General Landscaping'`
  - `leadId` = `body.contact.id`
  - **Include other fields:** enabled
- **Key expressions:**
  - Uses optional chaining for `customFields?.serviceType`.
- **Failure/edge cases:**
  - If `firstName` or `lastName` is missing, name concatenation may produce `"undefined "` or `" undefined"`.
  - If `body.contact` is missing, most expressions will error.

---

### Block 1.2 — AI Message Generation & GHL Outreach
**Overview:** Creates a personalized message with OpenAI and sends it as both SMS and Email through GHL’s conversations/messages endpoint.  
**Nodes involved:**  
- Generate Personalized Follow-Up Message  
- Send SMS via GoHighLevel  
- Send Email via GoHighLevel  

#### Node: Generate Personalized Follow-Up Message
- **Type / role:** OpenAI (LangChain) node to generate message text.
- **Configuration choices:**
  - Prompt instructs a friendly/professional follow-up, referencing `leadName` and `serviceType`.
  - Uses built-in OpenAI credentials: “n8n free OpenAI API credits”.
- **Key expressions/variables:**
  - Injects:
    - `{{ $json.leadName }}`
    - `{{ $json.serviceType }}`
- **Inputs/outputs:**
  - Input: normalized lead item.
  - Output: AI response (workflow later references `...json.message`).
- **Failure/edge cases:**
  - Output field naming can vary by node version/config; this workflow assumes `$('Generate Personalized Follow-Up Message').first().json.message` exists. If the node returns a different structure (e.g., `text`), downstream nodes will send blank messages.
  - Rate limits / quota errors from OpenAI.
  - Prompt injection risk if upstream lead fields contain malicious text (usually low impact here but can degrade message quality).

#### Node: Send SMS via GoHighLevel
- **Type / role:** HTTP Request to GHL API to send an SMS message.
- **Configuration choices:**
  - POST `{{ghlApiUrl}}/conversations/messages`
  - JSON body:
    - `type: "SMS"`
    - `contactId: {{$json.leadId}}`
    - `message: {{ OpenAI message }}`
  - Headers:
    - `Authorization: Bearer {{ghlApiKey}}`
    - `Content-Type: application/json`
  - Authentication: generic credential type (but token is manually set via header).
- **Inputs/outputs:**
  - Input: from OpenAI node (still contains `leadId` due to pass-through).
  - Output: GHL API response (often includes message metadata).
- **Failure/edge cases:**
  - 401/403 if API key invalid or lacks scope.
  - 404/400 if `ghlApiUrl` wrong or payload missing required fields.
  - SMS sending may fail if phone is invalid/unsubscribed.
  - If OpenAI message contains quotes/newlines, JSON is still valid because it’s string-interpolated—however extremely long messages may be rejected by SMS constraints.

#### Node: Send Email via GoHighLevel
- **Type / role:** HTTP Request to GHL API to send an Email message.
- **Configuration choices:**
  - POST `{{ghlApiUrl}}/conversations/messages`
  - JSON body:
    - `type: "Email"`
    - `contactId`
    - `subject: "Welcome to Our Landscaping Services!"`
    - `html: {{ OpenAI message }}`
  - Same auth headers pattern as SMS node.
- **Inputs/outputs:**
  - Input: output of “Send SMS via GoHighLevel”
  - Output: GHL API response (should include identifiers to query later).
- **Failure/edge cases:**
  - Same auth/base URL issues as SMS.
  - Email sending may fail if contact email missing/invalid, or email channel not configured in GHL.
  - **Downstream dependency risk:** the next node expects a conversation/message ID; if the response doesn’t contain it (or it’s referenced incorrectly), the “Check Email Open Status” step will fail.

---

### Block 1.3 — Engagement Check & Conditional Follow-up
**Overview:** Checks whether the email was opened, then either schedules a follow-up call (hot lead) or sends a reminder SMS (cold/unopened).  
**Nodes involved:**  
- Check Email Open Status  
- Email Opened?  
- Schedule Follow-Up in Calendar  
- Send Reminder SMS  

#### Node: Check Email Open Status
- **Type / role:** HTTP Request to GHL API to retrieve messages from a conversation.
- **Configuration choices:**
  - GET `{{ghlApiUrl}}/conversations/{{ $json.json.conversationId }}/messages`
  - Timeout option: 30000 ms
  - Header: `Authorization: Bearer {{ghlApiKey}}`
- **Key expressions/variables:**
  - Uses `{{ $json.json.conversationId }}`.
- **Inputs/outputs:**
  - Input: output from “Send Email via GoHighLevel”.
  - Output: conversation messages payload from GHL.
- **Failure/edge cases (important):**
  - **Likely expression/path bug:** `conversationId` probably won’t be located at `$json.json.conversationId`. Many n8n HTTP nodes return the body directly at `$json`, not nested under `$json.json`. If so, the URL becomes invalid and returns 404.
  - If the email send response does not include a conversationId, you must fetch it differently (e.g., from contact/conversation lookup).

#### Node: Email Opened?
- **Type / role:** IF node to branch based on whether any message has status `opened`.
- **Configuration choices:**
  - Condition: `{{ $json.json.messages && $json.json.messages.some(m => m.status === 'opened') }} == true`
- **Inputs/outputs:**
  - Input: output from “Check Email Open Status”.
  - Output (true branch index 0): to “Schedule Follow-Up in Calendar”
  - Output (false branch index 1): to “Send Reminder SMS”
- **Failure/edge cases (important):**
  - **Likely payload path bug:** again references `$json.json.messages`. If messages are at `$json.messages`, the check will always be false (or error), incorrectly routing all leads.
  - The node does not explicitly set a field like `emailOpened`; downstream logging references `$('Email Opened?').item.json.emailOpened`, which is **not created** here. This can cause incorrect sheet logging unless n8n implicitly preserves a similarly named field (it does not by default).

#### Node: Schedule Follow-Up in Calendar
- **Type / role:** Google Calendar node to create an event.
- **Configuration choices:**
  - Start: `now + 2 days`
  - End: `now + 2 days + 1 hour`
  - Calendar ID: placeholder
  - Summary: `Follow-up call with {{leadName}}`
  - Description: includes service type and phone
- **Inputs/outputs:**
  - Input: true branch from IF node.
  - Output: created event data; passed to merge/logging.
- **Failure/edge cases:**
  - Calendar ID placeholder must be replaced.
  - OAuth credential scope/permissions issues.
  - Timezone considerations: `$now` uses n8n server timezone unless configured.

#### Node: Send Reminder SMS
- **Type / role:** HTTP Request to GHL API sending a reminder SMS if the email wasn’t opened.
- **Configuration choices:**
  - POST `{{ghlApiUrl}}/conversations/messages`
  - Body uses lead fields from **Normalize Lead Information** via `$('Normalize Lead Information').first()`.
  - Message: “Hi {Name}, we sent you an email…”
- **Inputs/outputs:**
  - Input: false branch from IF node.
  - Output: GHL send result; passed to merge/logging.
- **Failure/edge cases:**
  - Same GHL auth/URL issues as prior HTTP nodes.
  - If leadId is missing, GHL will reject the request.

---

### Block 1.4 — Log & Notify
**Overview:** Combines the branch results, writes a status row into Google Sheets (append or update by Lead ID), then posts a Slack alert to the configured channel.  
**Nodes involved:**  
- Compile Lead Activity Report  
- Log Lead Activity to Google Sheets  
- Notify Manager in Slack  

#### Node: Compile Lead Activity Report
- **Type / role:** Merge node to recombine the two conditional branches.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Inputs/outputs:**
  - Input 1: from “Schedule Follow-Up in Calendar”
  - Input 2: from “Send Reminder SMS”
  - Output: merged item(s) to Google Sheets logging.
- **Failure/edge cases:**
  - `combineAll` behavior can produce unexpected structures if each branch outputs different fields or different item counts (usually 1 item each here).
  - If one branch never runs, merge still works in practice only when it receives input; however ensure the merge mode fits your expectations for single-branch execution.

#### Node: Log Lead Activity to Google Sheets
- **Type / role:** Google Sheets node to append or update the lead record.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `Lead ID`
  - Sheet name: `Lead Activity Log`
  - Range: implicitly via mapping; schema defines columns A–H conceptually.
  - Columns mapped:
    - Lead ID, Lead Name, Phone, Email, Service Type
    - Email Opened (Yes/No)
    - Status (“Follow-up Scheduled” / “Reminder Sent”)
    - Timestamp = `now`
- **Key expressions/variables (important):**
  - Status and Email Opened use: `$('Email Opened?').item.json.emailOpened ? ... : ...`
- **Failure/edge cases (important):**
  - **Likely logic bug:** `Email Opened?` IF node does not set `emailOpened`, so these expressions may evaluate as false/undefined, causing:
    - `Email Opened` always “No”
    - `Status` always “Reminder Sent”
  - If you want to log the branch result reliably, use:
    - The IF node branch itself (e.g., create a Set node in each branch setting `emailOpened=true/false`)
    - Or evaluate the same expression again here against the fetched messages payload.
  - Google Sheets credential and document ID placeholders must be valid.
  - Column names must match exactly in the target sheet header row.

#### Node: Notify Manager in Slack
- **Type / role:** Slack node posting a formatted message to a channel.
- **Configuration choices:**
  - Channel ID: from `Workflow Configuration.slackChannel`
  - Text includes lead details and `Status: {{ $json.Status }}` (expects Google Sheets node output to contain `Status`)
- **Inputs/outputs:**
  - Input: from “Log Lead Activity to Google Sheets”
  - Output: Slack API response.
- **Failure/edge cases:**
  - If Sheets node output doesn’t expose `Status` as `$json.Status`, the Slack message will show blank/undefined.
  - Slack channel ID must be correct and the Slack credential must have access.

---

### Block 1.5 — Summary Reporting (Weekly)
**Overview:** Weekly scheduled job reads the “Lead Activity Log” sheet and posts aggregate counts to Slack.  
**Nodes involved:**  
- Weekly Summary Schedule  
- Fetch Weekly Lead Data  
- Send Weekly Summary to Slack  

#### Node: Weekly Summary Schedule
- **Type / role:** Schedule Trigger (second entry point).
- **Configuration choices:**
  - Weekly on day 1 at 09:00 (based on n8n instance timezone).
- **Inputs/outputs:**
  - Input: time-based trigger.
  - Output: a trigger item (no lead data).

#### Node: Fetch Weekly Lead Data
- **Type / role:** Google Sheets read to pull log data.
- **Configuration choices:**
  - Range: `A:H`
  - Sheet: `Lead Activity Log`
  - Document ID: placeholder
  - Return first match: false (returns many rows)
- **Inputs/outputs:**
  - Input: from schedule trigger.
  - Output: multiple items (typically one item per row).
- **Failure/edge cases:**
  - This node does not filter “last week” rows; it fetches the entire range A:H. The “weekly” report is therefore “all-time” unless the sheet itself only contains weekly data or you add filtering.
  - If the sheet has empty rows in A:H, output may include blank items.

#### Node: Send Weekly Summary to Slack
- **Type / role:** Slack message with aggregate metrics.
- **Configuration choices / expressions (important):**
  - Text uses array-style operations:
    - `{{ $json.length }}`
    - `{{ $json.filter(...) }}`
- **Inputs/outputs:**
  - Input: from Google Sheets read.
- **Failure/edge cases (important):**
  - **Likely data-shape bug:** Google Sheets typically outputs *one item per row*, so `$json` is an object per item, not an array—`$json.length` and `$json.filter` will not work as intended.
  - Correct approaches:
    - Use `{{$items()}}` to access all items, e.g. `{{$items().length}}`
    - Or insert an “Item Lists” / “Code” node to aggregate before posting to Slack.
  - Slack channel ID is a placeholder and must be replaced.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GoHighLevel New Lead Trigger | Webhook | Receives new lead payload | — | Workflow Configuration | ## Trigger & Config |
| Workflow Configuration | Set | Stores API base URL/key + Slack channel | GoHighLevel New Lead Trigger | Normalize Lead Information | ## Trigger & Config |
| Normalize Lead Information | Set | Normalizes lead fields (name, phone, email, service, ID) | Workflow Configuration | Generate Personalized Follow-Up Message | ## Trigger & Config |
| Generate Personalized Follow-Up Message | OpenAI (LangChain) | Creates personalized follow-up message | Normalize Lead Information | Send SMS via GoHighLevel | ## AI routing and GHL |
| Send SMS via GoHighLevel | HTTP Request | Sends SMS through GHL conversations API | Generate Personalized Follow-Up Message | Send Email via GoHighLevel | ## AI routing and GHL |
| Send Email via GoHighLevel | HTTP Request | Sends email through GHL conversations API | Send SMS via GoHighLevel | Check Email Open Status | ## Email & Schedule |
| Check Email Open Status | HTTP Request | Retrieves conversation messages to detect opens | Send Email via GoHighLevel | Email Opened? | ## Email & Schedule |
| Email Opened? | IF | Branches on “opened” status | Check Email Open Status | Schedule Follow-Up in Calendar; Send Reminder SMS | ## Email & Schedule |
| Schedule Follow-Up in Calendar | Google Calendar | Creates follow-up call event | Email Opened? (true) | Compile Lead Activity Report | ## Email & Schedule |
| Send Reminder SMS | HTTP Request | Sends reminder SMS if unopened | Email Opened? (false) | Compile Lead Activity Report | ## Email & Schedule |
| Compile Lead Activity Report | Merge | Recombines branches for unified logging | Schedule Follow-Up in Calendar; Send Reminder SMS | Log Lead Activity to Google Sheets | ## Notify & Log |
| Log Lead Activity to Google Sheets | Google Sheets | Append/update lead log row | Compile Lead Activity Report | Notify Manager in Slack | ## Notify & Log |
| Notify Manager in Slack | Slack | Posts new-lead alert to Slack | Log Lead Activity to Google Sheets | — | ## Notify & Log |
| Weekly Summary Schedule | Schedule Trigger | Weekly reporting trigger | — | Fetch Weekly Lead Data | ## Summary Reporting |
| Fetch Weekly Lead Data | Google Sheets | Reads lead activity log rows | Weekly Summary Schedule | Send Weekly Summary to Slack | ## Summary Reporting |
| Send Weekly Summary to Slack | Slack | Posts weekly metrics to Slack | Fetch Weekly Lead Data | — | ## Summary Reporting |
| Sticky Note | Sticky Note | Comment block label | — | — |  |
| Sticky Note1 | Sticky Note | Comment block label | — | — |  |
| Sticky Note2 | Sticky Note | Comment block label | — | — |  |
| Sticky Note3 | Sticky Note | Comment block label | — | — |  |
| Sticky Note4 | Sticky Note | Comment block label | — | — |  |
| Sticky Note5 | Sticky Note | Global workflow notes/setup contact | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook trigger**
   - Add **Webhook** node named **“GoHighLevel New Lead Trigger”**
   - Method: `POST`
   - Path: `gohighlevel-lead-webhook`
   - Activate test URL and configure GHL (or your lead source) to POST lead JSON to it.

2. **Add configuration Set node**
   - Add **Set** node named **“Workflow Configuration”**
   - Enable **Include Other Fields**
   - Create string fields:
     - `ghlApiUrl` (e.g., `https://services.leadconnectorhq.com` or your GHL API base)
     - `ghlApiKey` (private token)
     - `slackChannel` (Slack channel ID, e.g. `C123...`)
   - Connect: **Webhook → Workflow Configuration**

3. **Normalize lead fields**
   - Add **Set** node named **“Normalize Lead Information”**
   - Enable **Include Other Fields**
   - Add fields (as expressions):
     - `leadName`: `{{ $json.body.contact.name || $json.body.contact.firstName + ' ' + $json.body.contact.lastName }}`
     - `leadPhone`: `{{ $json.body.contact.phone }}`
     - `leadEmail`: `{{ $json.body.contact.email }}`
     - `serviceType`: `{{ $json.body.contact.customFields?.serviceType || 'General Landscaping' }}`
     - `leadId`: `{{ $json.body.contact.id }}`
   - Connect: **Workflow Configuration → Normalize Lead Information**

4. **Generate message with OpenAI**
   - Add **OpenAI** (LangChain) node named **“Generate Personalized Follow-Up Message”**
   - Configure prompt/content to include `leadName` and `serviceType` and instruct a friendly professional outreach message.
   - Set OpenAI credentials (API key or n8n-provided credits).
   - Connect: **Normalize Lead Information → Generate Personalized Follow-Up Message**
   - Validate the output field you’ll use later (ensure you know whether it returns `message`, `text`, etc.).

5. **Send SMS via GHL (HTTP Request)**
   - Add **HTTP Request** node named **“Send SMS via GoHighLevel”**
   - Method: `POST`
   - URL: `{{ $('Workflow Configuration').first().json.ghlApiUrl }}/conversations/messages`
   - Body: JSON with fields:
     - `type`: `SMS`
     - `contactId`: `{{ $json.leadId }}`
     - `message`: reference the OpenAI output (adjust field name as needed)
   - Headers:
     - `Authorization: Bearer {{ $('Workflow Configuration').first().json.ghlApiKey }}`
     - `Content-Type: application/json`
   - Connect: **Generate Personalized Follow-Up Message → Send SMS via GoHighLevel**

6. **Send Email via GHL (HTTP Request)**
   - Add **HTTP Request** node named **“Send Email via GoHighLevel”**
   - Same URL and auth headers as SMS
   - Body JSON:
     - `type`: `Email`
     - `contactId`
     - `subject`: static subject
     - `html`: the AI message (or a formatted HTML variant)
   - Connect: **Send SMS via GoHighLevel → Send Email via GoHighLevel**

7. **Check open status (HTTP Request)**
   - Add **HTTP Request** node named **“Check Email Open Status”**
   - Method: `GET`
   - URL: `{{ghlApiUrl}}/conversations/{{conversationId}}/messages`
   - Set timeout to ~30000ms.
   - **Important:** Determine where `conversationId` actually exists in the prior node output and reference it correctly (do not assume `$json.json.conversationId` unless verified).
   - Connect: **Send Email via GoHighLevel → Check Email Open Status**

8. **Branch: Email opened?**
   - Add **IF** node named **“Email Opened?”**
   - Condition: check whether any message has status `opened` (ensure correct JSON path from the check node output).
   - Connect: **Check Email Open Status → Email Opened?**

9. **If opened: schedule a follow-up**
   - Add **Google Calendar** node named **“Schedule Follow-Up in Calendar”**
   - Create event:
     - Start: `{{$now.plus({ days: 2 }).toISO()}}`
     - End: `{{$now.plus({ days: 2, hours: 1 }).toISO()}}`
     - Summary/description from normalized lead fields
   - Set Google Calendar OAuth2 credential.
   - Choose the target calendar ID.
   - Connect: **Email Opened? (true) → Schedule Follow-Up in Calendar**

10. **If not opened: send reminder SMS**
   - Add **HTTP Request** node named **“Send Reminder SMS”**
   - POST to `.../conversations/messages` with SMS reminder text using normalized lead info.
   - Connect: **Email Opened? (false) → Send Reminder SMS**

11. **Merge branches**
   - Add **Merge** node named **“Compile Lead Activity Report”**
   - Mode: `Combine`
   - Combine by: `Combine All`
   - Connect:
     - **Schedule Follow-Up in Calendar → Merge (Input 1)**
     - **Send Reminder SMS → Merge (Input 2)**

12. **Log to Google Sheets**
   - Add **Google Sheets** node named **“Log Lead Activity to Google Sheets”**
   - Credential: Google Sheets OAuth2
   - Operation: **Append or Update**
   - Document ID: your spreadsheet
   - Sheet name: `Lead Activity Log`
   - Matching column: `Lead ID`
   - Map columns: Lead ID, Lead Name, Phone, Email, Service Type, Email Opened, Status, Timestamp
   - **Important:** Add a small **Set** node in each branch before the merge to set:
     - `emailOpened: true/false`
     - `Status: 'Follow-up Scheduled'/'Reminder Sent'`
     so the sheet formulas don’t rely on a non-existent `emailOpened` field.
   - Connect: **Compile Lead Activity Report → Log Lead Activity to Google Sheets**

13. **Notify Slack**
   - Add **Slack** node named **“Notify Manager in Slack”**
   - Operation: Post message to channel
   - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Message includes lead info and the logged status.
   - Connect: **Log Lead Activity to Google Sheets → Notify Manager in Slack**
   - Configure Slack OAuth/token credential with permission to post in the target channel.

14. **Weekly summary entry point**
   - Add **Schedule Trigger** node named **“Weekly Summary Schedule”**
   - Configure weekly run (e.g., Monday 09:00).
   - Add **Google Sheets** node named **“Fetch Weekly Lead Data”**
     - Read range `A:H` from `Lead Activity Log`
     - (Optional) Filter to last 7 days using a Code node or by querying a filtered sheet/view.
   - Add **Slack** node named **“Send Weekly Summary to Slack”**
   - **Important:** Aggregate properly:
     - Use `{{$items().length}}` and filter across `{{$items()}}` (or add a Code node that returns a single summary item).
   - Connect: **Weekly Summary Schedule → Fetch Weekly Lead Data → Send Weekly Summary to Slack**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates lead follow-up and booking for landscaping companies. It takes a new GoHighLevel lead, generates personalized SMS and email messages via AI, logs activity in Google Sheets, notifies your Slack team, and schedules follow-ups for hot leads.” | Sticky note “Main” description |
| Setup steps include connecting GoHighLevel, Google Sheets, Slack, and OpenAI; customize message templates for branding and services. | Sticky note “Main” setup guidance |
| Contact for help/setup assistance: **Hyrum Hurst** | mailto:hyrum@quartersmart.com |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.