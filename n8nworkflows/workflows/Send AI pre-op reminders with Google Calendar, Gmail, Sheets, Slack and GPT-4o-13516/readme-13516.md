Send AI pre-op reminders with Google Calendar, Gmail, Sheets, Slack and GPT-4o

https://n8nworkflows.xyz/workflows/send-ai-pre-op-reminders-with-google-calendar--gmail--sheets--slack-and-gpt-4o-13516


# Send AI pre-op reminders with Google Calendar, Gmail, Sheets, Slack and GPT-4o

## 1. Workflow Overview

**Workflow name:** AI-Powered Patient Pre-Op Reminder & Nurse Alert System  
**Template title provided:** Send AI pre-op reminders with Google Calendar, Gmail, Sheets, Slack and GPT-4o

**Purpose:**  
Automate daily pre-operative checklist reminder emails for patients scheduled for surgery, capture patient confirmations via a webhook link, store confirmation status in Google Sheets, and periodically alert staff on Slack when a patient has not confirmed.

**Target use cases:**
- Clinics/hospitals that schedule surgeries in Google Calendar and want automated, trackable patient readiness confirmations.
- Operations teams that need a lightweight audit trail (Google Sheets) and escalation (Slack) for missing confirmations.

### 1.1 Daily Intake & Patient Extraction (Calendar → structured patient items)
Runs every day at 9:00 AM, fetches calendar events, filters “surgery” events, and extracts patient fields from event descriptions.

### 1.2 Link Generation & Data Normalization
Creates a unique confirmation URL per patient and applies basic cleanup to extracted fields.

### 1.3 AI Email Generation & Delivery (GPT-4o → Gmail)
Uses Azure OpenAI (gpt-4o) to generate a patient-friendly HTML email + subject, parses the model output into structured JSON, and sends the email via Gmail.

### 1.4 Confirmation Capture (Webhook → Sheets)
Receives confirmation clicks from patients (GET /confirm), parses query params, and upserts confirmation data to Google Sheets.

### 1.5 Periodic Non-Confirmation Escalation (Sheets → Slack)
On an hourly schedule, reads the sheet, routes unconfirmed rows, and sends Slack alerts to a specified user.

---

## 2. Block-by-Block Analysis

### Block 1 — Daily Surgery Intake & Patient Extraction
**Overview:**  
Triggers daily, retrieves today’s events from Google Calendar, filters only surgery-related events, and extracts patient data fields from the calendar event description.

**Nodes involved:**
- Schedule Trigger: Daily 9:00 AM
- Google Calendar: Fetch Today’s Events
- Extract Surgery Events + Patient Fields

#### Node: Schedule Trigger: Daily 9:00 AM
- **Type / role:** `Schedule Trigger` — time-based entry point.
- **Configuration:** Cron expression `0 9 * * *` (daily at 09:00).
- **Outputs:** Emits one trigger item to Google Calendar node.
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings; may not be local clinic timezone unless configured globally.

#### Node: Google Calendar: Fetch Today’s Events
- **Type / role:** `Google Calendar` — reads calendar events.
- **Configuration choices:**
  - Operation: **Get All** events (no explicit time window shown in parameters; relies on node defaults/options).
  - Calendar selected: `user@example.com` (list mode).
  - OAuth2 credential: “Google Calendar account -anuj”.
- **Inputs:** From daily schedule trigger.
- **Outputs:** A list of calendar events to the Code node.
- **Edge cases / failures:**
  - OAuth token expiration / consent issues.
  - If the node doesn’t constrain to “today” via options, it may fetch more than intended (depends on defaults); consider explicitly setting `timeMin/timeMax` in options if needed.
  - Events without `description`, `start.dateTime`, or `summary`.

#### Node: Extract Surgery Events + Patient Fields
- **Type / role:** `Code` — transforms events into structured “patient surgery items”.
- **Key logic:**
  - Filters events where `event.summary` contains `"surgery"` (case-insensitive).
  - Strips HTML tags from `event.description` and normalizes whitespace.
  - Extracts values with regex for keys:
    - `patient_name`, `patient_email`, `patient_phone`, `patient_id`, `procedure`
  - Outputs JSON with:
    - `event_id`, `surgery_time` (from `event.start.dateTime`), patient fields, `procedure`, `calendar_link` (`event.htmlLink`)
- **Inputs/outputs:**
  - Input: Calendar event items.
  - Output: One item per matching surgery event.
- **Edge cases / failures:**
  - Regex extraction is fragile if description formatting varies (multi-word values, line breaks, “key: value” not consistent).
  - `surgery_time` will be `undefined` for all-day events (`start.date` instead of `start.dateTime`).
  - If summary doesn’t include the word “surgery”, the event is ignored.

---

### Block 2 — Unique Confirmation Link & Data Cleanup
**Overview:**  
Creates a per-patient confirmation URL (with uniqueId) and performs string cleanup on extracted patient fields.

**Nodes involved:**
- Build Unique Confirmation Link (confirmUrl)
- Clean Patient Fields (Name/Email/Phone/ID)

#### Node: Build Unique Confirmation Link (confirmUrl)
- **Type / role:** `Code` — adds tracking identifiers + confirmation link.
- **Key logic:**
  - Builds `uniqueId` = `confirm_{patient_id}_{Date.now()}_{index}`.
  - Builds `confirmUrl`:
    - `https://n8n.tdwebsites.in/webhook-test/confirm?patient_id=...&uniqueId=...`
  - Adds `createdAt` ISO timestamp.
- **Inputs/outputs:**
  - Input: extracted patient surgery items.
  - Output: same items enriched with `uniqueId`, `confirmUrl`, `createdAt`.
- **Important integration note:**
  - The URL uses `/webhook-test/confirm`, which corresponds to **test** webhooks in n8n. In production it is typically `/webhook/confirm` (unless your instance is configured differently).
- **Edge cases / failures:**
  - If `patient_id` is null/empty, uniqueId and confirmUrl become malformed and matching/upsert logic becomes unreliable.
  - `Date.now()` makes links non-deterministic; retries can generate different links for the same patient/event.

#### Node: Clean Patient Fields (Name/Email/Phone/ID)
- **Type / role:** `Code` — sanitizes extracted strings.
- **Key logic:** Removes specific substrings from fields:
  - `patient_name: replace(' patient_email','')`
  - `patient_email: replace(' patient_phone','')`
  - `patient_phone: replace(' patient_id','')`
  - `patient_id: replace(' procedure','')`
- **Inputs/outputs:** Enriched patient items → cleaned patient items.
- **Edge cases / failures:**
  - This cleanup implies the earlier regex may be over-capturing text fragments. It’s not robust: if formatting differs, cleanup may do nothing or remove valid data.
  - Only replaces the first occurrence; may leave trailing artifacts.

---

### Block 3 — AI Email Generation & Gmail Delivery
**Overview:**  
Uses GPT-4o (Azure OpenAI) to generate a compliant patient email (subject + HTML body) including the confirmation button linking to `confirmUrl`, then sends it via Gmail.

**Nodes involved:**
- AI: Generate Pre-Op Checklist Email (Subject + HTML Body)
- LLM: Azure OpenAI Chat Model (gpt-4o)
- Parse AI Output (Subject/Body JSON)
- Gmail: Send Pre-Op Checklist Reminder

#### Node: LLM: Azure OpenAI Chat Model (gpt-4o)
- **Type / role:** LangChain Azure OpenAI chat model node — provides the LLM.
- **Configuration:**
  - Model: `gpt-4o`
  - Credential: “Azure Open AI account”
- **Connections:** Connected to the Agent node via `ai_languageModel`.
- **Edge cases / failures:**
  - Azure deployment/model name mismatch (Azure often requires a deployment name; ensure n8n credential is configured accordingly).
  - Rate limits, content filters, timeouts.

#### Node: Parse AI Output (Subject/Body JSON)
- **Type / role:** Structured Output Parser — enforces JSON schema-like output.
- **Configuration:**
  - Example schema:
    ```json
    { "Subject": " ", "Body": " " }
    ```
- **Connections:** Feeds into the Agent node as `ai_outputParser`.
- **Edge cases / failures:**
  - If the model returns invalid JSON or extra keys, parsing can fail and stop the email send.

#### Node: AI: Generate Pre-Op Checklist Email (Subject + HTML Body)
- **Type / role:** LangChain Agent — prompt orchestration to produce the email.
- **Configuration choices (interpreted):**
  - Prompt includes the full patient JSON (`{{ JSON.stringify($json, null, 2) }}`).
  - Requires:
    - Subject exactly: `"Pre-Surgery Checklist – Action Required"`
    - HTML body with greeting by name, surgery time + procedure, checklist bullets, green confirmation button linking to `confirmUrl`, and a follow-up warning footer.
  - System message enforces:
    - No invented details
    - No medical advice beyond basic reminders
    - Output must be valid JSON with exactly keys `Subject` and `Body`
- **Inputs/outputs:**
  - Input: cleaned patient item.
  - Output: Agent result used by Gmail node; Gmail references `{{ $json.output.Subject }}` and `{{ $json.output.Body }}`.
- **Edge cases / failures:**
  - Missing `patient_name`, `procedure`, or `surgery_time` results in awkward email text or empty placeholders.
  - HTML email rendering quirks (some clients strip styles/buttons); ensure button is implemented as a styled `<a>`.

#### Node: Gmail: Send Pre-Op Checklist Reminder
- **Type / role:** `Gmail` — sends the generated email.
- **Configuration:**
  - Subject: `={{ $json.output.Subject }}`
  - Message (HTML): `={{ $json.output.Body }}`
  - `appendAttribution`: false
  - **Send To** is set to an expression `"="` but appears empty (no actual expression provided).
- **Inputs/outputs:** Receives agent output per patient; sends email.
- **Critical issue to fix:**
  - `sendTo` must be set (likely `={{ $json.patient_email }}`), otherwise sending will fail.
- **Edge cases / failures:**
  - Gmail OAuth scope/consent issues.
  - Sending limits / rate limits.
  - Invalid email addresses extracted from the calendar description.

---

### Block 4 — Patient Confirmation Capture (Webhook → Sheets)
**Overview:**  
When the patient clicks the confirmation button, n8n receives the GET request, extracts query params, marks confirmed, and stores it in Google Sheets.

**Nodes involved:**
- Webhook: Patient Checklist Confirmation (GET /confirm)
- Parse Query Params + Mark Confirmed (patient_id, uniqueId, confirmedAt)
- Google Sheets: Upsert Patient Confirmation Status

#### Node: Webhook: Patient Checklist Confirmation (GET /confirm)
- **Type / role:** `Webhook` — public entry point for confirmation clicks.
- **Configuration:**
  - Path: `confirm`
  - Method: not explicitly shown; typically defaults to GET/POST depending on node settings. The node name implies GET.
- **Inputs/outputs:** Starts confirmation flow → Code node.
- **Edge cases / failures:**
  - If the emailed link uses `/webhook-test/confirm`, it will only work when the workflow is in test mode; use production URL for real usage.
  - Anyone with the link can confirm; no authentication/signature check is implemented.

#### Node: Parse Query Params + Mark Confirmed (patient_id, uniqueId, confirmedAt)
- **Type / role:** `Code` — extracts query parameters and creates confirmation records.
- **Configuration/logic:**
  - Reads `item.json.query.patient_id` and `item.json.query.uniqueId`.
  - Cleans by removing `' procedure'` if present.
  - Outputs:
    - `patient_id`, `uniqueId`, `confirmed: true`, `confirmedAt` timestamp.
  - Mentions “in-memory store” but actually just creates an array within execution; it does **not** persist in memory across runs.
- **Edge cases / failures:**
  - Missing query params → null patient_id/uniqueId → upsert may create bad rows.
  - No validation that `uniqueId` matches one originally sent.

#### Node: Google Sheets: Upsert Patient Confirmation Status
- **Type / role:** `Google Sheets` — append-or-update by patient_id.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Document: `sample_leads_50` (Google Sheet ID `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`)
  - Sheet/tab: `patient data` (gid `208642893`)
  - Matching column: `patient_id`
  - Mapped columns include:
    - `patient_id` ← `{{ $json.patient_id }}`
    - `confirmed` ← `{{ $json.confirmed }}`
    - `confirmed_at` ← `{{ $json.confirmed }}` (**bug**: should likely be `{{ $json.confirmedAt }}`)
- **Edge cases / failures:**
  - If `confirmed` is stored as string (e.g., `"true"`), downstream boolean checks may fail.
  - Permissions or sheet structure mismatch (missing columns).
  - Upsert “match by patient_id” assumes patient_id is stable and unique.

---

### Block 5 — Periodic Confirmation Check & Slack Escalation
**Overview:**  
Runs hourly, reads all patient confirmation rows from Sheets, routes confirmed vs not confirmed, and alerts Slack for those not confirmed.

**Nodes involved:**
- Schedule Trigger: Periodic Confirmation Check
- Google Sheets: Fetch Patient Confirmation Rows
- IF: Confirmed = true (route unmatched = not confirmed)
- Slack: Alert Nurse/Owner — Checklist Not Confirmed

#### Node: Schedule Trigger: Periodic Confirmation Check
- **Type / role:** `Schedule Trigger` — recurring entry point.
- **Configuration:** Interval: every `hours` (defaults to every 1 hour).
- **Outputs:** Triggers Sheets fetch.
- **Edge cases:**
  - Frequency might be too high/low; no “4-hour window” logic is enforced in workflow—only “confirmed true vs not”.

#### Node: Google Sheets: Fetch Patient Confirmation Rows
- **Type / role:** `Google Sheets` — reads rows from the tracking tab.
- **Configuration:** Reads from same document/sheet as the upsert node.
- **Outputs:** Row items to IF node.
- **Edge cases:**
  - Large sheet sizes can slow executions.
  - Empty sheet or missing columns.

#### Node: IF: Confirmed = true (route unmatched = not confirmed)
- **Type / role:** `IF` — branching filter.
- **Configuration:**
  - Condition: `{{ $json.confirmed }}` **boolean equals** `true`
  - Strict type validation enabled.
  - “Unmatched” path is used for “not confirmed”.
- **Outputs:**
  - **True branch:** no downstream node connected.
  - **False/unmatched branch:** goes to Slack alert node.
- **Edge cases / failures:**
  - If Sheets stores `confirmed` as `"TRUE"`, `"true"`, or `1`, strict boolean comparison fails and will incorrectly treat rows as unconfirmed.

#### Node: Slack: Alert Nurse/Owner — Checklist Not Confirmed
- **Type / role:** `Slack` — sends escalation message to a user.
- **Configuration:**
  - Sends a formatted message containing patient identity, procedure, surgery time, contact details, calendar link.
  - Targets Slack user ID `U09HMPVD466` (select: user).
  - Credential: “Slack account vivek”.
- **Edge cases / failures:**
  - Slack token scopes missing (`chat:write`, user targeting permissions).
  - Patient fields may be blank if not stored in sheet (current upsert only writes confirmed fields; it does not write patient_name/email/phone unless already present in that matched row).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger: Daily 9:00 AM | n8n-nodes-base.scheduleTrigger | Daily entry point for reminder flow | — | Google Calendar: Fetch Today’s Events | ## 📅 Daily Surgery Intake & Patient Extraction; Fetches today’s calendar events and filters only surgery-related entries. • Pulls events from Google Calendar • Extracts patient details from description • Normalizes data for downstream steps |
| Google Calendar: Fetch Today’s Events | n8n-nodes-base.googleCalendar | Fetch calendar events | Schedule Trigger: Daily 9:00 AM | Extract Surgery Events + Patient Fields | ## 📅 Daily Surgery Intake & Patient Extraction; Fetches today’s calendar events and filters only surgery-related entries. • Pulls events from Google Calendar • Extracts patient details from description • Normalizes data for downstream steps |
| Extract Surgery Events + Patient Fields | n8n-nodes-base.code | Filter surgery events + extract patient fields | Google Calendar: Fetch Today’s Events | Build Unique Confirmation Link (confirmUrl) | ## 📅 Daily Surgery Intake & Patient Extraction; Fetches today’s calendar events and filters only surgery-related entries. • Pulls events from Google Calendar • Extracts patient details from description • Normalizes data for downstream steps |
| Build Unique Confirmation Link (confirmUrl) | n8n-nodes-base.code | Generate uniqueId + confirmUrl | Extract Surgery Events + Patient Fields | Clean Patient Fields (Name/Email/Phone/ID) | ## 🔗 Build Unique Patient Confirmation Links; Creates a unique, trackable confirmation URL per patient. • Generates secure confirmUrl • Adds timestamps + IDs • Enables click tracking per patient |
| Clean Patient Fields (Name/Email/Phone/ID) | n8n-nodes-base.code | Cleanup extracted strings | Build Unique Confirmation Link (confirmUrl) | AI: Generate Pre-Op Checklist Email (Subject + HTML Body) | ## 🔗 Build Unique Patient Confirmation Links; Creates a unique, trackable confirmation URL per patient. • Generates secure confirmUrl • Adds timestamps + IDs • Enables click tracking per patient |
| AI: Generate Pre-Op Checklist Email (Subject + HTML Body) | @n8n/n8n-nodes-langchain.agent | Create subject + HTML body using AI | Clean Patient Fields (Name/Email/Phone/ID); (LLM + output parser via AI ports) | Gmail: Send Pre-Op Checklist Reminder | ## ✉️ AI Pre-Op Checklist Email Generation; Creates a patient-friendly checklist email using real surgery details. • Personalized subject + HTML body • Includes confirmation button • Safe, non-clinical language |
| LLM: Azure OpenAI Chat Model (gpt-4o) | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | LLM provider for the agent | — | AI: Generate Pre-Op Checklist Email (Subject + HTML Body) | ## ✉️ AI Pre-Op Checklist Email Generation; Creates a patient-friendly checklist email using real surgery details. • Personalized subject + HTML body • Includes confirmation button • Safe, non-clinical language |
| Parse AI Output (Subject/Body JSON) | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON output | — | AI: Generate Pre-Op Checklist Email (Subject + HTML Body) | ## ✉️ AI Pre-Op Checklist Email Generation; Creates a patient-friendly checklist email using real surgery details. • Personalized subject + HTML body • Includes confirmation button • Safe, non-clinical language |
| Gmail: Send Pre-Op Checklist Reminder | n8n-nodes-base.gmail | Send generated email to patient | AI: Generate Pre-Op Checklist Email (Subject + HTML Body) | — | ## 📤 Send Checklist Reminder (Gmail); Sends the pre-op checklist email to the patient. • Uses Gmail for delivery • Supports styled HTML emails • Human-readable patient message |
| Webhook: Patient Checklist Confirmation (GET /confirm) | n8n-nodes-base.webhook | Entry point for confirmation clicks | — | Parse Query Params + Mark Confirmed (patient_id, uniqueId, confirmedAt) | ## ✅ Patient Confirmation Capture (Webhook); Receives confirmation when the patient clicks the email button. • Parses patient_id + uniqueId • Marks checklist as confirmed • Feeds data into tracking sheet |
| Parse Query Params + Mark Confirmed (patient_id, uniqueId, confirmedAt) | n8n-nodes-base.code | Parse query params and output confirmation record | Webhook: Patient Checklist Confirmation (GET /confirm) | Google Sheets: Upsert Patient Confirmation Status | ## ✅ Patient Confirmation Capture (Webhook); Receives confirmation when the patient clicks the email button. • Parses patient_id + uniqueId • Marks checklist as confirmed • Feeds data into tracking sheet |
| Google Sheets: Upsert Patient Confirmation Status | n8n-nodes-base.googleSheets | Append/update confirmation status | Parse Query Params + Mark Confirmed (patient_id, uniqueId, confirmedAt) | — | ## 🧾 Store Confirmation Status (Google Sheets); Stores and updates patient confirmation records for audit and ops visibility. • Upserts confirmation status • Tracks confirmed_at timestamp • Acts as source of truth |
| Schedule Trigger: Periodic Confirmation Check | n8n-nodes-base.scheduleTrigger | Hourly scan trigger | — | Google Sheets: Fetch Patient Confirmation Rows | ## ⏳ Periodic Confirmation Check; Periodically scans patients who have not confirmed on time. • Reads patient rows from Sheets • Filters confirmed vs not confirmed • Flags risky cases for follow-up |
| Google Sheets: Fetch Patient Confirmation Rows | n8n-nodes-base.googleSheets | Read tracking sheet rows | Schedule Trigger: Periodic Confirmation Check | IF: Confirmed = true (route unmatched = not confirmed) | ## ⏳ Periodic Confirmation Check; Periodically scans patients who have not confirmed on time. • Reads patient rows from Sheets • Filters confirmed vs not confirmed • Flags risky cases for follow-up |
| IF: Confirmed = true (route unmatched = not confirmed) | n8n-nodes-base.if | Route confirmed vs not confirmed | Google Sheets: Fetch Patient Confirmation Rows | (true branch none), Slack: Alert Nurse/Owner — Checklist Not Confirmed (false/unmatched) | ## ⏳ Periodic Confirmation Check; Periodically scans patients who have not confirmed on time. • Reads patient rows from Sheets • Filters confirmed vs not confirmed • Flags risky cases for follow-up |
| Slack: Alert Nurse/Owner — Checklist Not Confirmed | n8n-nodes-base.slack | Escalation alert to staff | IF: Confirmed = true (route unmatched = not confirmed) | — | ## 🚨 Alert Care Team for Missing Confirmations; Notifies nurse or owner when a patient has not confirmed the checklist. • Sends full patient context to Slack • Highlights surgery time + procedure • Enables fast manual follow-up |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🏥 AI-Powered Patient Pre-Op Reminder & Nurse Alert System; (contains overview, setup checklist, customization ideas) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 📅 Daily Surgery Intake & Patient Extraction; Fetches today’s calendar events and filters only surgery-related entries. • Pulls events from Google Calendar • Extracts patient details from description • Normalizes data for downstream steps |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🔗 Build Unique Patient Confirmation Links; Creates a unique, trackable confirmation URL per patient. • Generates secure confirmUrl • Adds timestamps + IDs • Enables click tracking per patient |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## ✉️ AI Pre-Op Checklist Email Generation; Creates a patient-friendly checklist email using real surgery details. • Personalized subject + HTML body • Includes confirmation button • Safe, non-clinical language |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 📤 Send Checklist Reminder (Gmail); Sends the pre-op checklist email to the patient. • Uses Gmail for delivery • Supports styled HTML emails • Human-readable patient message |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## ✅ Patient Confirmation Capture (Webhook); Receives confirmation when the patient clicks the email button. • Parses patient_id + uniqueId • Marks checklist as confirmed • Feeds data into tracking sheet |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🧾 Store Confirmation Status (Google Sheets); Stores and updates patient confirmation records for audit and ops visibility. • Upserts confirmation status • Tracks confirmed_at timestamp • Acts as source of truth |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## ⏳ Periodic Confirmation Check; Periodically scans patients who have not confirmed on time. • Reads patient rows from Sheets • Filters confirmed vs not confirmed • Flags risky cases for follow-up |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🚨 Alert Care Team for Missing Confirmations; Notifies nurse or owner when a patient has not confirmed the checklist. • Sends full patient context to Slack • Highlights surgery time + procedure • Enables fast manual follow-up |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 🔐 Credentials & Safety Notes; • Google Calendar OAuth • Gmail OAuth2 • Azure OpenAI API • Google Sheets OAuth • Slack API; Do not include medical advice beyond basic reminders. Always keep human follow-up in the loop for non-confirmed patients. |

---

## 4. Reproducing the Workflow from Scratch

### A) Create the daily reminder flow (Calendar → AI → Gmail)
1. **Add node:** *Schedule Trigger*  
   - Name: `Schedule Trigger: Daily 9:00 AM`  
   - Rule: Cron → `0 9 * * *` (set timezone in n8n settings as needed)

2. **Add node:** *Google Calendar*  
   - Name: `Google Calendar: Fetch Today’s Events`  
   - Operation: **Get All**  
   - Calendar: select the target calendar (e.g., the surgeon/clinic calendar)  
   - Credentials: connect **Google Calendar OAuth2**  
   - (Recommended) In **Options**, set a “today” window (timeMin/timeMax) to avoid pulling irrelevant dates.

3. **Connect:** Schedule Trigger → Google Calendar

4. **Add node:** *Code*  
   - Name: `Extract Surgery Events + Patient Fields`  
   - Paste logic that:
     - filters events by `summary` containing “surgery”
     - strips HTML from description
     - extracts `patient_name`, `patient_email`, `patient_phone`, `patient_id`, `procedure`
     - outputs `event_id`, `surgery_time`, `calendar_link`

5. **Connect:** Google Calendar → Extract Surgery Events

6. **Add node:** *Code*  
   - Name: `Build Unique Confirmation Link (confirmUrl)`  
   - Generate:
     - `uniqueId` using patient_id + timestamp
     - `confirmUrl` pointing to your n8n webhook URL  
       - Use **production** URL format (typical):  
         `https://<your-n8n-domain>/webhook/confirm?patient_id=...&uniqueId=...`
       - Only use `/webhook-test/` for testing.

7. **Connect:** Extract Surgery Events → Build Unique Confirmation Link

8. **Add node:** *Code*  
   - Name: `Clean Patient Fields (Name/Email/Phone/ID)`  
   - Apply field cleanup (or replace with more robust parsing if you control the calendar description format).

9. **Connect:** Build Unique Confirmation Link → Clean Patient Fields

10. **Add node:** *AI Agent* (LangChain Agent)  
    - Name: `AI: Generate Pre-Op Checklist Email (Subject + HTML Body)`  
    - Prompt: include `{{ JSON.stringify($json, null, 2) }}` and require:
      - JSON output with exactly `Subject` and `Body`
      - HTML body with confirmation button linking to `confirmUrl`
    - System message: enforce “no invented details” and non-clinical reminders only.

11. **Add node:** *Azure OpenAI Chat Model*  
    - Name: `LLM: Azure OpenAI Chat Model (gpt-4o)`  
    - Model: `gpt-4o`  
    - Credentials: connect **Azure OpenAI** (endpoint, key, and deployment/model mapping as required by n8n)

12. **Add node:** *Structured Output Parser*  
    - Name: `Parse AI Output (Subject/Body JSON)`  
    - Provide schema example with keys `Subject` and `Body`.

13. **Connect AI ports:**
    - LLM node → Agent node via **ai_languageModel**
    - Output Parser node → Agent node via **ai_outputParser**

14. **Connect main flow:** Clean Patient Fields → AI Agent

15. **Add node:** *Gmail*  
    - Name: `Gmail: Send Pre-Op Checklist Reminder`  
    - Operation: Send  
    - **To:** set to `={{ $json.patient_email }}` (this is required; the provided workflow has this misconfigured)  
    - **Subject:** `={{ $json.output.Subject }}`  
    - **Message (HTML):** `={{ $json.output.Body }}`  
    - Credentials: connect **Gmail OAuth2**

16. **Connect:** AI Agent → Gmail

---

### B) Create the confirmation capture flow (Webhook → Sheets)
17. **Add node:** *Webhook*  
    - Name: `Webhook: Patient Checklist Confirmation (GET /confirm)`  
    - Path: `confirm`  
    - Method: GET (or allow GET)  
    - Activate workflow to obtain the production webhook URL.

18. **Add node:** *Code*  
    - Name: `Parse Query Params + Mark Confirmed (patient_id, uniqueId, confirmedAt)`  
    - Read query params `patient_id` and `uniqueId` and output:
      - `patient_id`, `uniqueId`, `confirmed: true`, `confirmedAt` (ISO timestamp)

19. **Connect:** Webhook → Parse Query Params code node

20. **Add node:** *Google Sheets*  
    - Name: `Google Sheets: Upsert Patient Confirmation Status`  
    - Operation: **Append or Update**  
    - Spreadsheet: select your tracking sheet  
    - Tab: e.g., `patient data`  
    - Matching column: `patient_id`  
    - Map fields:
      - `patient_id` ← `={{ $json.patient_id }}`
      - `uniqueId` ← `={{ $json.uniqueId }}` (recommended)
      - `confirmed` ← `={{ $json.confirmed }}`
      - `confirmed_at` ← `={{ $json.confirmedAt }}` (**recommended fix**)
    - Credentials: connect **Google Sheets OAuth2**

21. **Connect:** Parse Query Params → Google Sheets Upsert

---

### C) Create the periodic escalation flow (Sheets → IF → Slack)
22. **Add node:** *Schedule Trigger*  
    - Name: `Schedule Trigger: Periodic Confirmation Check`  
    - Interval: every 1 hour (or your choice)

23. **Add node:** *Google Sheets*  
    - Name: `Google Sheets: Fetch Patient Confirmation Rows`  
    - Operation: Read/Get Many (default “read rows” behavior)  
    - Same document + tab as above

24. **Connect:** Periodic Schedule Trigger → Fetch Patient Confirmation Rows

25. **Add node:** *IF*  
    - Name: `IF: Confirmed = true (route unmatched = not confirmed)`  
    - Condition: `{{$json.confirmed}}` equals boolean `true`
    - If your sheet stores strings, either:
      - convert to boolean upstream, or
      - change condition to string compare (`"true"`), or
      - disable strict type validation.

26. **Connect:** Fetch Patient Confirmation Rows → IF

27. **Add node:** *Slack*  
    - Name: `Slack: Alert Nurse/Owner — Checklist Not Confirmed`  
    - Target: user (or channel)  
    - Message: include patient_name/id/procedure/surgery_time/contact/calendar_link  
    - Credentials: connect **Slack API** with `chat:write` permissions

28. **Connect:** IF “false/unmatched” output → Slack node  
    - Leave “true” output unconnected (or add logging).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates pre-surgery checklist reminders, captures confirmations, stores status in Sheets, and alerts staff on Slack for non-confirmations. | From sticky note “🏥 AI-Powered Patient Pre-Op Reminder & Nurse Alert System” |
| Setup checklist: Google Calendar OAuth; Azure OpenAI credentials; Gmail OAuth2; Google Sheets; Slack API | Same sticky note |
| Safety: Do not include medical advice beyond basic reminders; keep human follow-up in the loop for non-confirmed patients. | From “🔐 Credentials & Safety Notes” sticky note |
| Customization ideas: change reminder cadence; add SMS reminders; add nurse assignment logic per procedure | From main overview sticky note |

### Notable implementation gaps to address (recommended)
- **Gmail sendTo is not set** → set `sendTo = {{$json.patient_email}}`.
- **confirmUrl uses `/webhook-test/`** → switch to production `/webhook/` for real patient use.
- **Sheets mapping bug:** `confirmed_at` currently maps to `confirmed` instead of `confirmedAt`.
- **No “4-hour window” logic is enforced** in the periodic check; it only checks confirmed vs not. Add time comparison using `createdAt`/`surgery_time` and a threshold.
- **Boolean strict IF may misroute** if Sheets stores `confirmed` as text; normalize types before IF.