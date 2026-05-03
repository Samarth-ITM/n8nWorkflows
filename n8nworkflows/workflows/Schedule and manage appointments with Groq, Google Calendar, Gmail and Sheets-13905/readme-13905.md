Schedule and manage appointments with Groq, Google Calendar, Gmail and Sheets

https://n8nworkflows.xyz/workflows/schedule-and-manage-appointments-with-groq--google-calendar--gmail-and-sheets-13905


# Schedule and manage appointments with Groq, Google Calendar, Gmail and Sheets

# 1. Workflow Overview

This workflow implements a dual-purpose appointment management system for BytezTech using Groq, Google Calendar, Gmail, and Google Sheets.

It contains two independent execution paths:

- **AI Chat Flow**: a chat-based scheduling assistant that lets clients book, reschedule, cancel, or inquire about meetings. It uses a Groq-hosted LLaMA model, memory, Google Calendar tool nodes, and Gmail tool nodes.
- **Daily Sheet Sync Flow**: a scheduled reporting process that runs every day at 9:30 AM IST, clears prior sheet rows, pulls the current day’s Google Calendar events, formats them for tabular storage, inserts them into Google Sheets, and emails an admin summary.

## 1.1 AI Chat Flow

This block starts from a chat trigger and passes user requests to an AI agent. The agent uses memory plus five tools to interact with Google Calendar and Gmail.

## 1.2 Daily Calendar-to-Sheets Sync

This block starts from a schedule trigger. It clears the target sheet (except the header), fetches today’s calendar events in IST, conditionally formats valid events, appends them to Google Sheets, and separately builds an email summary for the admin.

## 1.3 Documentation / Visual Guidance Layer

The workflow also includes sticky notes that describe purpose, setup requirements, and the distinction between the two parallel branches. These do not execute but are important for maintainability.

---

# 2. Block-by-Block Analysis

## 2.1 Documentation and Visual Structure

### Overview
This block consists of sticky notes used to explain the workflow’s intent, setup, and major branches. These nodes are non-executable but provide operational context and credential requirements.

### Nodes Involved
- ⭐ Main Overview
- Group: AI Chat Flow
- Group: Daily Sheet Sync

### Node Details

#### ⭐ Main Overview
- **Type and technical role**: Sticky Note; visual documentation node.
- **Configuration choices**:
  - Describes the workflow as “BytezTech Appointment Bot”.
  - Explains that two parallel flows run independently.
  - Lists setup steps for Google Calendar OAuth, Gmail OAuth, Google Sheets OAuth and Service Account, and Groq API credentials.
- **Key expressions or variables used**: None.
- **Input and output connections**: None.
- **Version-specific requirements**: Sticky note node version 1.
- **Edge cases or potential failure types**:
  - No runtime failure; risk is only outdated documentation.
- **Sub-workflow reference**: None.

#### Group: AI Chat Flow
- **Type and technical role**: Sticky Note; visual grouping note.
- **Configuration choices**:
  - Documents the AI agent branch as a client-to-agent scheduling flow.
  - Notes use of Groq LLaMA 4, session memory, and 5 Calendar/Gmail tools.
- **Key expressions or variables used**: None.
- **Input and output connections**: None.
- **Version-specific requirements**: Sticky note node version 1.
- **Edge cases or potential failure types**:
  - No execution risk.
- **Sub-workflow reference**: None.

#### Group: Daily Sheet Sync
- **Type and technical role**: Sticky Note; visual grouping note.
- **Configuration choices**:
  - Documents the scheduled reporting branch.
  - States that the flow clears rows, fetches events, formats data, inserts into Sheets, and emails an admin summary.
- **Key expressions or variables used**: None.
- **Input and output connections**: None.
- **Version-specific requirements**: Sticky note node version 1.
- **Edge cases or potential failure types**:
  - No execution risk.
- **Sub-workflow reference**: None.

---

## 2.2 AI Chat Intake and Agent Orchestration

### Overview
This block receives chat messages from clients and routes them into an AI agent configured as a BytezTech scheduling assistant. The agent is backed by a Groq model and short-term conversational memory.

### Nodes Involved
- Clint Chat
- AI Agent
- Groq Chat Model
- Simple Memory

### Node Details

#### Clint Chat
- **Type and technical role**: `@n8n/n8n-nodes-langchain.chatTrigger`; entry point for chat-based interactions.
- **Configuration choices**:
  - Uses default options.
  - Exposes a webhook-backed chat trigger for client messages.
- **Key expressions or variables used**:
  - Produces session-related data consumed later by memory; the workflow expects `sessionId` to be available in the incoming JSON for memory continuity.
- **Input and output connections**:
  - Output to **AI Agent** via main connection.
- **Version-specific requirements**:
  - Type version 1.3.
  - Requires a recent n8n version with LangChain chat trigger support.
- **Edge cases or potential failure types**:
  - Webhook/chat endpoint misconfiguration.
  - Missing `sessionId` may reduce memory continuity.
  - Chat channel integration issues depending on deployment.
- **Sub-workflow reference**: None.

#### AI Agent
- **Type and technical role**: `@n8n/n8n-nodes-langchain.agent`; LLM agent that interprets natural-language scheduling requests and invokes tools.
- **Configuration choices**:
  - Uses a detailed system prompt defining:
    - BytezTech assistant role
    - Booking, rescheduling, cancellation, and retrieval duties
    - Mandatory IST timezone policy
    - Required booking fields
    - Update-vs-delete behavior for rescheduling
    - Explicit cancellation confirmation policy
    - ISO 8601 formatting requirement
    - Error handling behavior for missing email, unavailable slots, and ambiguity
- **Key expressions or variables used**:
  - Consumes chat input from the trigger.
  - Uses AI tool nodes that expose parameters through `$fromAI(...)`.
- **Input and output connections**:
  - Main input from **Clint Chat**
  - AI language model input from **Groq Chat Model**
  - AI memory input from **Simple Memory**
  - AI tool connections from:
    - Create an event in Google Calendar
    - Get many events in Google Calendar
    - Delete an event in Google Calendar
    - Update an event in Google Calendar
    - Send a message in Gmail
- **Version-specific requirements**:
  - Type version 2.2.
  - Requires compatible LangChain agent support in n8n.
- **Edge cases or potential failure types**:
  - Hallucinated tool arguments if user input is ambiguous.
  - Poor date parsing if the user provides informal or incomplete time descriptions.
  - Tool failure propagation from Calendar or Gmail nodes.
  - Agent may ask follow-up questions rather than complete the action if required information is missing.
- **Sub-workflow reference**: None.

#### Groq Chat Model
- **Type and technical role**: `@n8n/n8n-nodes-langchain.lmChatGroq`; language model provider for the AI agent.
- **Configuration choices**:
  - Uses model `meta-llama/llama-4-scout-17b-16e-instruct`.
  - No advanced options configured.
- **Key expressions or variables used**: None.
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_languageModel`.
- **Version-specific requirements**:
  - Type version 1.
  - Requires valid Groq API credentials.
- **Edge cases or potential failure types**:
  - Invalid API key.
  - Rate limits or provider-side outages.
  - Latency or timeout on long tool-using conversations.
- **Sub-workflow reference**: None.

#### Simple Memory
- **Type and technical role**: `@n8n/n8n-nodes-langchain.memoryBufferWindow`; keeps short-term conversation context.
- **Configuration choices**:
  - Session key: `={{ $json.sessionId }}`
  - Session ID type: custom key
  - Context window length: 50
- **Key expressions or variables used**:
  - `$json.sessionId`
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_memory`.
- **Version-specific requirements**:
  - Type version 1.3.
- **Edge cases or potential failure types**:
  - If `sessionId` is absent or unstable, memory continuity breaks.
  - Large context windows can increase token usage indirectly.
- **Sub-workflow reference**: None.

---

## 2.3 AI Agent Tooling: Google Calendar and Gmail

### Overview
This block gives the AI agent the ability to inspect calendar availability, create events, update events, delete events, and send confirmation emails. These nodes are not triggered directly by main workflow connections; they are callable tools used by the agent.

### Nodes Involved
- Create an event in Google Calendar
- Get many events in Google Calendar
- Delete an event in Google Calendar
- Update an event in Google Calendar
- Send a message in Gmail

### Node Details

#### Create an event in Google Calendar
- **Type and technical role**: `n8n-nodes-base.googleCalendarTool`; AI tool for event creation.
- **Configuration choices**:
  - Uses a listed calendar set to `user@example.com`.
  - Receives event fields dynamically from the AI agent.
  - Additional fields include summary, location, attendees, and description.
- **Key expressions or variables used**:
  - `{{ $fromAI('Start', '', 'string') }}`
  - `{{ $fromAI('End', '', 'string') }}`
  - `{{ $fromAI('Summary', '', 'string') }}`
  - `{{ $fromAI('Location', '', 'string') }}`
  - `{{ $fromAI('attendees0_Attendees', '', 'string') }}`
  - `{{ $fromAI('Description', '', 'string') }}`
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_tool`.
- **Version-specific requirements**:
  - Type version 1.3.
  - Requires Google Calendar OAuth2 credentials.
- **Edge cases or potential failure types**:
  - Invalid ISO datetime formatting.
  - Calendar access denied.
  - Missing attendee email.
  - Scheduling conflicts if the agent fails to check first.
- **Sub-workflow reference**: None.

#### Get many events in Google Calendar
- **Type and technical role**: `n8n-nodes-base.googleCalendarTool`; AI tool for availability checks and event lookup.
- **Configuration choices**:
  - Operation: `getAll`
  - Return all events: enabled
  - Calendar set to `user@example.com`
  - Uses dynamic time boundaries from the agent.
- **Key expressions or variables used**:
  - `{{ $fromAI('After', '', 'string') }}`
  - `{{ $fromAI('Before', '', 'string') }}`
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_tool`.
- **Version-specific requirements**:
  - Type version 1.3.
- **Edge cases or potential failure types**:
  - Poorly formatted time bounds.
  - Very broad ranges may return large result sets.
  - Permission or quota issues with Google Calendar API.
- **Sub-workflow reference**: None.

#### Delete an event in Google Calendar
- **Type and technical role**: `n8n-nodes-base.googleCalendarTool`; AI tool for event deletion.
- **Configuration choices**:
  - Operation: `delete`
  - Calendar set to `user@example.com`
  - Event ID is supplied dynamically by the agent.
- **Key expressions or variables used**:
  - `{{ $fromAI('Event_ID', '', 'string') }}`
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_tool`.
- **Version-specific requirements**:
  - Type version 1.3.
- **Edge cases or potential failure types**:
  - Invalid or missing event ID.
  - Event already deleted.
  - Calendar access denied.
- **Sub-workflow reference**: None.

#### Update an event in Google Calendar
- **Type and technical role**: `n8n-nodes-base.googleCalendarTool`; AI tool for rescheduling or editing event data.
- **Configuration choices**:
  - Operation: `update`
  - Calendar set to `user@example.com`
  - Dynamic update fields: start, end, location
- **Key expressions or variables used**:
  - `{{ $fromAI('Event_ID', '', 'string') }}`
  - `{{ $fromAI('Start', '', 'string') }}`
  - `{{ $fromAI('End', '', 'string') }}`
  - `{{ $fromAI('Location', '', 'string') }}`
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_tool`.
- **Version-specific requirements**:
  - Type version 1.3.
- **Edge cases or potential failure types**:
  - Missing event ID.
  - Invalid datetime values.
  - Partial updates may leave stale fields if the agent omits expected values.
- **Sub-workflow reference**: None.

#### Send a message in Gmail
- **Type and technical role**: `n8n-nodes-base.gmailTool`; AI tool for sending confirmation, reschedule, or cancellation emails.
- **Configuration choices**:
  - Uses dynamic recipient, subject, and body from the agent.
  - Gmail OAuth2 credential is required.
- **Key expressions or variables used**:
  - `{{ $fromAI('To', '', 'string') }}`
  - `{{ $fromAI('Subject', '', 'string') }}`
  - `{{ $fromAI('Message', '', 'string') }}`
- **Input and output connections**:
  - Connected to **AI Agent** through `ai_tool`.
- **Version-specific requirements**:
  - Type version 2.1.
- **Edge cases or potential failure types**:
  - Invalid recipient address.
  - Gmail OAuth consent or token expiration issues.
  - Sending limits or anti-abuse restrictions.
- **Sub-workflow reference**: None.

---

## 2.4 Daily Trigger and Sheet Reset

### Overview
This block starts the daily reporting branch and clears old appointment rows from the target Google Sheet while preserving the header row.

### Nodes Involved
- Schedule Trigger
- Clear Old Appointments

### Node Details

#### Schedule Trigger
- **Type and technical role**: `n8n-nodes-base.scheduleTrigger`; scheduled entry point.
- **Configuration choices**:
  - Runs daily at 9:30.
  - The sticky note states IST, but the node itself does not explicitly set timezone in its parameters.
- **Key expressions or variables used**: None.
- **Input and output connections**:
  - Output to **Clear Old Appointments**.
- **Version-specific requirements**:
  - Type version 1.
  - Actual runtime timezone depends on workflow/server timezone unless instance-level timezone is configured to IST.
- **Edge cases or potential failure types**:
  - If the n8n instance timezone is not Asia/Kolkata, the trigger may fire at a different local time than expected.
- **Sub-workflow reference**: None.

#### Clear Old Appointments
- **Type and technical role**: `n8n-nodes-base.googleSheets`; clears sheet contents.
- **Configuration choices**:
  - Operation: `clear`
  - Document: spreadsheet named “Apponitment Booking”
  - Sheet target: `gid=0` / Sheet1
  - `keepFirstRow` enabled to preserve column headers
  - Uses Google Sheets OAuth2 authentication
- **Key expressions or variables used**: None.
- **Input and output connections**:
  - Input from **Schedule Trigger**
  - Output to **Get Today Events**
- **Version-specific requirements**:
  - Type version 4.7.
- **Edge cases or potential failure types**:
  - Wrong sheet selected could erase unintended data.
  - OAuth token expiration.
  - Insufficient spreadsheet permissions.
- **Sub-workflow reference**: None.

---

## 2.5 Daily Event Retrieval and Summary Counting

### Overview
This block fetches all calendar events for the current day in IST and simultaneously sends the results into two paths: one for row-by-row sheet insertion and one for aggregate summary generation.

### Nodes Involved
- Get Today Events
- IF Has Events
- Build Daily Summary

### Node Details

#### Get Today Events
- **Type and technical role**: `n8n-nodes-base.googleCalendar`; retrieves today’s events from Google Calendar.
- **Configuration choices**:
  - Operation: `getAll`
  - Return all results enabled
  - Calendar specified by ID `user@example.com`
  - Time range:
    - Start of day in Asia/Kolkata
    - End of day in Asia/Kolkata
  - `alwaysOutputData` enabled so downstream nodes still execute even with no events
- **Key expressions or variables used**:
  - `{{ $now.setZone('Asia/Kolkata').startOf('day').toISO() }}`
  - `{{ $now.setZone('Asia/Kolkata').endOf('day').toISO() }}`
- **Input and output connections**:
  - Input from **Clear Old Appointments**
  - Outputs to:
    - **IF Has Events**
    - **Build Daily Summary**
- **Version-specific requirements**:
  - Type version 1.3.
  - Requires Luxon/DateTime expression support, which n8n provides in expressions.
- **Edge cases or potential failure types**:
  - Calendar auth failure.
  - Empty result set, though downstream processing is partially protected by `alwaysOutputData`.
  - All-day events may have different date structures than timed events.
- **Sub-workflow reference**: None.

#### IF Has Events
- **Type and technical role**: `n8n-nodes-base.if`; filters only items that appear to represent real calendar events.
- **Configuration choices**:
  - Condition: string `id` is not empty
- **Key expressions or variables used**:
  - `{{ $json.id }}`
- **Input and output connections**:
  - Input from **Get Today Events**
  - True output to **Format Event Data**
  - No false branch connected
- **Version-specific requirements**:
  - Type version 2.
- **Edge cases or potential failure types**:
  - If Google returns output without `id`, valid edge-case records would be skipped.
  - False branch is intentionally ignored, so no explicit no-event handling here.
- **Sub-workflow reference**: None.

#### Build Daily Summary
- **Type and technical role**: `n8n-nodes-base.code`; aggregates event count and prepares email metadata.
- **Configuration choices**:
  - JavaScript counts items where `json.id` exists.
  - Produces a single summary object with:
    - `count`
    - `reportDate` formatted in IST as `dd/MM/yyyy`
    - `sheetUrl`
- **Key expressions or variables used**:
  - Uses `items.filter(i => i.json && i.json.id).length`
  - Uses `$now.setZone('Asia/Kolkata').toFormat('dd/MM/yyyy')`
- **Input and output connections**:
  - Input from **Get Today Events**
  - Output to **Send Email to Admin**
- **Version-specific requirements**:
  - Type version 2.
- **Edge cases or potential failure types**:
  - If upstream returns a placeholder item with no `id`, count becomes 0 as intended.
  - Hardcoded sheet URL must stay in sync with the actual spreadsheet.
- **Sub-workflow reference**: None.

---

## 2.6 Event Formatting and Sheet Insertion

### Overview
This block converts raw Google Calendar event objects into a sheet-friendly schema and appends them to Google Sheets.

### Nodes Involved
- Format Event Data
- Insert Events Details

### Node Details

#### Format Event Data
- **Type and technical role**: `n8n-nodes-base.set`; transforms raw event records into normalized appointment fields.
- **Configuration choices**:
  - Keeps only explicitly set fields.
  - Builds the following output schema:
    - `ID`
    - `Full Name`
    - `Email`
    - `Purpose`
    - `Date (IST)`
    - `Start Time (IST)`
    - `End Time (IST)`
    - `Duration`
    - `Status`
  - Includes fallbacks for missing fields:
    - Summary or private extended property for name
    - “N/A” for missing email/name
    - “Meeting” fallback purpose
    - Generated fallback ID if event `id` is missing
- **Key expressions or variables used**:
  - ID fallback:
    - `{{ $json.id || ($now.setZone('Asia/Kolkata').toFormat('yyyyMMdd-HHmmss') + '-' + $itemIndex) }}`
  - Full Name:
    - `{{ $json.summary || $json.extendedProperties?.private?.fullName || 'N/A' }}`
  - Email:
    - maps attendees to joined email string
  - Purpose:
    - `{{ $json.description || $json.summary || 'Meeting' }}`
  - Date/time formatting:
    - `DateTime.fromISO(...).setZone('Asia/Kolkata').toFormat(...)`
  - Duration:
    - computes rounded minute difference with minimum 1 minute
- **Input and output connections**:
  - Input from **IF Has Events**
  - Output to **Insert Events Details**
- **Version-specific requirements**:
  - Type version 2.
  - Depends on expression support for Luxon `DateTime`.
- **Edge cases or potential failure types**:
  - Malformed `start.dateTime` or `end.dateTime` values could break formatting.
  - All-day events are handled with fallback values like `All Day` and `All day`.
  - `summary` is used as full name, which may not always actually be a person’s name.
- **Sub-workflow reference**: None.

#### Insert Events Details
- **Type and technical role**: `n8n-nodes-base.googleSheets`; appends normalized event rows into the target sheet.
- **Configuration choices**:
  - Operation: `append`
  - Authentication: service account
  - Spreadsheet: “Apponitment Booking”
  - Sheet: `Sheet1`
  - Explicit column mapping for all formatted fields
  - Mapping mode: define below
  - No automatic type conversion
- **Key expressions or variables used**:
  - Maps each sheet column from formatted JSON fields, such as:
    - `{{ $json.ID }}`
    - `{{ $json['Full Name'] }}`
    - `{{ $json.Email }}`
    - etc.
- **Input and output connections**:
  - Input from **Format Event Data**
  - No downstream node
- **Version-specific requirements**:
  - Type version 4.7.
  - Requires Google Service Account credentials with spreadsheet access.
- **Edge cases or potential failure types**:
  - Service account lacks access to the sheet unless the spreadsheet is explicitly shared with it.
  - Header mismatch between schema and actual sheet can produce mapping issues.
  - Duplicate rows may occur if the flow reruns after partial completion.
- **Sub-workflow reference**: None.

---

## 2.7 Admin Notification

### Overview
This block emails the admin a daily summary after the event retrieval phase, regardless of whether event insertion happens for every item.

### Nodes Involved
- Send Email to Admin

### Node Details

#### Send Email to Admin
- **Type and technical role**: `n8n-nodes-base.gmail`; sends a standard Gmail message to the administrator.
- **Configuration choices**:
  - Recipient is hardcoded as `user@example.com`
  - Subject includes the formatted report date
  - Message includes:
    - total meeting count
    - sheet link
    - a greeting and signature
- **Key expressions or variables used**:
  - Subject:
    - `📅 Daily Meeting Report - {{ $json.reportDate }}`
  - Message:
    - references `{{ $json.count }}` and `{{ $json.sheetUrl }}`
- **Input and output connections**:
  - Input from **Build Daily Summary**
  - No downstream node
- **Version-specific requirements**:
  - Type version 2.1.
  - Requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types**:
  - Invalid or placeholder admin address.
  - Gmail auth expiration.
  - Mail sending quota or provider restrictions.
- **Sub-workflow reference**: None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⭐ Main Overview | n8n-nodes-base.stickyNote | Global workflow documentation |  |  | # 🗓️ BytezTech Appointment Bot<br>## How it works<br>Two parallel flows run independently:<br>1. **AI Chat Flow** — clients message the bot to book, reschedule, or cancel meetings. The agent checks Google Calendar availability, creates/updates/deletes events, and sends Gmail confirmations automatically.<br>2. **Daily Sync Flow** — every morning at 9:30 AM IST, today's calendar events are cleared from the sheet, re-fetched, formatted, and re-inserted. An admin summary email is then sent.<br>## Setup steps<br>1. Add **Google Calendar OAuth** credential (jigneshmevada87@gmail.com)<br>2. Add **Gmail OAuth** credential for agent tool + admin report node<br>3. Add **Google Sheets OAuth** for Clear node; **Service Account** for Insert node<br>4. Add **Groq API** key for the LLaMA 4 Scout language model<br>5. **Activate** the workflow — both flows will run independently |
| Group: AI Chat Flow | n8n-nodes-base.stickyNote | Visual grouping for AI branch |  |  | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Group: Daily Sheet Sync | n8n-nodes-base.stickyNote | Visual grouping for scheduled sync branch |  |  | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Clint Chat | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point for appointment requests |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | LLM scheduling coordinator | Clint Chat; Groq Chat Model; Simple Memory; Create an event in Google Calendar; Get many events in Google Calendar; Delete an event in Google Calendar; Update an event in Google Calendar; Send a message in Gmail |  | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Groq Chat Model | @n8n/n8n-nodes-langchain.lmChatGroq | LLM provider for the agent |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Session conversation memory |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Create an event in Google Calendar | n8n-nodes-base.googleCalendarTool | AI tool to create appointments |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Get many events in Google Calendar | n8n-nodes-base.googleCalendarTool | AI tool to check availability and find events |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Delete an event in Google Calendar | n8n-nodes-base.googleCalendarTool | AI tool to cancel appointments |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Update an event in Google Calendar | n8n-nodes-base.googleCalendarTool | AI tool to reschedule appointments |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Send a message in Gmail | n8n-nodes-base.gmailTool | AI tool to send client email notifications |  | AI Agent | ## 🤖 AI Chat Agent Flow<br>Client sends a chat message → AI Agent (Groq LLaMA 4) processes the request using session memory and 5 Calendar/Gmail tools. Handles booking, rescheduling, and cancellation end-to-end. |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily entry point for sync/report flow |  | Clear Old Appointments | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Clear Old Appointments | n8n-nodes-base.googleSheets | Clears previous rows while keeping headers | Schedule Trigger | Get Today Events | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Get Today Events | n8n-nodes-base.googleCalendar | Fetches current day calendar events in IST | Clear Old Appointments | IF Has Events; Build Daily Summary | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| IF Has Events | n8n-nodes-base.if | Filters valid event records | Get Today Events | Format Event Data | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Format Event Data | n8n-nodes-base.set | Normalizes event fields for Sheets | IF Has Events | Insert Events Details | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Insert Events Details | n8n-nodes-base.googleSheets | Appends formatted events into Google Sheets | Format Event Data |  | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Build Daily Summary | n8n-nodes-base.code | Builds aggregate admin report payload | Get Today Events | Send Email to Admin | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |
| Send Email to Admin | n8n-nodes-base.gmail | Sends daily summary email | Build Daily Summary |  | ## 📊 Daily Sheet Sync Flow<br>Scheduled at 9:30 AM IST daily. Clears yesterday's sheet rows, fetches today's Google Calendar events, formats each event, inserts into Sheets, then emails the admin a count summary with sheet link. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add three Sticky Note nodes** for documentation:
   1. One overall note titled something like “BytezTech Appointment Bot” describing both branches and setup steps.
   2. One note for the AI chat branch.
   3. One note for the daily sync branch.

## Build the AI Chat Flow

3. **Add a Chat Trigger node**
   - Type: `Chat Trigger`
   - Name it `Clint Chat`
   - Keep default options unless your chat deployment requires specific settings.

4. **Add an AI Agent node**
   - Type: `AI Agent`
   - Name it `AI Agent`
   - In the system message, paste a scheduling instruction set equivalent to:
     - role = BytezTech scheduling assistant
     - timezone = IST only
     - responsibilities = booking, rescheduling, cancellation, retrieval
     - booking requires full name, email, purpose, date, time, duration
     - rescheduling must update existing event rather than delete/recreate
     - cancellation must only happen on explicit user request
     - use ISO 8601 with `+05:30`
     - ask follow-up questions for missing email or ambiguity
     - suggest 3 alternatives when slot unavailable

5. **Connect `Clint Chat` → `AI Agent`** using the main connection.

6. **Add a Groq Chat Model node**
   - Type: `Groq Chat Model`
   - Name it `Groq Chat Model`
   - Select model: `meta-llama/llama-4-scout-17b-16e-instruct`
   - Create/select a **Groq API credential**

7. **Connect `Groq Chat Model` to `AI Agent`** via the `ai_languageModel` port.

8. **Add a Memory node**
   - Type: `Simple Memory` / buffer window memory
   - Name it `Simple Memory`
   - Session ID type: `Custom Key`
   - Session key expression: `{{ $json.sessionId }}`
   - Context window length: `50`

9. **Connect `Simple Memory` to `AI Agent`** via the `ai_memory` port.

10. **Add a Google Calendar Tool node for event creation**
    - Type: `Google Calendar Tool`
    - Name it `Create an event in Google Calendar`
    - Choose the target calendar, e.g. the owner mailbox calendar
    - Set:
      - Start = `{{ $fromAI('Start', '', 'string') }}`
      - End = `{{ $fromAI('End', '', 'string') }}`
      - Summary = `{{ $fromAI('Summary', '', 'string') }}`
      - Location = `{{ $fromAI('Location', '', 'string') }}`
      - Description = `{{ $fromAI('Description', '', 'string') }}`
      - Attendee = `{{ $fromAI('attendees0_Attendees', '', 'string') }}`
    - Use **Google Calendar OAuth2** credentials

11. **Connect this node to `AI Agent`** through the `ai_tool` connection.

12. **Add a Google Calendar Tool node for availability lookup**
    - Name it `Get many events in Google Calendar`
    - Operation: `Get Many` / `Get All`
    - Return all: enabled
    - Calendar: same calendar as above
    - Time Min = `{{ $fromAI('After', '', 'string') }}`
    - Time Max = `{{ $fromAI('Before', '', 'string') }}`

13. **Connect it to `AI Agent`** through `ai_tool`.

14. **Add a Google Calendar Tool node for deletion**
    - Name it `Delete an event in Google Calendar`
    - Operation: `Delete`
    - Event ID = `{{ $fromAI('Event_ID', '', 'string') }}`
    - Calendar: same calendar
    - Use the same Google Calendar OAuth2 credential

15. **Connect it to `AI Agent`** through `ai_tool`.

16. **Add a Google Calendar Tool node for updates**
    - Name it `Update an event in Google Calendar`
    - Operation: `Update`
    - Event ID = `{{ $fromAI('Event_ID', '', 'string') }}`
    - Update fields:
      - Start = `{{ $fromAI('Start', '', 'string') }}`
      - End = `{{ $fromAI('End', '', 'string') }}`
      - Location = `{{ $fromAI('Location', '', 'string') }}`
    - Calendar: same calendar

17. **Connect it to `AI Agent`** through `ai_tool`.

18. **Add a Gmail Tool node**
    - Type: `Gmail Tool`
    - Name it `Send a message in Gmail`
    - Send To = `{{ $fromAI('To', '', 'string') }}`
    - Subject = `{{ $fromAI('Subject', '', 'string') }}`
    - Message = `{{ $fromAI('Message', '', 'string') }}`
    - Use **Gmail OAuth2** credentials

19. **Connect it to `AI Agent`** through `ai_tool`.

## Build the Daily Sheet Sync Flow

20. **Add a Schedule Trigger node**
    - Name it `Schedule Trigger`
    - Set it to run daily at `09:30`
    - Important: ensure your n8n instance timezone is set to **Asia/Kolkata** if you want true 9:30 AM IST execution.

21. **Add a Google Sheets node to clear rows**
    - Name it `Clear Old Appointments`
    - Operation: `Clear`
    - Select the spreadsheet used for appointment storage
    - Select the target sheet
    - Enable `Keep First Row`
    - Use **Google Sheets OAuth2** credentials

22. **Connect `Schedule Trigger` → `Clear Old Appointments`**.

23. **Add a Google Calendar node**
    - Name it `Get Today Events`
    - Operation: `Get All`
    - Return all: enabled
    - Calendar: select the same appointment calendar
    - Time Min expression:
      - `{{ $now.setZone('Asia/Kolkata').startOf('day').toISO() }}`
    - Time Max expression:
      - `{{ $now.setZone('Asia/Kolkata').endOf('day').toISO() }}`
    - In node settings, enable `Always Output Data`

24. **Connect `Clear Old Appointments` → `Get Today Events`**.

25. **Add an IF node**
    - Name it `IF Has Events`
    - Condition type: String
    - Check that `{{ $json.id }}` is not empty

26. **Connect `Get Today Events` → `IF Has Events`**.

27. **Add a Set node**
    - Name it `Format Event Data`
    - Enable `Keep Only Set`
    - Create the following fields:

   - `ID` = `{{ $json.id || ($now.setZone('Asia/Kolkata').toFormat('yyyyMMdd-HHmmss') + '-' + $itemIndex) }}`
   - `Full Name` = `{{ $json.summary || $json.extendedProperties?.private?.fullName || 'N/A' }}`
   - `Email` = `{{ ($json.attendees && $json.attendees.length) ? $json.attendees.map(a => a.email).filter(Boolean).join(', ') : 'N/A' }}`
   - `Purpose` = `{{ $json.description || $json.summary || 'Meeting' }}`
   - `Date (IST)` = `{{ $json.start?.dateTime ? DateTime.fromISO($json.start.dateTime).setZone('Asia/Kolkata').toFormat('yyyy-MM-dd') : ($json.start?.date || $now.setZone('Asia/Kolkata').toFormat('yyyy-MM-dd')) }}`
   - `Start Time (IST)` = `{{ $json.start?.dateTime ? DateTime.fromISO($json.start.dateTime).setZone('Asia/Kolkata').toFormat('HH:mm') : 'All Day' }}`
   - `End Time (IST)` = `{{ $json.end?.dateTime ? DateTime.fromISO($json.end.dateTime).setZone('Asia/Kolkata').toFormat('HH:mm') : 'All Day' }}`
   - `Duration` = `{{ ($json.start?.dateTime && $json.end?.dateTime) ? (Math.max(1, Math.round(DateTime.fromISO($json.end.dateTime).diff(DateTime.fromISO($json.start.dateTime), 'minutes').minutes)) + ' minutes') : 'All day' }}`
   - `Status` = `Confirmed`

28. **Connect `IF Has Events` true output → `Format Event Data`**.

29. **Create the target Google Sheet**
    - Spreadsheet name can be anything, but it must contain header columns matching:
      - `ID`
      - `Full Name`
      - `Email`
      - `Purpose`
      - `Date (IST)`
      - `Start Time (IST)`
      - `End Time (IST)`
      - `Duration`
      - `Status`

30. **Add a Google Sheets node for append**
    - Name it `Insert Events Details`
    - Operation: `Append`
    - Authentication: `Service Account`
    - Select the spreadsheet and sheet
    - Set mapping mode to manual/define-below
    - Map each sheet column from the Set node fields

31. **Configure Service Account access**
    - Create/select a Google Service Account credential in n8n
    - Share the target Google Sheet with the service account email
    - Without this, append operations will fail with permission errors

32. **Connect `Format Event Data` → `Insert Events Details`**.

33. **Add a Code node**
    - Name it `Build Daily Summary`
    - Paste equivalent code:

   ```javascript
   const count = items.filter(i => i.json && i.json.id).length;
   return [
     {
       json: {
         count,
         reportDate: $now.setZone('Asia/Kolkata').toFormat('dd/MM/yyyy'),
         sheetUrl: 'YOUR_SHEET_URL'
       }
     }
   ];
   ```

34. **Connect `Get Today Events` → `Build Daily Summary`**.

35. **Add a Gmail node**
    - Type: regular `Gmail` node, not Gmail Tool
    - Name it `Send Email to Admin`
    - Send To: admin email address
    - Subject: `📅 Daily Meeting Report - {{ $json.reportDate }}`
    - Message body should include:
      - total meeting count: `{{ $json.count }}`
      - sheet URL: `{{ $json.sheetUrl }}`
    - Use **Gmail OAuth2** credentials

36. **Connect `Build Daily Summary` → `Send Email to Admin`**.

## Final validation

37. **Verify credentials**
    - Groq API credential for the model node
    - Google Calendar OAuth2 for all calendar nodes/tools
    - Gmail OAuth2 for both Gmail nodes
    - Google Sheets OAuth2 for the clear node
    - Google Service Account for the append node

38. **Verify calendar and email placeholders**
    - Replace `user@example.com` with the real calendar/admin addresses.
    - Confirm the selected Google Calendar matches the intended owner account.

39. **Verify timezone handling**
    - The event-fetch expressions explicitly use `Asia/Kolkata`.
    - The schedule trigger itself may depend on server/workflow timezone; set n8n accordingly.

40. **Test the AI branch**
    - Send sample messages:
      - “Book a meeting tomorrow at 3 PM IST”
      - “Reschedule my meeting to Friday at 11 AM”
      - “Cancel my appointment”
    - Confirm the agent asks for missing email/name when needed.

41. **Test the daily branch**
    - Run the scheduled path manually.
    - Confirm:
      - old sheet rows are cleared except headers
      - today’s events are inserted
      - admin email contains the correct count and link

42. **Activate the workflow** once both branches work correctly.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BytezTech Appointment Bot manages two independent flows: AI chat scheduling and daily calendar-to-sheet synchronization. | Workflow-wide context |
| Setup requires Google Calendar OAuth, Gmail OAuth, Google Sheets OAuth for clearing, Google Service Account for insertion, and Groq API access. | Workflow-wide setup |
| The daily flow is described as running at 9:30 AM IST, but actual trigger timing depends on the n8n instance or workflow timezone configuration. | Operational caution |
| Spreadsheet used in the workflow is labeled “Apponitment Booking”. | Target Google Sheet |
| Sheet link used in summary logic: https://docs.google.com/spreadsheets/d/1X5KnR2bwuy4_mmD0Imw6k6fA7Bk91Drrr4jngz_IYkk/edit#gid=0 | Admin report / sheet reference |