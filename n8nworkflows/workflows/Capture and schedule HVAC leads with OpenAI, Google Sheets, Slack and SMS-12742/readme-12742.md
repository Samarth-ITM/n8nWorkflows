Capture and schedule HVAC leads with OpenAI, Google Sheets, Slack and SMS

https://n8nworkflows.xyz/workflows/capture-and-schedule-hvac-leads-with-openai--google-sheets--slack-and-sms-12742


# Capture and schedule HVAC leads with OpenAI, Google Sheets, Slack and SMS

## 1. Workflow Overview

**Workflow name (JSON):** HVAC Lead Capture and Automated Service Routing  
**Provided title:** Capture and schedule HVAC leads with OpenAI, Google Sheets, Slack and SMS  
**Purpose:** Automate door-to-door HVAC lead capture, AI-based triage (urgency + category), internal routing/notifications, client confirmation, appointment scheduling, optional SMS reminder, and optional CRM/dispatch push.

### 1.1 Trigger & Configuration
Receives lead data from an n8n Form and sets central configuration values (Slack channel IDs, Calendar ID, CRM endpoint). Normalizes incoming form fields into consistent internal keys.

### 1.2 Lead Logging
Stores/updates the lead record in Google Sheets keyed by email.

### 1.3 AI Classification
Uses an OpenAI chat model through an n8n LangChain Agent node with a structured output parser to classify urgency and service category.

### 1.4 Routing & Team Notification
Routes leads by AI-determined service category (Installation/Repair/Maintenance) and posts a formatted Slack message to the appropriate team channel.

### 1.5 Client Confirmation, Scheduling, Reminders, and Handoff
Sends a confirmation email, creates a Google Calendar event for “tomorrow” by default, optionally sends an SMS reminder via Twilio, then optionally forwards the lead to an external CRM/dispatch API.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Config

**Overview:** Captures the lead from a hosted n8n form, injects environment-specific configuration placeholders, and normalizes form fields into stable property names used across the workflow.

**Nodes involved:**
- Lead Submission Form
- Workflow Configuration
- Normalize Lead Data

#### Node: Lead Submission Form
- **Type / role:** `Form Trigger` — entry point; collects lead details via an n8n-hosted form.
- **Key configuration:**
  - Form title: “HVAC Service Request”
  - Fields (all required):
    - Full Name (text)
    - Email Address (email)
    - Phone Number (text)
    - Property Address (text)
    - Service Type (dropdown: Installation, Repair, Maintenance)
    - Describe Your HVAC Need (textarea)
  - Attribution appending disabled (`appendAttribution: false`)
- **Input / output:**
  - **Input:** none (trigger)
  - **Output:** one item with keys exactly matching field labels (e.g., `$json['Full Name']`).
- **Edge cases / failures:**
  - Users may enter non-E.164 phone numbers; later SMS delivery may fail.
  - “Service Type” from the form may conflict with AI classification later (workflow routes by AI output, not the form selection).
- **Version notes:** Node typeVersion `2.3` (Form Trigger behavior may vary by n8n version; ensure Form Trigger is available in your n8n edition).

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes configurable IDs/URLs.
- **Key configuration (placeholders to replace):**
  - `installationSlackChannel`: Slack channel ID for installation team
  - `repairSlackChannel`: Slack channel ID for repair team
  - `maintenanceSlackChannel`: Slack channel ID for maintenance team
  - `calendarId`: Google Calendar ID for appointments
  - `crmApiUrl`: CRM/Dispatch API endpoint URL
  - `includeOtherFields: true` (keeps original form data too)
- **Input / output connections:**
  - Input: Lead Submission Form
  - Output: Normalize Lead Data
- **Expressions / variables used elsewhere:**
  - Referenced by later nodes via: `$('Workflow Configuration').first().json.<key>`
- **Edge cases / failures:**
  - If placeholders are not replaced, Slack/Calendar/HTTP nodes will fail (invalid channel ID, calendar ID, or URL).
  - If this node ever outputs zero items (unusual), downstream `.first()` calls can error.

#### Node: Normalize Lead Data
- **Type / role:** `Set` — maps form field labels to stable internal names and timestamps the submission.
- **Key mappings:**
  - `leadName` = `{{$json['Full Name']}}`
  - `leadEmail` = `{{$json['Email Address']}}`
  - `leadPhone` = `{{$json['Phone Number']}}`
  - `leadAddress` = `{{$json['Property Address']}}`
  - `serviceType` = `{{$json['Service Type']}}` (original user-chosen)
  - `serviceDescription` = `{{$json['Describe Your HVAC Need']}}`
  - `submittedAt` = `{{$now.toISO()}}`
  - `includeOtherFields: true`
- **Input / output connections:**
  - Input: Workflow Configuration
  - Output: Store Lead in Google Sheets
- **Edge cases / failures:**
  - If a form label changes, these expressions break (because they reference the label string).
  - `leadEmail` missing/invalid can impact Google Sheets matching and email sending.

---

### Block 2 — Lead Logging (Google Sheets)

**Overview:** Writes the normalized lead record into a Google Sheet, appending or updating based on matching the lead’s email.

**Nodes involved:**
- Store Lead in Google Sheets

#### Node: Store Lead in Google Sheets
- **Type / role:** `Google Sheets` — persistence/logging.
- **Operation:** `appendOrUpdate`
- **Key configuration choices:**
  - Document: placeholder Google Sheets Document ID
  - Sheet name: placeholder (e.g., “Leads”)
  - Mapping mode: auto-map input fields
  - Matching columns: `leadEmail` (used to update if existing)
- **Credentials:** Google Sheets OAuth2
- **Input / output connections:**
  - Input: Normalize Lead Data
  - Output: Classify Lead Urgency and Service Type
- **Edge cases / failures:**
  - OAuth permission/scopes issues (401/403).
  - Sheet name mismatch, missing worksheet tab, or wrong document ID.
  - If `leadEmail` is empty, appendOrUpdate may append duplicates or fail depending on node behavior.
  - Column headers in the sheet must align with incoming keys for clean mapping; otherwise data lands in unexpected columns.

---

### Block 3 — AI Logic (Classification)

**Overview:** Sends lead context to OpenAI to classify (1) urgency and (2) service category, enforcing a JSON schema via a structured output parser. The result is used for routing and messaging.

**Nodes involved:**
- Classify Lead Urgency and Service Type (LangChain Agent)
- OpenAI Chat Model
- Structured Output Parser

#### Node: OpenAI Chat Model
- **Type / role:** `LangChain Chat Model (OpenAI)` — provides the underlying LLM.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API (configured in n8n)
- **Connections:**
  - Connected to the Agent node via `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid API key, insufficient quota, or model not available in your account/region.
  - Latency/timeouts on large workloads.

#### Node: Structured Output Parser
- **Type / role:** `LangChain Structured Output Parser` — forces the model response into a defined schema.
- **Schema (manual JSON schema):**
  - `urgency` (string): Low | Medium | High | Emergency
  - `serviceCategory` (string): Installation | Repair | Maintenance
  - `reasoning` (string): brief explanation
- **Connections:**
  - Connected to the Agent node via `ai_outputParser`.
- **Edge cases / failures:**
  - If the model returns non-conforming output, parsing fails and downstream routing breaks.
  - Schema does not explicitly require properties; consider adding “required” fields if you want stricter validation.

#### Node: Classify Lead Urgency and Service Type
- **Type / role:** `LangChain Agent` — orchestrates prompt + model + structured parsing.
- **Prompt content (summarized):**
  - Injects: `leadName`, `serviceType` (from form), `serviceDescription`, `leadAddress`
  - System message defines urgency heuristics and requires structured return.
- **Key expressions:**
  - Uses `{{$json.leadName}}`, `{{$json.serviceType}}`, `{{$json.serviceDescription}}`, `{{$json.leadAddress}}`
- **Output shape used later:**
  - Parsed fields appear under: `$json.output.urgency`, `$json.output.serviceCategory`, `$json.output.reasoning`
- **Input / output connections:**
  - Input: Store Lead in Google Sheets
  - Output: Route by Service Type
  - AI inputs: OpenAI Chat Model + Structured Output Parser
- **Edge cases / failures:**
  - Prompt ambiguity: user-selected `serviceType` may bias the model; if you want the AI to override, keep it; if you want it to confirm, clarify in system message.
  - If `serviceDescription` is empty or too short, classification may be unreliable.

---

### Block 4 — Routing & Internal Notification (Slack)

**Overview:** Routes the lead to one of three Slack notification nodes based on AI `serviceCategory`, then all routes converge into the same post-notification flow.

**Nodes involved:**
- Route by Service Type
- Notify Installation Team
- Notify Repair Team
- Notify Maintenance Team

#### Node: Route by Service Type
- **Type / role:** `Switch` — conditional branching.
- **Rules:**
  - If `{{$json.output.serviceCategory}} == "Installation"` → “Installation” output
  - If `... == "Repair"` → “Repair” output
  - If `... == "Maintenance"` → “Maintenance” output
- **Input / output connections:**
  - Input: Classify Lead Urgency and Service Type
  - Outputs:
    - Installation → Notify Installation Team
    - Repair → Notify Repair Team
    - Maintenance → Notify Maintenance Team
- **Edge cases / failures:**
  - If the AI returns a non-matching value (e.g., lowercase “repair”), no branch matches and the workflow effectively stops.
  - Consider adding a default/fallback output (not configured here).

#### Node: Notify Installation Team
- **Type / role:** `Slack` — posts a message to the installation channel.
- **Channel:** uses configuration expression:
  - `channelId = $('Workflow Configuration').first().json.installationSlackChannel`
- **Message contents:** includes urgency, customer contact, address, description, and AI reasoning.
- **Input / output connections:**
  - Input: Route by Service Type (Installation branch)
  - Output: Send Confirmation Email to Lead
- **Edge cases / failures:**
  - Wrong channel ID or missing Slack scopes (e.g., `chat:write`).
  - Message includes special characters; generally fine, but formatting may vary.

#### Node: Notify Repair Team
- **Type / role:** `Slack` — posts a message to the repair channel.
- **Channel:** `$('Workflow Configuration').first().json.repairSlackChannel`
- **Flow:** same pattern as above.
- **Input / output connections:**
  - Input: Route by Service Type (Repair branch)
  - Output: Send Confirmation Email to Lead
- **Edge cases / failures:** same as other Slack nodes.

#### Node: Notify Maintenance Team
- **Type / role:** `Slack` — posts a message to the maintenance channel.
- **Channel:** `$('Workflow Configuration').first().json.maintenanceSlackChannel`
- **Input / output connections:**
  - Input: Route by Service Type (Maintenance branch)
  - Output: Send Confirmation Email to Lead

---

### Block 5 — Appointment, Reminder, and Optional CRM Push

**Overview:** Sends a confirmation email to the lead, creates a Google Calendar appointment “tomorrow”, sends an SMS reminder, and posts the lead to an external CRM/dispatch endpoint.

**Nodes involved:**
- Send Confirmation Email to Lead
- Schedule Appointment
- Send SMS Reminder
- Send to CRM/Dispatch System

#### Node: Send Confirmation Email to Lead
- **Type / role:** `Email Send` — customer confirmation.
- **To:** `{{$json.leadEmail}}`
- **From:** placeholder company email address
- **Subject:** “Thank You for Your HVAC Service Request”
- **HTML body:** includes `leadName`, AI-derived `output.serviceCategory` and `output.urgency`, plus address and contact info.
- **Input / output connections:**
  - Input: any of the Slack nodes (all converge here)
  - Output: Schedule Appointment
- **Edge cases / failures:**
  - SMTP/Email credentials not configured or rejected by provider.
  - Using an unverified “from” address can cause delivery failure.
  - If AI output is missing (parser failure), email template expressions may error.

#### Node: Schedule Appointment
- **Type / role:** `Google Calendar` — creates an event.
- **Calendar:** `$('Workflow Configuration').first().json.calendarId`
- **Start / end (relative to now):**
  - Start: `{{$now().plus({ days: 1 }).toISO()}}`
  - End: `{{$now().plus({ days: 1, hours: 2 }).toISO()}}`
- **Event fields:**
  - Summary: `HVAC {{ $json.output.serviceCategory }} - {{ $json.leadName }}`
  - Description: full lead details + urgency + description
- **Input / output connections:**
  - Input: Send Confirmation Email to Lead
  - Output: Send SMS Reminder
- **Edge cases / failures:**
  - Calendar ID invalid or OAuth scope issues.
  - Timezone considerations: `$now()` uses n8n server timezone unless configured; appointments may be scheduled at undesired local times.
  - This schedules an appointment automatically without checking availability—possible double-booking.

#### Node: Send SMS Reminder
- **Type / role:** `Twilio` — sends an SMS reminder/confirmation request.
- **To:** `{{$json.leadPhone}}`
- **From:** placeholder Twilio number
- **Message:** appointment reminder for tomorrow; asks to reply CONFIRM.
- **Input / output connections:**
  - Input: Schedule Appointment
  - Output: Send to CRM/Dispatch System
- **Edge cases / failures:**
  - Phone not in E.164 format → Twilio may reject.
  - Region restrictions, messaging compliance (A2P 10DLC in the US), opt-in requirements.
  - If you truly want “optional”, you’d typically add an IF node; currently it always sends.

#### Node: Send to CRM/Dispatch System
- **Type / role:** `HTTP Request` — pushes lead payload to an external system.
- **Method:** POST
- **URL:** `$('Workflow Configuration').first().json.crmApiUrl`
- **Body:** JSON with lead identity, contact, address, AI classification, description, and submittedAt.
- **Input / output connections:**
  - Input: Send SMS Reminder
  - Output: (none; end)
- **Edge cases / failures:**
  - Missing authentication headers (none configured here). Most CRMs require API keys/OAuth.
  - 4xx/5xx responses; no retry/backoff configured.
  - If `crmApiUrl` placeholder not replaced, request fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Lead Submission Form | Form Trigger | Capture lead via hosted form | — | Workflow Configuration | ## Trigger & Config |
| Workflow Configuration | Set | Store environment-specific IDs/URLs | Lead Submission Form | Normalize Lead Data | ## Trigger & Config |
| Normalize Lead Data | Set | Map form fields to stable internal keys | Workflow Configuration | Store Lead in Google Sheets | ## Trigger & Config |
| Store Lead in Google Sheets | Google Sheets | Append/update lead record in Sheets | Normalize Lead Data | Classify Lead Urgency and Service Type | ## AI Logic |
| Classify Lead Urgency and Service Type | LangChain Agent | Classify urgency + category with structured output | Store Lead in Google Sheets; (AI) OpenAI Chat Model; (AI) Structured Output Parser | Route by Service Type | ## AI Logic |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for classifier | — | Classify Lead Urgency and Service Type (ai_languageModel) | ## AI Logic |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for AI output | — | Classify Lead Urgency and Service Type (ai_outputParser) | ## AI Logic |
| Route by Service Type | Switch | Branch by `output.serviceCategory` | Classify Lead Urgency and Service Type | Notify Installation Team; Notify Repair Team; Notify Maintenance Team | ## Notify |
| Notify Installation Team | Slack | Post lead to Installation channel | Route by Service Type (Installation) | Send Confirmation Email to Lead | ## Notify |
| Notify Repair Team | Slack | Post lead to Repair channel | Route by Service Type (Repair) | Send Confirmation Email to Lead | ## Notify |
| Notify Maintenance Team | Slack | Post lead to Maintenance channel | Route by Service Type (Maintenance) | Send Confirmation Email to Lead | ## Notify |
| Send Confirmation Email to Lead | Email Send | Confirm receipt to the lead | Notify Installation Team / Notify Repair Team / Notify Maintenance Team | Schedule Appointment | ## Appointment & Reminder |
| Schedule Appointment | Google Calendar | Create calendar appointment | Send Confirmation Email to Lead | Send SMS Reminder | ## Appointment & Reminder |
| Send SMS Reminder | Twilio | Send SMS reminder/confirmation prompt | Schedule Appointment | Send to CRM/Dispatch System | ## Appointment & Reminder |
| Send to CRM/Dispatch System | HTTP Request | POST lead to external CRM/dispatch | Send SMS Reminder | — | ## Appointment & Reminder |
| Sticky Note | Sticky Note | Comment/section label | — | — |  |
| Sticky Note1 | Sticky Note | Comment/section label | — | — |  |
| Sticky Note2 | Sticky Note | Comment/section label | — | — |  |
| Sticky Note3 | Sticky Note | Comment/section label | — | — |  |
| Sticky Note4 | Sticky Note | Comment/overview panel | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: “HVAC Lead Capture and Automated Service Routing” (or your preferred name).

2. **Add Trigger: “Lead Submission Form” (Form Trigger)**
   - Title: *HVAC Service Request*
   - Description: *Submit your HVAC service needs and we will get back to you shortly*
   - Fields (all required):
     1) Full Name (Text)  
     2) Email Address (Email)  
     3) Phone Number (Text)  
     4) Property Address (Text)  
     5) Service Type (Dropdown): Installation, Repair, Maintenance  
     6) Describe Your HVAC Need (Textarea)
   - Options: disable attribution append.

3. **Add a Set node: “Workflow Configuration”**
   - Turn on **Include Other Fields**.
   - Add string fields (replace placeholders with real values):
     - `installationSlackChannel` = Slack channel ID
     - `repairSlackChannel` = Slack channel ID
     - `maintenanceSlackChannel` = Slack channel ID
     - `calendarId` = Google Calendar ID
     - `crmApiUrl` = CRM endpoint URL (e.g., `https://api.example.com/leads`)

4. **Add a Set node: “Normalize Lead Data”**
   - Turn on **Include Other Fields**.
   - Add fields (Expressions):
     - `leadName` = `{{$json["Full Name"]}}`
     - `leadEmail` = `{{$json["Email Address"]}}`
     - `leadPhone` = `{{$json["Phone Number"]}}`
     - `leadAddress` = `{{$json["Property Address"]}}`
     - `serviceType` = `{{$json["Service Type"]}}`
     - `serviceDescription` = `{{$json["Describe Your HVAC Need"]}}`
     - `submittedAt` = `{{$now.toISO()}}`

5. **Add Google Sheets node: “Store Lead in Google Sheets”**
   - **Credentials:** Connect Google Sheets OAuth2.
   - Operation: **Append or Update**
   - Document: select your spreadsheet
   - Sheet: select tab (e.g., “Leads”)
   - Matching column(s): `leadEmail`
   - Mapping: Auto-map input data (ensure sheet has columns matching your keys like `leadName`, `leadEmail`, etc.).

6. **Add AI nodes for classification**
   1) **OpenAI Chat Model** node
      - Credentials: connect OpenAI API key
      - Model: `gpt-4.1-mini` (or any suitable chat model)
   2) **Structured Output Parser** node
      - Schema type: Manual
      - Schema: object with `urgency`, `serviceCategory`, `reasoning` as strings (as described above)
   3) **LangChain Agent** node: “Classify Lead Urgency and Service Type”
      - Prompt type: “Define”
      - Text (use expressions):
        - Include Name, Service Type, Description, Address
      - System message: include urgency rubric and required structured response.
      - Connect:
        - OpenAI Chat Model → Agent (AI Language Model connection)
        - Structured Output Parser → Agent (AI Output Parser connection)

7. **Add Switch node: “Route by Service Type”**
   - Create 3 rules:
     - If `{{$json.output.serviceCategory}}` equals `Installation`
     - If equals `Repair`
     - If equals `Maintenance`
   - Ensure each rule routes to a distinct output.

8. **Add Slack nodes (3)**
   - “Notify Installation Team”
     - Resource/operation: post message to channel
     - Channel ID expression: `{{$('Workflow Configuration').first().json.installationSlackChannel}}`
     - Message: include lead fields + `{{$json.output.urgency}}` + `{{$json.output.reasoning}}`
   - “Notify Repair Team”
     - Channel ID: `{{$('Workflow Configuration').first().json.repairSlackChannel}}`
   - “Notify Maintenance Team”
     - Channel ID: `{{$('Workflow Configuration').first().json.maintenanceSlackChannel}}`
   - **Credentials:** connect Slack (OAuth) with permission to post in those channels.

9. **Add Email node: “Send Confirmation Email to Lead”**
   - To: `{{$json.leadEmail}}`
   - From: your company email (must be valid for your email provider/SMTP setup)
   - Subject: confirmation subject
   - HTML body: reference `leadName`, `output.serviceCategory`, `output.urgency`, address, etc.
   - **Credentials:** configure SMTP / email provider in n8n (depends on node setup in your environment).

10. **Add Google Calendar node: “Schedule Appointment”**
   - **Credentials:** connect Google Calendar OAuth2.
   - Calendar ID expression: `{{$('Workflow Configuration').first().json.calendarId}}`
   - Start: `{{$now().plus({ days: 1 }).toISO()}}`
   - End: `{{$now().plus({ days: 1, hours: 2 }).toISO()}}`
   - Summary/Description: include lead details and AI classification.

11. **Add Twilio node: “Send SMS Reminder”**
   - **Credentials:** connect Twilio (Account SID/Auth Token).
   - From: your Twilio phone number
   - To: `{{$json.leadPhone}}`
   - Message: reminder referencing `leadName`, `output.serviceCategory`, address.
   - Note: If SMS should be optional, insert an **IF** node before this step (not present in the provided workflow).

12. **Add HTTP Request node: “Send to CRM/Dispatch System” (optional integration)**
   - Method: POST
   - URL expression: `{{$('Workflow Configuration').first().json.crmApiUrl}}`
   - Body type: JSON
   - JSON payload: include lead fields and AI output (urgency/category)
   - Add auth headers if your CRM requires it (API key / bearer token).

13. **Wire the nodes in order**
   - Form Trigger → Workflow Configuration → Normalize Lead Data → Google Sheets → Agent → Switch
   - Switch outputs:
     - Installation → Notify Installation Team → Send Confirmation Email
     - Repair → Notify Repair Team → Send Confirmation Email
     - Maintenance → Notify Maintenance Team → Send Confirmation Email
   - Send Confirmation Email → Schedule Appointment → Send SMS Reminder → HTTP Request

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Main Automates door-to-door HVAC lead capture and appointment scheduling. AI classifies leads, routes to appropriate service, logs info, sends confirmations, and schedules appointments.” | Sticky note (overview panel) |
| “## Setup 1. Connect Webhook/Form for lead capture 2. Connect OpenAI for lead classification 3. Connect Google Sheets for lead logging 4. Connect Slack for internal notifications 5. Connect email and SMS for client communication 6. Connect calendar for scheduling 7. Optional CRM/dispatch integration” | Sticky note (setup checklist) |
| **Author:** Hyrum Hurst, AI Automation Engineer — **Company:** QuarterSmart — **Contact:** hyrum@quartersmart.com | Sticky note (credits/contact) |
| Sections labeled: “## Trigger & Config”, “## AI Logic”, “## Notify”, “## Appointment & Reminder” | Sticky notes used as visual grouping headers in the canvas |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.