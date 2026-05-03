Sync AI-enriched TimeRex bookings to Google Sheets and Slack with Gemini

https://n8nworkflows.xyz/workflows/sync-ai-enriched-timerex-bookings-to-google-sheets-and-slack-with-gemini-12063


# Sync AI-enriched TimeRex bookings to Google Sheets and Slack with Gemini

Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync AI-enriched TimeRex bookings to Google Sheets and Slack with Gemini  
**Internal name:** TimeRex AI-Powered Booking Automation

**Purpose:**  
Receive TimeRex booking webhooks, verify a security token, filter relevant calendars, then:
- For **confirmed bookings**: enrich booking data (media source, inferred company), classify the booking and generate a brief using **Google Gemini**, append the enriched record to **Google Sheets**, and post a **Slack** notification.
- For **cancelled bookings**: locate the booking row in Google Sheets by `event_id`, delete it, and notify Slack.

**Primary use cases:**
- Centralizing scheduling events into a “Bookings” sheet with consistent columns.
- Providing hosts/team with an AI-generated “meeting prep brief” directly in Slack.
- Keeping the sheet clean by removing cancelled events.
- Rejecting unauthorized webhook calls and notifying security on Slack.

### 1.1 Trigger & Security Layer
Webhook intake → token verification → calendar filtering → event type routing.

### 1.2 Booking Flow — AI Enhancement Pipeline (event_confirmed)
Media detection (from “Media Master” sheet) → company inference → Gemini categorization → Gemini brief → merge → append to “Bookings” sheet → Slack alert.

### 1.3 Cancellation Flow (event_cancelled)
Find row by Event ID → delete row → Slack cancellation alert.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Security Layer
**Overview:** Receives TimeRex webhook events, rejects unauthorized calls, optionally filters only certain calendars, then routes to confirmation vs cancellation processing.

**Nodes involved:**
- TimeRex Webhook
- Verify Security Token
- Slack: Security Alert
- Filter by Calendar Type
- Route by Event Type

#### Node: TimeRex Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point HTTP endpoint.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: `/timerex-booking`
  - Produces payload under `{{$json.body}}` and headers under `{{$json.headers}}`.
- **Connections:**
  - Output → **Verify Security Token**
- **Edge cases / failures:**
  - TimeRex sending unexpected schema (missing `body.event`, etc.) can break downstream expressions.
  - If TimeRex doesn’t include `x-timerex-authorization`, token check will fail (expected behavior).

#### Node: Verify Security Token
- **Type / role:** `n8n-nodes-base.if` — authorization gate.
- **Configuration:**
  - Condition checks header equality:  
    `{{ $json.headers['x-timerex-authorization'] }}` equals `YOUR_TIMEREX_SECURITY_TOKEN`
- **Connections:**
  - **True** → Filter by Calendar Type
  - **False** → Slack: Security Alert
- **Version-specific:** IF node v2.2 uses the “conditions (version 2)” model.
- **Edge cases / failures:**
  - Header key casing differences can cause false negatives (depends on n8n’s normalized headers; verify in executions).
  - If token is hardcoded, rotating secrets requires workflow edit (consider env vars).

#### Node: Slack: Security Alert
- **Type / role:** `n8n-nodes-base.slack` — sends alert when token verification fails.
- **Configuration:**
  - Auth: **OAuth2**
  - Channel: selected via `channelId`
  - Message includes timestamp: `{{ $now.format('yyyy-MM-dd HH:mm:ss') }}`
- **Connections:** none (terminal on failure branch)
- **Edge cases / failures:**
  - Slack OAuth scopes missing (e.g., `chat:write`) → auth errors.
  - Channel not selected → message send failure.

#### Node: Filter by Calendar Type
- **Type / role:** `n8n-nodes-base.switch` — filters out calendars not relevant to this automation.
- **Configuration:**
  - Rule: `{{ $json.body.calendar_name }}` **contains** `YOUR_CALENDAR_FILTER_KEYWORD`
  - Only matching items continue.
- **Connections:**
  - Matched output (rule #1) → Route by Event Type
- **Edge cases / failures:**
  - If `body.calendar_name` is missing/null, rule won’t match and the flow ends silently (no “else” branch configured).
  - Keyword is placeholder; must be updated or nothing will pass.

#### Node: Route by Event Type
- **Type / role:** `n8n-nodes-base.switch` — routes confirmed vs cancelled.
- **Configuration:**
  - If `{{ $json.body.webhook_type }}` equals `event_confirmed` → Booking flow
  - If equals `event_cancelled` → Cancellation flow
- **Connections:**
  - Confirmed → Get Media Master
  - Cancelled → Find Booking by Event ID
- **Edge cases / failures:**
  - Any other webhook type will not route anywhere (silent stop).
  - If TimeRex uses different event names, update these values.

---

### Block 2 — Booking Flow: Media Detection + Data Extraction
**Overview:** Pulls a list of media sources from a “Media Master” sheet and tries to detect which media name appears inside the TimeRex `calendar_name`. Then extracts guest/host/meeting basics and infers company from email domain.

**Nodes involved:**
- Get Media Master
- Detect Media Source
- Extract Company from Email

#### Node: Get Media Master
- **Type / role:** `n8n-nodes-base.googleSheets` — reads the media master list.
- **Configuration:**
  - Document ID: `YOUR_MEDIA_MASTER_SHEET_ID` (placeholder)
  - Sheet tab: “Media Master” (gid=0 in template)
  - Operation not explicitly set in JSON snippet; in n8n Sheets v4.x this typically defaults to **Read/Get All** depending on node UI. The downstream code assumes it outputs rows containing a `media_name` column.
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
- **Connections:**
  - Output → Detect Media Source
- **Edge cases / failures:**
  - Wrong sheet ID / tab → 404 or “sheet not found”.
  - Missing `media_name` column → media detection always fails (returns empty).

#### Node: Detect Media Source
- **Type / role:** `n8n-nodes-base.code` — matches media source based on calendar name.
- **Configuration (logic):**
  - Reads `calendar_name` from: `$('TimeRex Webhook').first().json.body.calendar_name`
  - Builds `mediaList` from incoming sheet items: `item.json["media_name"]`
  - Finds first `media` where `calendarName.includes(media)`
  - Outputs object: `{ calendar_name: matchedMedia ?? "" }`
- **Connections:**
  - Output → Extract Company from Email
- **Key variables / expressions:**
  - Uses `$('TimeRex Webhook')` to reference trigger data.
  - Uses `$items()` from current input (Google Sheets rows).
- **Edge cases / failures:**
  - Output key name is `calendar_name` but semantically it is the **matched media**. This is confusing and later treated as media source via `Detect Media Source`. Consider renaming output key to `media_source`.
  - If multiple media names match, only first match is returned (order depends on sheet row order).
  - If Google Sheets returns no items, `$items()` is empty and match returns `""`.

#### Node: Extract Company from Email
- **Type / role:** `n8n-nodes-base.code` — normalizes booking data and infers company.
- **Configuration (logic):**
  - Reads guest email and name from `body.event.form[]` by `field_type`:
    - `guest_email`, `guest_name`
  - Reads:
    - `calendarName` from webhook `body.calendar_name`
    - `mediaSource` from Detect Media Source output `json.calendar_name` (see naming caveat above)
    - `booking_date` from `body.event.local_start_datetime`
    - `host_name` from `body.event.hosts[0].name`
    - `meeting_url` from `body.event.google_meet_meeting.join_url`
    - `event_id` from `body.event.id`
  - Infers `company_name`:
    - Extracts domain part of email
    - If domain is in common free providers → “Individual”
    - Else uses first label of domain (before first dot) and capitalizes
- **Connections:**
  - Output → AI: Categorize Booking
- **Edge cases / failures:**
  - If TimeRex form field types differ, `find()` returns undefined and values become empty strings.
  - `hosts[0]` may be missing → outputs empty host.
  - `google_meet_meeting` may not exist (non-Google meeting) → meeting_url empty.
  - Domain parsing is simplistic (e.g., `co.uk` domains, subdomains, or brand domains like `mail.company.com`).

---

### Block 3 — Booking Flow: AI Enrichment with Google Gemini
**Overview:** Uses Gemini chat models via LangChain nodes to (1) categorize the booking and (2) generate a short host-facing brief, then merges outputs.

**Nodes involved:**
- Google Gemini (Categorize)
- AI: Categorize Booking
- Google Gemini (Brief)
- AI: Generate Meeting Brief
- Merge AI Results

#### Node: Google Gemini (Categorize)
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM “languageModel” to the chain node.
- **Configuration:**
  - Uses default options (model selection happens in credentials or node defaults depending on setup).
- **Connections:**
  - `ai_languageModel` output → AI: Categorize Booking (as language model input)
- **Edge cases / failures:**
  - Missing/invalid Gemini credentials → runtime auth error.
  - Model rate limits or safety blocks can cause failed executions.

#### Node: AI: Categorize Booking
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — prompt-based classification.
- **Configuration:**
  - Prompt instructs: output only one category name from a fixed list.
  - Uses fields from current JSON (`$json.guest_name`, etc.) supplied by Extract Company from Email.
- **Connections:**
  - Main output → AI: Generate Meeting Brief
  - Language model input comes from **Google Gemini (Categorize)** via `ai_languageModel`.
- **Outputs:**
  - The category is expected in `{{$json.text}}` (used later).
- **Edge cases / failures:**
  - Model may return extra text (violating “only category”); downstream trims but does not validate membership.
  - If the LLM node errors, the booking won’t be appended and no Slack notification occurs.

#### Node: Google Gemini (Brief)
- **Type / role:** `lmChatGoogleGemini` — second Gemini model provider for brief generation.
- **Connections:**
  - `ai_languageModel` → AI: Generate Meeting Brief
- **Edge cases / failures:** same as Gemini (Categorize).

#### Node: AI: Generate Meeting Brief
- **Type / role:** `chainLlm` — generates short prep brief.
- **Configuration:**
  - Pulls booking details using explicit node references:
    - From `Extract Company from Email`
    - Category from `AI: Categorize Booking`.json.text
  - Constrains format and requests English only, under 150 characters (LLMs may not strictly comply).
- **Connections:**
  - Main output → Merge AI Results
  - Language model input from **Google Gemini (Brief)**.
- **Edge cases / failures:**
  - If category output is missing, prompt includes blank category.
  - Length constraint not enforced programmatically.

#### Node: Merge AI Results
- **Type / role:** `n8n-nodes-base.code` — merges structured booking data with AI outputs.
- **Configuration (logic):**
  - `extractedData` from Extract Company from Email
  - `category` from AI: Categorize Booking `json.text` default “Other”
  - `meetingBrief` from AI: Generate Meeting Brief `json.text` default `""`
  - Adds `created_at` = ISO timestamp
- **Connections:**
  - Output → Append Enriched Booking
- **Edge cases / failures:**
  - If upstream nodes output multiple items, `.first()` usage forces single-item semantics (might ignore additional items).

---

### Block 4 — Booking Flow: Persistence (Sheets) + Notification (Slack)
**Overview:** Appends the enriched booking record to a “Bookings” sheet and posts a Slack notification including AI brief.

**Nodes involved:**
- Append Enriched Booking
- Slack: New Booking Alert

#### Node: Append Enriched Booking
- **Type / role:** `n8n-nodes-base.googleSheets` — append row.
- **Configuration:**
  - Operation: **Append**
  - Document ID: `YOUR_BOOKINGS_SHEET_ID` (placeholder)
  - Sheet tab: “Bookings” (gid=0 in template)
  - Maps fields:
    - `event_id, host_name, created_at, guest_name, guest_email, meeting_url, booking_date, company_name, media_source, calendar_name, ai_meeting_brief, booking_category`
  - No type conversion attempted (`attemptToConvertTypes: false`)
- **Connections:**
  - Output → Slack: New Booking Alert
- **Edge cases / failures:**
  - If sheet columns differ from expected headers, append may create misaligned data or fail depending on configuration.
  - Duplicate `event_id` not prevented (no upsert/dedup logic).
  - API quota / permission issues.

#### Node: Slack: New Booking Alert
- **Type / role:** `n8n-nodes-base.slack` — posts enriched booking summary.
- **Configuration:**
  - Auth: OAuth2
  - Message includes:
    - Guest, company, email, date, host, category, media source (defaults to “Direct” if empty)
    - AI brief
    - Meeting URL link formatted as `<url|Join Meeting>`
- **Connections:** none (terminal)
- **Edge cases / failures:**
  - Missing `meeting_url` can produce an empty Slack link target.
  - Slack markdown rendering issues if AI output contains special characters.

---

### Block 5 — Cancellation Flow
**Overview:** On cancellation events, looks up the row in Google Sheets by `event_id`, deletes that row, and posts a cancellation message to Slack.

**Nodes involved:**
- Find Booking by Event ID
- Delete Cancelled Booking
- Slack: Cancellation Alert

#### Node: Find Booking by Event ID
- **Type / role:** `n8n-nodes-base.googleSheets` — lookup rows in “Bookings”.
- **Configuration:**
  - Filters: `event_id` equals `{{ $node['TimeRex Webhook'].json.body.event.id }}`
  - Document ID: `YOUR_BOOKINGS_SHEET_ID`
  - Sheet: “Bookings”
- **Connections:**
  - Output → Delete Cancelled Booking
- **Edge cases / failures:**
  - If no row found, downstream `Delete` may fail due to missing `row_number`.
  - If multiple rows match (duplicates), behavior depends on Sheets node output (may return multiple items and delete multiple rows in subsequent node, but current delete config appears single-row oriented).

#### Node: Delete Cancelled Booking
- **Type / role:** `n8n-nodes-base.googleSheets` — deletes row(s) by index.
- **Configuration:**
  - Operation: **delete**
  - `startIndex`: `{{ $json.row_number }}`
  - `numberToDelete`: `=1` (intended as 1 row)
- **Connections:**
  - Output → Slack: Cancellation Alert
- **Edge cases / failures:**
  - `row_number` indexing can be off-by-one depending on node semantics (header row vs data row). Validate in your environment.
  - If `row_number` is undefined → expression failure.
  - If multiple items come in, this node may run per-item and delete multiple rows.

#### Node: Slack: Cancellation Alert
- **Type / role:** `n8n-nodes-base.slack` — notifies about cancellation.
- **Configuration:**
  - Message pulls from webhook form fields:
    - Guest name/email via `.find(f => f.field_type === 'guest_name'/'guest_email')`
  - Includes original datetime, host, and event ID.
- **Connections:** none (terminal)
- **Edge cases / failures:**
  - If form fields absent → message shows blanks.
  - Slack auth/channel issues as above.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| TimeRex Webhook | n8n-nodes-base.webhook | Receives TimeRex POST webhook events | — | Verify Security Token | ## 🔐 Trigger & Security Layer<br>Webhook Endpoint receives POST from TimeRex… Event routing confirmed/cancelled |
| Verify Security Token | n8n-nodes-base.if | Validates `x-timerex-authorization` header | TimeRex Webhook | Filter by Calendar Type (true); Slack: Security Alert (false) | ## 🔐 Trigger & Security Layer<br>Validates header token; failed attempts trigger Slack alert |
| Slack: Security Alert | n8n-nodes-base.slack | Sends Slack alert on invalid token | Verify Security Token (false) | — | ## 🔐 Trigger & Security Layer<br>Failed attempts trigger an immediate Slack security alert |
| Filter by Calendar Type | n8n-nodes-base.switch | Filters relevant calendars by keyword | Verify Security Token (true) | Route by Event Type | ## 🔐 Trigger & Security Layer<br>Routes only relevant calendar types… customize filter |
| Route by Event Type | n8n-nodes-base.switch | Routes confirmed vs cancelled | Filter by Calendar Type | Get Media Master (confirmed); Find Booking by Event ID (cancelled) | ## 🔐 Trigger & Security Layer<br>`event_confirmed` → Booking flow; `event_cancelled` → Cancellation flow |
| Get Media Master | n8n-nodes-base.googleSheets | Reads media names list from “Media Master” sheet | Route by Event Type (confirmed) | Detect Media Source | ## 📅 Booking Flow — AI Enhancement Pipeline<br>Media Detection → Company Extraction → AI Categorization… |
| Detect Media Source | n8n-nodes-base.code | Matches media source from calendar name | Get Media Master | Extract Company from Email | ## 📅 Booking Flow — AI Enhancement Pipeline<br>Media Detection → Company Extraction → AI Categorization… |
| Extract Company from Email | n8n-nodes-base.code | Extracts guest/host details; infers company | Detect Media Source | AI: Categorize Booking | ## 📅 Booking Flow — AI Enhancement Pipeline<br>Media Detection → Company Extraction → AI Categorization… |
| Google Gemini (Categorize) | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for categorization | — | AI: Categorize Booking (ai_languageModel) | ## 📅 Booking Flow — AI Enhancement Pipeline<br>AI Processing: Categorize + Brief |
| AI: Categorize Booking | @n8n/n8n-nodes-langchain.chainLlm | Classifies booking into category | Extract Company from Email + Gemini (ai_languageModel) | AI: Generate Meeting Brief | ## 📅 Booking Flow — AI Enhancement Pipeline<br>AI Processing: Categorize + Brief |
| Google Gemini (Brief) | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for brief generation | — | AI: Generate Meeting Brief (ai_languageModel) | ## 📅 Booking Flow — AI Enhancement Pipeline<br>AI Processing: Categorize + Brief |
| AI: Generate Meeting Brief | @n8n/n8n-nodes-langchain.chainLlm | Produces short host prep brief | AI: Categorize Booking + Gemini (ai_languageModel) | Merge AI Results | ## 📅 Booking Flow — AI Enhancement Pipeline<br>AI Processing: Categorize + Brief |
| Merge AI Results | n8n-nodes-base.code | Combines extracted + AI data; adds created_at | AI: Generate Meeting Brief | Append Enriched Booking | ## 📅 Booking Flow — AI Enhancement Pipeline<br>… → Data Merge → Sheet Append → Slack Alert |
| Append Enriched Booking | n8n-nodes-base.googleSheets | Appends enriched row to Bookings sheet | Merge AI Results | Slack: New Booking Alert | ## 📅 Booking Flow — AI Enhancement Pipeline<br>Output: Enriched booking record in Sheets |
| Slack: New Booking Alert | n8n-nodes-base.slack | Posts enriched booking + AI brief to Slack | Append Enriched Booking | — | ## 📅 Booking Flow — AI Enhancement Pipeline<br>Output: Slack notification with AI-generated insights |
| Find Booking by Event ID | n8n-nodes-base.googleSheets | Finds row(s) in Bookings by event_id | Route by Event Type (cancelled) | Delete Cancelled Booking | ## ❌ Cancellation Flow<br>Find → Delete → Notify |
| Delete Cancelled Booking | n8n-nodes-base.googleSheets | Deletes the matched booking row | Find Booking by Event ID | Slack: Cancellation Alert | ## ❌ Cancellation Flow<br>Find → Delete → Notify |
| Slack: Cancellation Alert | n8n-nodes-base.slack | Posts cancellation details to Slack | Delete Cancelled Booking | — | ## ❌ Cancellation Flow<br>Keeps booking data clean and team informed |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 🚀 TimeRex AI-Powered Booking Automation<br>Setup checklist + required sheet columns |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 📅 Booking Flow — AI Enhancement Pipeline |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## ❌ Cancellation Flow |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace documentation | — | — | ## 🔐 Trigger & Security Layer |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **TimeRex AI-Powered Booking Automation** (or your preferred name).

2) **Add Webhook trigger**
- Add node: **Webhook**
  - HTTP Method: **POST**
  - Path: `/timerex-booking`
- Copy the **Production URL** and paste it into **TimeRex → Settings → Webhook**.

3) **Add security gate**
- Add node: **IF** named “Verify Security Token”
  - Condition: String equals
  - Left: `{{$json.headers['x-timerex-authorization']}}`
  - Right: your secret token (replace `YOUR_TIMEREX_SECURITY_TOKEN`)
- Connect: **Webhook → Verify Security Token**

4) **Add Slack security alert (failure branch)**
- Add node: **Slack** named “Slack: Security Alert”
  - Authentication: **OAuth2** (connect Slack credentials)
  - Select channel
  - Text: message indicating token verification failed (use your desired format)
- Connect: **Verify Security Token (false) → Slack: Security Alert**

5) **Filter calendars**
- Add node: **Switch** named “Filter by Calendar Type”
  - Rule 1: String contains
  - Value 1: `{{$json.body.calendar_name}}`
  - Contains: `YOUR_CALENDAR_FILTER_KEYWORD` (replace with your keyword)
- Connect: **Verify Security Token (true) → Filter by Calendar Type**

6) **Route by event type**
- Add node: **Switch** named “Route by Event Type”
  - Rule 1: `{{$json.body.webhook_type}}` equals `event_confirmed`
  - Rule 2: `{{$json.body.webhook_type}}` equals `event_cancelled`
- Connect: **Filter by Calendar Type → Route by Event Type**

### Confirmed booking branch (event_confirmed)

7) **Read Media Master sheet**
- Add node: **Google Sheets** named “Get Media Master”
  - Credentials: Google Sheets OAuth2
  - Document ID: your Media Master spreadsheet ID (`YOUR_MEDIA_MASTER_SHEET_ID`)
  - Sheet/tab: “Media Master”
  - Configure it to read all rows (must output a column named `media_name`)
- Connect: **Route by Event Type (confirmed) → Get Media Master**

8) **Detect media source**
- Add node: **Code** named “Detect Media Source”
  - Implement logic: read `body.calendar_name` from the webhook and check if it contains any `media_name` from the sheet rows; output the matched name (or blank).
- Connect: **Get Media Master → Detect Media Source**

9) **Extract booking + infer company**
- Add node: **Code** named “Extract Company from Email”
  - Extract `guest_email` and `guest_name` from `body.event.form[]` by `field_type`
  - Infer `company_name` from email domain (free domains → “Individual”)
  - Gather `event_id`, `booking_date`, `host_name`, `meeting_url`, `calendar_name`
  - Carry forward `media_source` from the previous step
- Connect: **Detect Media Source → Extract Company from Email**

10) **Add Gemini model node for categorization**
- Add node: **Google Gemini Chat Model** (LangChain) named “Google Gemini (Categorize)”
  - Configure Gemini credentials (API key / project as required by your n8n setup)
  - Pick a model if your node UI requires it
- Add node: **LLM Chain** named “AI: Categorize Booking”
  - Prompt: classify into one of: Sales Meeting, Customer Support, Job Interview, Partnership, Media Interview, Other
  - Ensure output is only the category
- Connect:
  - **Extract Company from Email → AI: Categorize Booking**
  - **Google Gemini (Categorize) → AI: Categorize Booking** via the **AI language model** connection

11) **Add Gemini model node for brief**
- Add node: **Google Gemini Chat Model** named “Google Gemini (Brief)”
- Add node: **LLM Chain** named “AI: Generate Meeting Brief”
  - Prompt: produce short brief using guest/company/category/date; English only
- Connect:
  - **AI: Categorize Booking → AI: Generate Meeting Brief**
  - **Google Gemini (Brief) → AI: Generate Meeting Brief** via **AI language model** connection

12) **Merge results**
- Add node: **Code** named “Merge AI Results”
  - Merge extracted booking fields + `booking_category` + `ai_meeting_brief`
  - Add `created_at` ISO timestamp
- Connect: **AI: Generate Meeting Brief → Merge AI Results**

13) **Append to Bookings sheet**
- Add node: **Google Sheets** named “Append Enriched Booking”
  - Operation: **Append**
  - Document ID: your Bookings spreadsheet ID (`YOUR_BOOKINGS_SHEET_ID`)
  - Sheet/tab: “Bookings”
  - Map columns exactly to your header names:
    - `event_id, booking_date, guest_name, guest_email, calendar_name, meeting_url, host_name, media_source, company_name, booking_category, ai_meeting_brief, created_at`
- Connect: **Merge AI Results → Append Enriched Booking**

14) **Slack new booking alert**
- Add node: **Slack** named “Slack: New Booking Alert”
  - OAuth2 credentials
  - Channel selection
  - Message uses mapped fields and includes the AI brief + meeting link
- Connect: **Append Enriched Booking → Slack: New Booking Alert**

### Cancelled booking branch (event_cancelled)

15) **Find booking row by event_id**
- Add node: **Google Sheets** named “Find Booking by Event ID”
  - Document: `YOUR_BOOKINGS_SHEET_ID`
  - Sheet/tab: “Bookings”
  - Filter: `event_id` equals `{{$node['TimeRex Webhook'].json.body.event.id}}`
- Connect: **Route by Event Type (cancelled) → Find Booking by Event ID**

16) **Delete row**
- Add node: **Google Sheets** named “Delete Cancelled Booking”
  - Operation: **Delete**
  - Start index: `{{$json.row_number}}`
  - Number to delete: `1`
- Connect: **Find Booking by Event ID → Delete Cancelled Booking**

17) **Slack cancellation alert**
- Add node: **Slack** named “Slack: Cancellation Alert”
  - Message includes guest name/email (from webhook), original date, host, event_id
- Connect: **Delete Cancelled Booking → Slack: Cancellation Alert**

18) **Finalize**
- Ensure all placeholders are replaced:
  - `YOUR_TIMEREX_SECURITY_TOKEN`
  - `YOUR_CALENDAR_FILTER_KEYWORD`
  - `YOUR_MEDIA_MASTER_SHEET_ID`
  - `YOUR_BOOKINGS_SHEET_ID`
  - Slack channels selected
  - Gemini credentials configured
- Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Copy webhook URL → TimeRex Settings → Webhook” | Setup checklist (Sticky Note) |
| “Set your security token in `Verify Security Token` node” | Setup checklist (Sticky Note) |
| “Update Google Sheet IDs in all Sheets nodes” | Setup checklist (Sticky Note) |
| “Connect your Google Gemini API credentials” | Setup checklist (Sticky Note) |
| “Select your Slack channel in notification nodes” | Setup checklist (Sticky Note) |
| Required Sheet Columns: `event_id, booking_date, guest_name, guest_email, calendar_name, meeting_url, host_name, media_source, company_name, booking_category, ai_meeting_brief, created_at` | Data model expectation (Sticky Note) |
| Booking flow description: “Media Detection → Company Extraction → AI Categorization → AI Brief Generation → Data Merge → Sheet Append → Slack Alert” | Architecture note (Sticky Note1) |
| Cancellation flow description: “Find → Delete → Notify” | Architecture note (Sticky Note2) |
| Security layer description: header token verification and routing | Architecture note (Sticky Note3) |