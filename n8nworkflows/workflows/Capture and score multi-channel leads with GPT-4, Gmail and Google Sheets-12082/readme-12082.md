Capture and score multi-channel leads with GPT-4, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/capture-and-score-multi-channel-leads-with-gpt-4--gmail-and-google-sheets-12082


# Capture and score multi-channel leads with GPT-4, Gmail and Google Sheets

## 1. Workflow Overview

**Title:** Capture and score multi-channel leads with GPT-4, Gmail and Google Sheets

**Purpose / use cases**  
This workflow captures inbound leads from multiple channels (Gmail inbox, web form webhook, chat webhook), normalizes them into a unified lead schema, enriches and qualifies them using GPT-4 (intent/urgency/buying signals), calculates a 0–100 lead score, and then:
- Generates and sends an appropriate email response (demo / info / nurturing), **or**
- Creates a call task and notifies the sales team in Slack, **or**
- Disqualifies the lead automatically.

All interactions are saved into **Google Sheets** (as a lightweight CRM), and high-scoring leads generate **Slack alerts**. It also includes scheduled blocks for **daily nurturing selection** and **weekly reporting**, plus global **error alerting** via Slack.

### 1.1 Logical blocks (high-level)
1. **Multi-channel lead capture (Entry points)**  
   Gmail trigger + two webhooks + schedules (nurturing/report) + error trigger.
2. **Normalization & validation**  
   Convert inputs into one schema; filter out notification/system emails.
3. **CRM lookup & history merge**  
   Pull existing leads from Google Sheets and build a bounded interaction history.
4. **AI analysis & scoring**  
   GPT extracts structured attributes; scoring algorithm produces score/stage/priority/next action.
5. **Response routing & execution**  
   Switch routes to demo/info/nurture/call/disqualify; generate response content; prepare/send emails or create tasks.
6. **CRM persistence & notifications**  
   Upsert lead row, log interaction row, and Slack alerts for hot leads.
7. **Automated nurturing & weekly reporting**  
   Daily selection of leads for nurturing + AI message generation (but currently **no send step**); weekly metrics posted to Slack.
8. **Error handling**  
   Any execution error triggers Slack alert with execution link.

---

## 2. Block-by-Block Analysis

### Block 1 — Multi-channel Lead Capture (Entry Points)
**Overview:** Receives lead events from Gmail, two HTTP webhooks, and scheduled triggers. These are independent entry points feeding downstream logic (except schedules, which have their own branches).  
**Nodes involved:** Gmail Trigger, Webhook - Web Form, Webhook - WhatsApp/Telegram, Auto Nurturing Trigger, Weekly Report Trigger, Error Trigger.

#### Node: Gmail Trigger
- **Type / role:** `gmailTrigger` — polls Gmail inbox for new messages.
- **Key configuration:**
  - Query: `newer_than:1d -from:me`
  - Label filter: `INBOX`
  - Polling: every minute
- **Outputs:** → Normalize Input
- **Failure modes / edge cases:**
  - Gmail OAuth expiry / missing scopes
  - Polling duplicates depending on Gmail trigger behavior and query window
  - Email parsing variability (from/subject fields)

#### Node: Webhook - Web Form
- **Type / role:** `webhook` — receives POST submissions (web form).
- **Key configuration:** POST `/ai-sales-agent`
- **Outputs:** → Normalize Input
- **Failure modes / edge cases:**
  - Public endpoint security (no auth configured)
  - Payload schema mismatch (missing expected fields like `email`, `message`)

#### Node: Webhook - WhatsApp/Telegram
- **Type / role:** `webhook` — receives POST events (chat integrations).
- **Key configuration:** POST `/ai-sales-chat`
- **Outputs:** → Normalize Input
- **Failure modes / edge cases:**
  - Same as web form webhook; also channel field may not match expected values

#### Node: Auto Nurturing Trigger
- **Type / role:** `scheduleTrigger` — daily run for nurturing selection.
- **Key configuration:** Cron `0 10 * * *` (10:00 daily)
- **Outputs:** → Leads for Nurturing
- **Failure modes:** timezone misunderstandings; schedule not firing if workflow inactive.

#### Node: Weekly Report Trigger
- **Type / role:** `scheduleTrigger` — weekly metrics.
- **Key configuration:** Cron `0 9 * * 1` (09:00 every Monday)
- **Outputs:** → Read Report Data

#### Node: Error Trigger
- **Type / role:** `errorTrigger` — starts a flow when any workflow execution errors.
- **Outputs:** → Slack - Error Alert

**Sticky note (applies to this block’s nodes):**  
“## Lead Capture … Multi-channel entry points: Gmail inbox polling, web form webhooks, and chat integrations. All sources are normalized into a unified lead format for consistent processing.”

---

### Block 2 — Normalization & Validation
**Overview:** Converts all inbound payloads into one canonical “lead” object, tags notifications/system emails, and stops processing if the message is not a real lead.  
**Nodes involved:** Normalize Input, Is Valid Lead?, Ignore.

#### Node: Normalize Input
- **Type / role:** `code` — builds a unified lead schema from Gmail or webhook payloads.
- **Key logic / variables:**
  - Detects source: `body ? 'webhook' : 'gmail'`
  - Produces unified fields:
    - `source`, `channel`, `email`, `name`, `phone`, `company`, `message`, `subject`,
      `messageId`, `threadId`, `timestamp`
  - Parses Gmail “From” using regex `(.*?)\s*<(.+?)>`
  - Notification detection via email patterns:
    - `noreply`, `no-reply`, `mailer-daemon`, `calendly.com`, `notifications`
  - Generates:
    - `isNotification`
    - `leadId = LEAD-${Date.now()}-${random}`
- **Inputs:** from Gmail Trigger or either Webhook
- **Outputs:** → Is Valid Lead?
- **Failure modes / edge cases:**
  - Missing/empty `email` from webhook payloads (downstream matching breaks)
  - Gmail trigger payload field names can vary; code attempts `from`/`From`, `subject`/`Subject`
  - LeadId uses `Date.now()` + random: possible collisions are unlikely but not impossible

#### Node: Is Valid Lead?
- **Type / role:** `if` — filters out notifications/system messages.
- **Configuration:** condition checks boolean `{{ $json.isNotification }}`
- **Connections:**
  - **True branch** (isNotification): → Ignore
  - **False branch** (valid lead): → Find Existing Lead
- **Edge cases:**
  - False positives/negatives in notification detection (e.g., real leads using “noreply” domains)

#### Node: Ignore
- **Type / role:** `noOp` — terminates notification/system message path.
- **Outputs:** none

**Sticky note (applies to this block’s nodes):**  
“## Validation … Filters out system notifications … checks if lead already exists … retrieves interaction history…”

---

### Block 3 — CRM Lookup & History Merge
**Overview:** Loads existing lead rows from Google Sheets and merges the current message into a bounded interaction history used later for AI context and scoring.  
**Nodes involved:** Find Existing Lead, Check History.

#### Node: Find Existing Lead
- **Type / role:** `googleSheets` — reads lead rows from the “Leads” sheet.
- **Configuration choices (interpreted):**
  - Document: `YOUR_DOCUMENT_ID`
  - Sheet: `Leads`
  - **Operation is not explicitly set in parameters.** In n8n this typically defaults to a “read/get many” operation depending on node defaults/version.
- **Connections:**
  - Input: Is Valid Lead? (valid path)
  - Output: → Check History
- **Failure modes / edge cases:**
  - Missing/incorrect Sheet ID
  - Google Sheets OAuth issues
  - Performance: reading the entire sheet for each inbound lead can be slow at scale
  - If operation defaults unexpectedly, it may not return any rows

#### Node: Check History
- **Type / role:** `code` — matches current lead by email against loaded sheet rows and builds history.
- **Key logic:**
  - `currentLead = $('Normalize Input').first().json`
  - `existingLeads = $input.all().map(i => i.json)`
  - `existingLead = existingLeads.find(l => l.email?.toLowerCase() === currentLead.email.toLowerCase())`
  - Initializes and updates:
    - `interactions`, `currentScore`, `currentStage`, `lastInteraction`
  - Parses `existingLead.history` as JSON; appends current interaction summary; keeps last 10.
  - Outputs merged object:
    - `isNew`, `interactions`, `currentScore`, `currentStage`, `lastInteraction`,
      `history`, `historyJSON`
- **Outputs:** → AI Analysis Engine
- **Failure modes / edge cases:**
  - If `currentLead.email` is empty, matching fails and everything becomes “new”
  - If the sheet’s column names differ from expected (`email`, `score`, `stage`, `history`, etc.), parsing/matching breaks
  - JSON parse errors handled (falls back to empty history)

---

### Block 4 — AI Analysis & Scoring
**Overview:** Sends the lead context + recent history to GPT, forces JSON output, then computes score/stage/priority/next action for routing.  
**Nodes involved:** AI Analysis Engine, Calculate Lead Score, Action Router.

#### Node: AI Analysis Engine
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion for structured lead analysis.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: 0.3 (more deterministic)
  - Max tokens: 1000
  - System prompt requires **strict JSON only** with keys:
    `sentiment, urgency, intent, budget_signals, decision_maker, pain_points, interests, next_action, lead_temperature, summary`
  - User message includes lead fields and `historyJSON`.
- **Inputs:** from Check History
- **Outputs:** → Calculate Lead Score
- **Failure modes / edge cases:**
  - OpenAI credential/config issues
  - Model returns non-JSON; downstream parsing attempts to strip code fences, then falls back to defaults
  - Prompt injection from inbound messages (mitigated somewhat by strict JSON requirement, but not guaranteed)

#### Node: Calculate Lead Score
- **Type / role:** `code` — parses AI JSON, computes score and derives stage/priority/flags.
- **Key logic:**
  - Reads lead context: `$('Check History').first().json`
  - Parses AI content from `($json.message?.content || $json.content || '{}')`
  - Fallback default analysis if JSON parse fails
  - Score components:
    - Channel value (`email` 10, `web_form` 15, `whatsapp` 20, etc.)
    - Sentiment (+15 positive, -10 negative)
    - Urgency (+25 high, +10 medium)
    - Intent (purchase +30, partnership +25, info +10, etc.)
    - Budget (+20 high, +10 medium)
    - Decision maker (+20)
    - Interaction bonus `min(interactions*5, 25)`
    - Temperature bonus (+15 hot, +5 warm)
  - Clamps score 0–100
  - Stage mapping:
    - ≥80 opportunity; ≥60 qualified; ≥40 interested; ≥20 contacted; else new
  - Priority:
    - High if score ≥70 OR urgency high; medium if ≥40; else low
  - Sets flags:
    - `nextAction` from AI
    - `autoReply = true` (always)
    - `notifySales = score >= 70 || lead_temperature == 'hot'`
- **Outputs:** → Action Router
- **Failure modes / edge cases:**
  - If AI output differs from expected keys (e.g., `budget_signals` missing), scoring may be weaker but still works due to defaults.
  - Interactions counting is inconsistent across the workflow (see Block 6 note).

#### Node: Action Router
- **Type / role:** `switch` — routes by `nextAction`.
- **Configuration:**
  - Switch value: `{{ $json.nextAction }}`
  - Rules: `schedule_demo`, `send_info`, `call`, `nurturing`, `disqualify`
- **Connections:**
  - schedule_demo → Generate Response - Demo
  - send_info → Generate Response - Info
  - call → Create Call Task
  - nurturing → Generate Response - Nurturing
  - disqualify → Disqualify Lead
- **Edge cases:**
  - If `nextAction` is not exactly one of these strings, no rule will match (execution stops on this branch unless n8n default output is configured; not shown here).

**Sticky note (applies to this block’s nodes):**  
“## AI Analysis & Scoring … GPT-4 extracts … scoring engine combines AI insights with channel value and interaction history…”

---

### Block 5 — Response Generation (Email / Call / Disqualify)
**Overview:** Depending on routing, the workflow generates a tailored response using GPT, builds an HTML email template (for email paths), sends via Gmail, or creates a Slack call task. Disqualified leads skip email send.  
**Nodes involved:** Generate Response - Demo, Generate Response - Info, Generate Response - Nurturing, Prepare Email - Demo, Prepare Email - Info, Prepare Email - Nurturing, Send Email, Create Call Task, Slack - Call Task, Disqualify Lead.

#### Node: Generate Response - Demo
- **Type / role:** OpenAI — produces plain-text demo invitation.
- **Configuration:**
  - Model `gpt-4o-mini`, temp 0.7, maxTokens 500
  - System prompt includes placeholders:
    - COMPANY / PRODUCT must be customized
    - Calendly link: `https://calendly.com/your-user/demo`
  - Uses lead fields from `$('Calculate Lead Score').item.json.*`
- **Outputs:** → Prepare Email - Demo
- **Edge cases:** if lead name missing, greeting may be odd.

#### Node: Prepare Email - Demo
- **Type / role:** `code` — wraps AI response into branded HTML email with CTA button.
- **Configuration choices:**
  - CONFIG placeholders: `companyName`, colors, `calendlyLink`
  - Subject becomes `Re: <original subject>` if not already `Re:`
  - Output fields: `to`, `subject`, `htmlContent`, `aiResponse`, `responseType='demo'`
- **Outputs:** → Send Email
- **Edge cases:** empty subject from webhook leads to `Re: `.

#### Node: Generate Response - Info
- **Type / role:** OpenAI — provides info and nudges towards a call/demo.
- **Outputs:** → Prepare Email - Info

#### Node: Prepare Email - Info
- **Type / role:** `code` — HTML email template for informational response.
- **CONFIG placeholders:** `companyName`, `primaryColor`, `calendlyLink`
- **Outputs:** → Send Email

#### Node: Generate Response - Nurturing
- **Type / role:** OpenAI — short friendly nurturing message.
- **Outputs:** → Prepare Email - Nurturing

#### Node: Prepare Email - Nurturing
- **Type / role:** `code` — HTML email template for nurturing response.
- **Outputs:** → Send Email

#### Node: Send Email
- **Type / role:** `gmail` — sends email reply.
- **Key configuration:**
  - To: `{{ $json.to }}`
  - Subject: `{{ $json.subject }}`
  - Message: `{{ $json.htmlContent }}`
  - `appendAttribution: false`
  - **onError:** `continueErrorOutput` (workflow continues even if sending fails)
- **Outputs:** → Save/Update Lead (both success and error output are connected)
- **Failure modes / edge cases:**
  - Gmail sending limits, invalid recipient
  - HTML content may be rejected by strict clients (generally fine)
  - Because errors continue, CRM may record a “response” even if it was not actually sent unless you add explicit error handling.

#### Node: Create Call Task
- **Type / role:** `code` — builds a structured `task` object for sales outreach.
- **Output includes:** `responseType='call_task'` and `task` (type/priority/lead/email/phone/reason/created)
- **Outputs:** → Slack - Call Task

#### Node: Slack - Call Task
- **Type / role:** `slack` — posts a call task notification.
- **Configuration:** message text “New Task: Call Lead”
- **Outputs:** → Save/Update Lead
- **Edge cases:** Slack credentials/webhook errors.

#### Node: Disqualify Lead
- **Type / role:** `code` — marks lead as disqualified and skips email sending.
- **Sets:**
  - `stage = 'disqualified'`
  - `responseType = 'disqualified'`
  - `aiResponse = 'Lead automatically disqualified'`
  - Empty email fields (`subject`, `htmlContent`) retained
- **Outputs:** → Save/Update Lead
- **Edge cases:** still updates CRM and logs interaction.

**Sticky note (applies to this block’s nodes):**  
“## Response Generation … demo invitations for hot leads, informative content for warm leads, and gentle nurturing … Call tasks are created…”

---

### Block 6 — CRM Persistence & Sales Notifications
**Overview:** Persists the lead to Google Sheets (append vs update), logs the interaction, and posts Slack alerts for hot leads (score ≥ 70).  
**Nodes involved:** Save/Update Lead, Is New Lead?, Sheet - New Lead, Sheet - Update Lead, Log Interaction, Notify Sales?, Slack - Hot Lead Alert, Done.

#### Node: Save/Update Lead
- **Type / role:** `code` — maps internal lead object into Google Sheets row schema.
- **Key mapping fields:** `lead_id, email, name, company, phone, score, stage, priority, temperature, sentiment, intent, urgency, ...`
- **Important detail / edge case: interactions counting bug**
  - It sets: `interactions: lead.interactions + 1`
  - But `lead.interactions` already got incremented in **Check History** for existing leads, and it also represents the current interaction in other places.
  - This can cause double increments, especially for returning leads.
- **Outputs:** → Is New Lead?
- **Failure modes:** if `message` is null, `lead.message.substring` may throw (in current code it assumes `message` is a string).

#### Node: Is New Lead?
- **Type / role:** `if` — chooses append vs update.
- **Condition:** `{{ $json.is_new }}` equals `"Yes"`
- **True:** → Sheet - New Lead  
- **False:** → Sheet - Update Lead

#### Node: Sheet - New Lead
- **Type / role:** `googleSheets` append row into `Leads`.
- **Configuration:** document `YOUR_DOCUMENT_ID`, sheet `Leads`, operation `append`, mapping `autoMapInputData`
- **Outputs:** → Log Interaction

#### Node: Sheet - Update Lead
- **Type / role:** `googleSheets` update row in `Leads`.
- **Configuration:** operation `update`, mapping `autoMapInputData`
- **Critical requirement / edge case:** Google Sheets “update” typically requires a row identifier (row number or a key column mapping). The workflow does not show any explicit “match by lead_id/email” configuration. If not configured in UI, updates may fail or update the wrong row.

#### Node: Log Interaction
- **Type / role:** `googleSheets` append row into `Interactions`.
- **Mapping (explicit):**
  - Email, Score, Stage, Intent, Channel, Lead ID, Subject, Timestamp, Response Type
  - Most values referenced from `$('Save/Update Lead').item.json.*`
- **Outputs:** → Notify Sales?

#### Node: Notify Sales?
- **Type / role:** `if` — hot lead alert gate.
- **Condition:** score ≥ 70 using `{{ $('Calculate Lead Score').item.json.score }}`
- **True:** → Slack - Hot Lead Alert  
- **False:** → Done

#### Node: Slack - Hot Lead Alert
- **Type / role:** `slack` — posts “Hot Lead Detected”.
- **Outputs:** → Done

#### Node: Done
- **Type / role:** `noOp` — terminator.

**Sticky note (applies to this block’s nodes):**  
“## CRM & Notifications … Emails are sent via Gmail, lead data is saved or updated in Google Sheets … Hot leads (score 70+) trigger instant Slack alerts…”

---

### Block 7 — Auto Nurturing (Daily)
**Overview:** Once per day, selects leads in a “warm” score band with no recent activity and generates a nurturing email with GPT. **This branch currently does not send the email or log the interaction.**  
**Nodes involved:** Leads for Nurturing, Filter for Nurturing, AI Nurturing Email (plus Auto Nurturing Trigger as the entry).

#### Node: Leads for Nurturing
- **Type / role:** `googleSheets` — reads rows from `Leads`.
- **Outputs:** → Filter for Nurturing
- **Edge cases:** scale/performance if Leads sheet grows.

#### Node: Filter for Nurturing
- **Type / role:** `code` — selects candidates for follow-up.
- **Selection rules:**
  - `score` between 20 and 60
  - `stage` not in `disqualified/won/lost`
  - `daysSinceContact >= 3` based on `last_interaction`
  - Take up to 10 leads
- **Outputs:** → AI Nurturing Email

#### Node: AI Nurturing Email
- **Type / role:** OpenAI — generates short follow-up text.
- **Input fields referenced:** `name`, `interests`, `pain_points`, `last_subject`
- **Outputs:** **No downstream connection** in the provided workflow.
- **Practical implication:** nurturing content is generated but never emailed, saved, or logged.

**Sticky note (applies to this block’s nodes):**  
“## Auto Nurturing … Runs daily at 10 AM … sends AI-generated follow-up emails…”  
(Note: the described behavior requires additional nodes; the current graph stops after generation.)

---

### Block 8 — Weekly Reports
**Overview:** Reads all leads, computes weekly pipeline metrics, and posts a summary to Slack.  
**Nodes involved:** Read Report Data, Calculate Metrics, Slack - Weekly Report (plus Weekly Report Trigger as entry).

#### Node: Read Report Data
- **Type / role:** `googleSheets` — reads `Leads`.
- **Outputs:** → Calculate Metrics

#### Node: Calculate Metrics
- **Type / role:** `code` — computes report statistics for last 7 days.
- **Outputs fields:**
  - `period` (date range)
  - `totalLeads`, `newThisWeek`, `hotLeads`, `avgScore`
  - breakdown objects + JSON strings: `stages/channels/intents`, `byStageJSON/byChannelJSON/byIntentJSON`
- **Outputs:** → Slack - Weekly Report
- **Edge cases:**
  - Assumes `last_interaction` is parseable date
  - `newThisWeek` uses `is_new === 'Yes'` AND last_interaction within 7 days (may not reflect actual creation date)

#### Node: Slack - Weekly Report
- **Type / role:** `slack` — posts “Weekly Lead Report”.
- **Edge case:** message currently does not include computed metrics (only static text). To be useful, it should interpolate `$json` metrics.

**Sticky note (applies to this block’s nodes):**  
“## Weekly Reports … calculates pipeline metrics … Summary is posted to Slack…”

---

### Block 9 — Error Handling
**Overview:** Any workflow execution error triggers a Slack alert with a link to the failed execution.  
**Nodes involved:** Slack - Error Alert (triggered by Error Trigger).

#### Node: Slack - Error Alert
- **Type / role:** `slack` — sends error notification.
- **Configuration:** text “Error in AI Sales Agent”; includes link to workflow execution (`includeLinkToWorkflow: true`)
- **Failure modes:** Slack connection issues; consider also emailing admins.

**Sticky note (applies to this block’s nodes):**  
“## Error Handling … posts detailed alerts … link to the failed execution…”

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Entry point: poll inbox for new emails | — | Normalize Input | ## Lead Capture … Multi-channel entry points… |
| Webhook - Web Form | n8n-nodes-base.webhook | Entry point: web form POST ingestion | — | Normalize Input | ## Lead Capture … Multi-channel entry points… |
| Webhook - WhatsApp/Telegram | n8n-nodes-base.webhook | Entry point: chat POST ingestion | — | Normalize Input | ## Lead Capture … Multi-channel entry points… |
| Normalize Input | n8n-nodes-base.code | Normalize all sources into unified lead schema | Gmail Trigger; Webhook - Web Form; Webhook - WhatsApp/Telegram | Is Valid Lead? | ## Lead Capture … normalized into a unified lead format… |
| Is Valid Lead? | n8n-nodes-base.if | Filter out notifications/system emails | Normalize Input | Ignore; Find Existing Lead | ## Validation … Filters out system notifications… |
| Ignore | n8n-nodes-base.noOp | Stop processing notifications | Is Valid Lead? | — | ## Validation … Filters out system notifications… |
| Find Existing Lead | n8n-nodes-base.googleSheets | Read existing leads from Google Sheets | Is Valid Lead? | Check History | ## Validation … checks if lead already exists… |
| Check History | n8n-nodes-base.code | Match by email; merge history and prior metrics | Find Existing Lead | AI Analysis Engine | ## Validation … retrieves interaction history… |
| AI Analysis Engine | @n8n/n8n-nodes-langchain.openAi | GPT structured lead analysis (JSON) | Check History | Calculate Lead Score | ## AI Analysis & Scoring … GPT-4 extracts… |
| Calculate Lead Score | n8n-nodes-base.code | Parse AI JSON; compute score/stage/priority/nextAction | AI Analysis Engine | Action Router | ## AI Analysis & Scoring … scoring engine… |
| Action Router | n8n-nodes-base.switch | Route by nextAction | Calculate Lead Score | Generate Response - Demo; Generate Response - Info; Create Call Task; Generate Response - Nurturing; Disqualify Lead | ## AI Analysis & Scoring … routes to the appropriate response path. |
| Generate Response - Demo | @n8n/n8n-nodes-langchain.openAi | Write demo invitation text | Action Router | Prepare Email - Demo | ## Response Generation … demo invitations… |
| Prepare Email - Demo | n8n-nodes-base.code | Convert demo text into branded HTML email | Generate Response - Demo | Send Email | ## Response Generation … demo invitations… |
| Generate Response - Info | @n8n/n8n-nodes-langchain.openAi | Write informative reply text | Action Router | Prepare Email - Info | ## Response Generation … informative content… |
| Prepare Email - Info | n8n-nodes-base.code | Convert info text into HTML email | Generate Response - Info | Send Email | ## Response Generation … informative content… |
| Generate Response - Nurturing | @n8n/n8n-nodes-langchain.openAi | Write nurturing reply text | Action Router | Prepare Email - Nurturing | ## Response Generation … gentle nurturing… |
| Prepare Email - Nurturing | n8n-nodes-base.code | Convert nurturing text into HTML email | Generate Response - Nurturing | Send Email | ## Response Generation … gentle nurturing… |
| Send Email | n8n-nodes-base.gmail | Send HTML email via Gmail | Prepare Email - Demo; Prepare Email - Info; Prepare Email - Nurturing | Save/Update Lead | ## CRM & Notifications … Emails are sent via Gmail… |
| Disqualify Lead | n8n-nodes-base.code | Mark lead disqualified and skip email | Action Router | Save/Update Lead | ## Response Generation … Call tasks are created… |
| Create Call Task | n8n-nodes-base.code | Create call task payload | Action Router | Slack - Call Task | ## Response Generation … Call tasks are created… |
| Slack - Call Task | n8n-nodes-base.slack | Notify sales to call lead | Create Call Task | Save/Update Lead | ## Response Generation … Call tasks are created… |
| Save/Update Lead | n8n-nodes-base.code | Map lead into CRM row schema | Send Email; Slack - Call Task; Disqualify Lead | Is New Lead? | ## CRM & Notifications … lead data is saved or updated… |
| Is New Lead? | n8n-nodes-base.if | Choose append vs update | Save/Update Lead | Sheet - New Lead; Sheet - Update Lead | ## CRM & Notifications … saved or updated… |
| Sheet - New Lead | n8n-nodes-base.googleSheets | Append to Leads sheet | Is New Lead? | Log Interaction | ## CRM & Notifications … saved or updated… |
| Sheet - Update Lead | n8n-nodes-base.googleSheets | Update existing row in Leads sheet | Is New Lead? | Log Interaction | ## CRM & Notifications … saved or updated… |
| Log Interaction | n8n-nodes-base.googleSheets | Append to Interactions sheet | Sheet - New Lead; Sheet - Update Lead | Notify Sales? | ## CRM & Notifications … interactions are logged… |
| Notify Sales? | n8n-nodes-base.if | Gate Slack alert for hot leads | Log Interaction | Slack - Hot Lead Alert; Done | ## CRM & Notifications … Hot leads (score 70+) trigger instant Slack alerts… |
| Slack - Hot Lead Alert | n8n-nodes-base.slack | Slack alert for hot lead | Notify Sales? | Done | ## CRM & Notifications … Hot leads (score 70+)… |
| Done | n8n-nodes-base.noOp | Flow terminator | Notify Sales?; Slack - Hot Lead Alert | — | ## CRM & Notifications … |
| Auto Nurturing Trigger | n8n-nodes-base.scheduleTrigger | Daily entry point for nurturing | — | Leads for Nurturing | ## Auto Nurturing … Runs daily at 10 AM… |
| Leads for Nurturing | n8n-nodes-base.googleSheets | Read leads for nurturing | Auto Nurturing Trigger | Filter for Nurturing | ## Auto Nurturing … Identifies warm leads… |
| Filter for Nurturing | n8n-nodes-base.code | Select leads needing follow-up | Leads for Nurturing | AI Nurturing Email | ## Auto Nurturing … score 20-60 … 3+ days… |
| AI Nurturing Email | @n8n/n8n-nodes-langchain.openAi | Generate nurturing follow-up text | Filter for Nurturing | — | ## Auto Nurturing … sends AI-generated follow-up emails… |
| Weekly Report Trigger | n8n-nodes-base.scheduleTrigger | Weekly entry point for report | — | Read Report Data | ## Weekly Reports … Every Monday at 9 AM… |
| Read Report Data | n8n-nodes-base.googleSheets | Read all leads for reporting | Weekly Report Trigger | Calculate Metrics | ## Weekly Reports … calculates pipeline metrics… |
| Calculate Metrics | n8n-nodes-base.code | Compute weekly KPIs and breakdowns | Read Report Data | Slack - Weekly Report | ## Weekly Reports … calculates pipeline metrics… |
| Slack - Weekly Report | n8n-nodes-base.slack | Post report to Slack | Calculate Metrics | — | ## Weekly Reports … Summary is posted to Slack… |
| Error Trigger | n8n-nodes-base.errorTrigger | Entry point on workflow error | — | Slack - Error Alert | ## Error Handling … Catches any workflow errors… |
| Slack - Error Alert | n8n-nodes-base.slack | Notify errors to Slack (with link) | Error Trigger | — | ## Error Handling … posts detailed alerts… |
| Overview (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 1 Lead Capture (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 2 Validation (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 3 AI Analysis (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 4 Response (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 5 CRM (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 6 Nurturing (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Step 7 Reports (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |
| Error Handling (Sticky) | n8n-nodes-base.stickyNote | Documentation block (not executed) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

1. **Create a new workflow** in n8n and set the workflow name to the provided title.

2. **Add credentials (required)**
   1. **Gmail OAuth2** credential (for Gmail Trigger + Gmail Send).
      - Ensure scopes allow reading and sending email.
   2. **Google Sheets OAuth2** credential.
      - Ensure access to the target spreadsheet.
   3. **Slack** credential (or Slack webhook-style credential depending on your Slack node setup).
   4. **OpenAI / LangChain OpenAI** credential for `@n8n/n8n-nodes-langchain.openAi`.

3. **Create your Google Sheet**
   - Create a spreadsheet with at least these tabs:
     - `Leads`
     - `Interactions`
     - (Optional if you extend later) `Tasks`
   - Copy the spreadsheet ID and replace all `YOUR_DOCUMENT_ID` usages in Google Sheets nodes.

4. **Block: Lead capture entry points**
   1. Add **Gmail Trigger**
      - Filter query: `newer_than:1d -from:me`
      - Label: `INBOX`
      - Polling: every minute
      - Select your Gmail credential
   2. Add **Webhook** named “Webhook - Web Form”
      - Method: POST
      - Path: `ai-sales-agent`
   3. Add **Webhook** named “Webhook - WhatsApp/Telegram”
      - Method: POST
      - Path: `ai-sales-chat`
   4. Add **Schedule Trigger** named “Auto Nurturing Trigger”
      - Cron: `0 10 * * *`
   5. Add **Schedule Trigger** named “Weekly Report Trigger”
      - Cron: `0 9 * * 1`
   6. Add **Error Trigger** node.

5. **Add Normalize + validation**
   1. Add **Code** node “Normalize Input” and paste the normalization JavaScript (adapt fields if your webhooks differ).
   2. Connect:
      - Gmail Trigger → Normalize Input
      - Webhook - Web Form → Normalize Input
      - Webhook - WhatsApp/Telegram → Normalize Input
   3. Add **IF** node “Is Valid Lead?”
      - Condition: Boolean `={{ $json.isNotification }}`
   4. Add **NoOp** node “Ignore”
   5. Connect:
      - Normalize Input → Is Valid Lead?
      - Is Valid Lead? (true) → Ignore

6. **Add CRM lookup & history**
   1. Add **Google Sheets** node “Find Existing Lead”
      - Document ID: your spreadsheet ID
      - Sheet: `Leads`
      - Operation: configure to **read/get many rows** (return all rows) unless you implement a query filter by email.
   2. Add **Code** node “Check History” and paste the history merge logic.
   3. Connect:
      - Is Valid Lead? (false) → Find Existing Lead
      - Find Existing Lead → Check History

7. **Add AI analysis & scoring**
   1. Add **OpenAI (LangChain)** node “AI Analysis Engine”
      - Model: `gpt-4o-mini`
      - Temperature: 0.3
      - Max tokens: 1000
      - Add a system message enforcing JSON-only output (as in the workflow).
      - Add a user message with lead context and `historyJSON`.
   2. Add **Code** node “Calculate Lead Score” and paste the scoring JS.
   3. Add **Switch** node “Action Router”
      - Value to evaluate: `={{ $json.nextAction }}`
      - Rules (string equals):
        - schedule_demo
        - send_info
        - call
        - nurturing
        - disqualify
   4. Connect:
      - Check History → AI Analysis Engine → Calculate Lead Score → Action Router

8. **Add response generation branches**
   1. **Demo branch**
      - Add OpenAI node “Generate Response - Demo” (temp 0.7, maxTokens 500) with your company/product/calendly details.
      - Add Code node “Prepare Email - Demo” (HTML template + CTA).
      - Connect: Action Router (schedule_demo) → Generate Response - Demo → Prepare Email - Demo
   2. **Info branch**
      - Add OpenAI node “Generate Response - Info”
      - Add Code node “Prepare Email - Info”
      - Connect: Action Router (send_info) → Generate Response - Info → Prepare Email - Info
   3. **Nurturing branch**
      - Add OpenAI node “Generate Response - Nurturing”
      - Add Code node “Prepare Email - Nurturing”
      - Connect: Action Router (nurturing) → Generate Response - Nurturing → Prepare Email - Nurturing
   4. **Call branch**
      - Add Code node “Create Call Task”
      - Add Slack node “Slack - Call Task”
      - Connect: Action Router (call) → Create Call Task → Slack - Call Task
   5. **Disqualify branch**
      - Add Code node “Disqualify Lead”
      - Connect: Action Router (disqualify) → Disqualify Lead

9. **Add Gmail sending (for email branches)**
   1. Add **Gmail** node “Send Email”
      - To: `={{ $json.to }}`
      - Subject: `={{ $json.subject }}`
      - Message/body: `={{ $json.htmlContent }}`
      - Turn off attribution append
      - Set **On Error** behavior to continue (if desired)
   2. Connect:
      - Prepare Email - Demo → Send Email
      - Prepare Email - Info → Send Email
      - Prepare Email - Nurturing → Send Email

10. **Add CRM mapping + upsert logic**
   1. Add **Code** node “Save/Update Lead” to map fields for the `Leads` sheet.
   2. Connect:
      - Send Email → Save/Update Lead
      - Slack - Call Task → Save/Update Lead
      - Disqualify Lead → Save/Update Lead
   3. Add **IF** node “Is New Lead?”
      - Condition string equals: `={{ $json.is_new }}` == `Yes`
   4. Add **Google Sheets** node “Sheet - New Lead”
      - Operation: Append
      - Sheet: `Leads`
      - Mapping: Auto-map input data
   5. Add **Google Sheets** node “Sheet - Update Lead”
      - Operation: Update
      - Sheet: `Leads`
      - Mapping: Auto-map input data
      - Configure a proper row match key (e.g., row number, or match on `lead_id`/`email` depending on your chosen method).
   6. Connect:
      - Save/Update Lead → Is New Lead?
      - Is New Lead? (true) → Sheet - New Lead
      - Is New Lead? (false) → Sheet - Update Lead

11. **Add interaction logging + hot lead Slack alert**
   1. Add **Google Sheets** node “Log Interaction”
      - Operation: Append
      - Sheet: `Interactions`
      - Use explicit mapping fields (Email/Score/Stage/Intent/Channel/Lead ID/Subject/Timestamp/Response Type)
   2. Add **IF** node “Notify Sales?”
      - Condition number: `={{ $('Calculate Lead Score').item.json.score }}` ≥ `70`
   3. Add **Slack** node “Slack - Hot Lead Alert”
      - Text: “Hot Lead Detected”
   4. Add **NoOp** node “Done”
   5. Connect:
      - Sheet - New Lead → Log Interaction
      - Sheet - Update Lead → Log Interaction
      - Log Interaction → Notify Sales?
      - Notify Sales? (true) → Slack - Hot Lead Alert → Done
      - Notify Sales? (false) → Done

12. **Add daily nurturing branch**
   1. Add Google Sheets node “Leads for Nurturing” (read `Leads`)
   2. Add Code node “Filter for Nurturing” (score 20–60, last_interaction ≥ 3 days, max 10)
   3. Add OpenAI node “AI Nurturing Email”
   4. Connect:
      - Auto Nurturing Trigger → Leads for Nurturing → Filter for Nurturing → AI Nurturing Email
   5. **Important:** to actually send nurturing emails, add:
      - a “Prepare Email” code node (similar to Prepare Email - Nurturing),
      - then connect to “Send Email” (or a dedicated Gmail send node),
      - then log interaction and update last_interaction in Sheets.

13. **Add weekly report branch**
   1. Add Google Sheets node “Read Report Data” (read `Leads`)
   2. Add Code node “Calculate Metrics”
   3. Add Slack node “Slack - Weekly Report”
      - Include metrics in message text using expressions (currently it’s static).
   4. Connect:
      - Weekly Report Trigger → Read Report Data → Calculate Metrics → Slack - Weekly Report

14. **Add error handling branch**
   1. Add Slack node “Slack - Error Alert” with “include link to workflow execution” enabled.
   2. Connect:
      - Error Trigger → Slack - Error Alert

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Create a Google Sheet with three tabs: Leads, Interactions, and Tasks” | Mentioned in the Overview sticky note; the current workflow uses `Leads` and `Interactions` only. |
| Replace `YOUR_DOCUMENT_ID` in all Google Sheets nodes | Required to make any Sheets read/write work. |
| Customize email templates (company info + Calendly link) | Required in “Prepare Email - *” code nodes and AI prompts (COMPANY/PRODUCT placeholders). |
| Auto nurturing description vs implementation mismatch | Sticky note claims emails are sent; current workflow stops after generating the nurturing email text. |
| Weekly report Slack message is static | “Slack - Weekly Report” currently posts “Weekly Lead Report” without metrics; add expressions to include computed values. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.