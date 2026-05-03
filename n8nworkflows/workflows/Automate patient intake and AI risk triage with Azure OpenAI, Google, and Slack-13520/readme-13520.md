Automate patient intake and AI risk triage with Azure OpenAI, Google, and Slack

https://n8nworkflows.xyz/workflows/automate-patient-intake-and-ai-risk-triage-with-azure-openai--google--and-slack-13520


# Automate patient intake and AI risk triage with Azure OpenAI, Google, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Patient Pre-Arrival Intake Automation  
**Provided title:** Automate patient intake and AI risk triage with Azure OpenAI, Google, and Slack

This workflow automates two parallel healthcare operations:

### 1.1 Appointment Reminder Branch (scheduled)
Runs hourly, fetches upcoming Google Calendar events, and sends a pre-arrival intake email via Gmail.

### 1.2 Intake Reception (form responses)
Polls a Google Sheets “Form Responses” sheet every minute to detect new intake submissions.

### 1.3 AI Risk Assessment (Azure OpenAI via LangChain Agent)
Builds a structured prompt from the intake row, asks an Azure OpenAI chat model to return **strict JSON** risk triage output.

### 1.4 Normalization + Storage + Notification
Parses and normalizes the AI JSON, writes results into a separate sheet (“Sheet1”), checks if risk is “High”, and if so posts a Slack alert.

---

## 2. Block-by-Block Analysis

### Block 1 — Documentation / Operator Notes
**Overview:** Sticky notes describe purpose, setup, and required credentials for Google/Azure/Slack.  
**Nodes involved:** `📋 Overview`, `Section: Appointment Reminder`, `Section: Intake & AI Analysis`, `Section: Risk Triage & Notification`, `⚠️ Warning: Slack Config`, `⚠️ Warning: Azure OpenAI`

#### Node: 📋 Overview
- **Type/role:** Sticky Note (documentation)
- **Configuration:** Explains both branches, credential setup steps, and customization ideas.
- **Connections:** None.
- **Failure modes:** None (non-executable).

#### Node: Section: Appointment Reminder
- **Type/role:** Sticky Note (documentation)
- **Connections/failures:** None.

#### Node: Section: Intake & AI Analysis
- **Type/role:** Sticky Note (documentation)
- **Connections/failures:** None.

#### Node: Section: Risk Triage & Notification
- **Type/role:** Sticky Note (documentation)
- **Connections/failures:** None.

#### Node: ⚠️ Warning: Slack Config
- **Type/role:** Sticky Note (documentation)
- **Key requirement:** Channel ID must replace `YOUR_SLACK_CHANNEL`; Slack app needs `chat:write`.
- **Connections/failures:** None.

#### Node: ⚠️ Warning: Azure OpenAI
- **Type/role:** Sticky Note (documentation)
- **Key requirement:** Valid Azure OpenAI credential + deployed model `gpt-4o-mini`.
- **Connections/failures:** None.

---

### Block 2 — Appointment Reminder Branch (Hourly)
**Overview:** On an hourly cron, fetches calendar events then sends an email (currently configured with static recipient/content).  
**Nodes involved:** `Hourly Schedule Trigger`, `Fetch Calendar Appointments`, `Send Pre-Arrival Intake Email`

#### Node: Hourly Schedule Trigger
- **Type/role:** Cron trigger to start the reminder branch.
- **Configuration:** `everyHour`.
- **Outputs:** To `Fetch Calendar Appointments`.
- **Failure modes:** None typical; may not run if workflow inactive.

#### Node: Fetch Calendar Appointments
- **Type/role:** Google Calendar “getAll” events.
- **Configuration choices:**
  - Calendar: `primary`
  - Operation: `getAll`
  - `returnAll: true` (fetches all matching results)
- **Credentials:** Google Calendar OAuth2.
- **Input:** From `Hourly Schedule Trigger`.
- **Output:** To `Send Pre-Arrival Intake Email`.
- **Edge cases/failures:**
  - OAuth expired/invalid → auth error.
  - Large calendars: `returnAll` can be heavy; may need time window filters (not configured here).
  - Timezone/event filtering not configured (may email too broadly).

#### Node: Send Pre-Arrival Intake Email
- **Type/role:** Gmail send message.
- **Configuration choices (current state):**
  - `toList`: `user@example.com` (static placeholder)
  - `subject`: `hii` (placeholder)
  - `message`: `cdcd` (placeholder)
- **Credentials:** Gmail OAuth2.
- **Input:** From `Fetch Calendar Appointments` (but no mapping from event attendee/patient email is implemented).
- **Edge cases/failures:**
  - Placeholder recipient will not reach actual patients unless replaced with expressions.
  - Gmail API quota / “From” restrictions depending on account.
  - If Calendar returns multiple items, Gmail node will execute per item (could spam unintentionally without filtering).

---

### Block 3 — Intake Reception (Google Sheets Trigger)
**Overview:** Polls a Google Sheet tied to form responses to detect new intake entries.  
**Nodes involved:** `Watch Intake Form Responses`

#### Node: Watch Intake Form Responses
- **Type/role:** Google Sheets Trigger (polling).
- **Configuration choices:**
  - Polling: every minute
  - Document: “Patient pre-arrival intake form (Responses)” (by ID)
  - Sheet: “Form Responses 1”
- **Credentials:** Google Sheets Trigger OAuth2.
- **Output:** To `AI Risk Assessment Agent`.
- **Edge cases/failures:**
  - Polling triggers can miss/duplicate if rows edited; depends on trigger’s internal checkpointing.
  - Column names include extra spaces (e.g., `'  Date of Birth  '`)—expressions must match exactly.

---

### Block 4 — AI Risk Assessment (LangChain Agent + Azure OpenAI Chat Model)
**Overview:** Builds a clinical-intake-safe prompt, asks the model for strict JSON risk triage, returns the model output for downstream parsing.  
**Nodes involved:** `Azure OpenAI Chat Model1`, `AI Risk Assessment Agent`

#### Node: Azure OpenAI Chat Model1
- **Type/role:** LangChain Azure OpenAI chat model provider node.
- **Configuration choices:**
  - Model deployment name: `gpt-4o-mini`
- **Credentials:** Azure OpenAI API (key/endpoint/deployment in credential).
- **Connections:**
  - Provides `ai_languageModel` input to `AI Risk Assessment Agent`.
- **Edge cases/failures:**
  - Wrong deployment name or region endpoint mismatch → 404/401.
  - Content filtering / policy blocks can interrupt completion depending on tenant settings.
  - Rate limits.

#### Node: AI Risk Assessment Agent
- **Type/role:** LangChain Agent node that composes prompt and calls the connected model.
- **Configuration choices:**
  - **Prompt text** uses expressions from the intake row:
    - `{{ $json["Name"] }}`
    - `{{ $json['  Date of Birth  '] }}`
    - `{{ $json['  Reason for Visit  '] }}`
    - `{{ $json['  Please describe your current symptoms  '] }}`
    - `{{ $json['  Do you have any known allergies?  if yes, please describe'] }}`
  - **System message**: clinical safety constraints (no diagnosis, no treatment), urgency rubric, and **required strict JSON schema**:
    - `risk_level` in {Low, Medium, High, Emergency}
    - flags: `requires_immediate_attention`, `allergy_flag`, `medication_conflict_risk`
    - `key_concerns` list
    - `doctor_preparation_notes` string
    - `confidence_score` 0–100
- **Inputs:**
  - Main input: from `Watch Intake Form Responses`.
  - Language model input: from `Azure OpenAI Chat Model1` via `ai_languageModel`.
- **Output:** To `Normalize AI Output`.
- **Edge cases/failures:**
  - If sheet column headers differ (spaces/case), prompt fields become empty.
  - Model may still output non-JSON (despite instruction) → downstream parsing fails.
  - Safety instruction conflicts: e.g., patient asks for treatment; should still return JSON only.

---

### Block 5 — Normalize + Store + Triage + Slack
**Overview:** Parses the agent’s JSON string, normalizes types, writes to a results sheet, then alerts Slack for “High” risk.  
**Nodes involved:** `Normalize AI Output`, `Save Assessment to Sheet`, `Check Risk Level`, `Notify Doctor via Slack`

#### Node: Normalize AI Output
- **Type/role:** Code node (JavaScript) to validate/parse/normalize AI output.
- **Key logic:**
  - Reads `const rawOutput = $json.output;`
  - Throws if missing output.
  - `JSON.parse(rawOutput)` with error thrown if invalid JSON.
  - Normalizes:
    - `risk_level` trimmed (default “Unknown”)
    - flags coerced to boolean via `Boolean(...)`
    - `confidence_score` coerced to Number; defaults to `0` if NaN
  - Returns one item with normalized fields:
    - `risk_level`, `requires_immediate_attention`, `allergy_flag`, `medication_conflict_risk`, `key_concerns`, `doctor_preparation_notes`, `confidence_score`
- **Input:** From `AI Risk Assessment Agent`.
- **Output:** To `Save Assessment to Sheet`.
- **Edge cases/failures:**
  - If the Agent node outputs the model content under a different property than `output`, this will throw.
  - `Boolean("false")` becomes `true` (string truthiness). If the model outputs `"false"` (string), booleans will be wrong. Prefer explicit string handling if needed.
  - `key_concerns` could be non-array; currently passed through as-is.

#### Node: Save Assessment to Sheet
- **Type/role:** Google Sheets append-or-update risk assessment results.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document: same spreadsheet ID as the form responses
  - Sheet: `Sheet1` (results/triage sheet)
  - Matching column: `Name` (used to update existing rows)
  - Column mapping uses expressions:
    - `Name`: `={{ $('Watch Intake Form Responses').item.json.Name }}`
    - `Risk level`: `={{ $json.risk_level }}`
    - `allergy flag`: `={{ $json.allergy_flag }}`
    - `key concerns`: `={{ $json.key_concerns }}`
    - `confidence score`: `={{ $json.confidence_score }}`
    - `immidiate action`: `={{ $json.requires_immediate_attention }}`
    - `Doctor prepartion notes`: `={{ $json.doctor_preparation_notes }}`
- **Inputs:** From `Normalize AI Output`.
- **Output:** To `Check Risk Level`.
- **Edge cases/failures:**
  - Matching on `Name` can overwrite the wrong patient if two patients share the same name; a unique response ID / timestamp is safer.
  - Column names contain typos (“immidiate”, “prepartion”)—must match actual sheet headers exactly.
  - If `key concerns` is an array, Sheets may store it as `[object Object]` or comma-joined depending on node behavior; often best to `JSON.stringify(...)` or join lines.

#### Node: Check Risk Level
- **Type/role:** IF node for routing high-risk cases to Slack.
- **Configuration choices:**
  - Condition (string equality):
    - `value1 = {{ $json["Risk level"] }}`
    - `value2 = High`
- **Input:** From `Save Assessment to Sheet`.
- **Output:**
  - True branch → `Notify Doctor via Slack`
  - False branch → not connected (no action)
- **Important integration note:** Depending on the Google Sheets node’s output schema, `Risk level` may not be present at the top level after `appendOrUpdate`. If the node returns metadata instead of the written row, this condition will never match. A safer check is to branch on the normalized value **before** writing, or carry forward `risk_level` explicitly.
- **Edge cases/failures:**
  - Case differences (“High ”, “HIGH”) will fail exact match unless normalized.

#### Node: Notify Doctor via Slack
- **Type/role:** Slack message post for high-risk alerts.
- **Configuration choices:**
  - Channel: `YOUR_SLACK_CHANNEL` (must be replaced)
  - OAuth2 auth
  - Text template references fields:
    - `{{$json["Name"]}}`
    - `{{ $json["key concerns"] }}`
    - `{{ $json["allergy flag"] }}`
    - `{{ $json["Doctor prepartion notes"] }}`
- **Input:** From `Check Risk Level` (true branch).
- **Edge cases/failures:**
  - Channel placeholder causes failure or posting to nowhere (depending on Slack node validation).
  - Field naming mismatch: Slack node expects the row fields to exist exactly as referenced (`key concerns` etc.). If the prior node output does not contain them, message will be blank.
  - Missing `chat:write` scope or workspace restrictions → auth/permission error.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Overview | stickyNote | Workflow documentation and setup guidance |  |  | (content) ## 🏥 Patient Pre-Arrival Intake Automation … (includes setup steps and customization) |
| Section: Appointment Reminder | stickyNote | Documents hourly email reminder branch |  |  | (content) ## 📅 Appointment Reminder Branch … |
| Section: Intake & AI Analysis | stickyNote | Documents intake trigger + AI analysis branch |  |  | (content) ## 🤖 Patient Intake & AI Risk Analysis … |
| Section: Risk Triage & Notification | stickyNote | Documents sheet storage + Slack escalation |  |  | (content) ## 📊 Risk Triage & Doctor Notification … |
| ⚠️ Warning: Slack Config | stickyNote | Warns about Slack channel/scopes |  |  | (content) Replace `YOUR_SLACK_CHANNEL`… needs `chat:write`… |
| ⚠️ Warning: Azure OpenAI | stickyNote | Warns about Azure OpenAI credentials/deployment |  |  | (content) valid Azure OpenAI API key + `gpt-4o-mini` deployment… |
| Hourly Schedule Trigger | cron | Scheduled trigger (hourly) |  | Fetch Calendar Appointments | ## 📅 Appointment Reminder Branch … |
| Fetch Calendar Appointments | googleCalendar | Fetch upcoming appointments/events | Hourly Schedule Trigger | Send Pre-Arrival Intake Email | ## 📅 Appointment Reminder Branch … |
| Send Pre-Arrival Intake Email | gmail | Sends intake email to patient (currently placeholder) | Fetch Calendar Appointments |  | ## 📅 Appointment Reminder Branch … |
| Watch Intake Form Responses | googleSheetsTrigger | Polls for new form response rows |  | AI Risk Assessment Agent | ## 🤖 Patient Intake & AI Risk Analysis … |
| Azure OpenAI Chat Model1 | lmChatAzureOpenAi | Azure OpenAI chat model provider for agent |  | AI Risk Assessment Agent (ai_languageModel) | ⚠️ **Azure OpenAI Credentials Required** … |
| AI Risk Assessment Agent | langchain.agent | Produces strict JSON risk assessment from intake text | Watch Intake Form Responses; Azure OpenAI Chat Model1 | Normalize AI Output | ## 🤖 Patient Intake & AI Risk Analysis … |
| Normalize AI Output | code | Parses/normalizes model JSON | AI Risk Assessment Agent | Save Assessment to Sheet | ## 🤖 Patient Intake & AI Risk Analysis … |
| Save Assessment to Sheet | googleSheets | Append/update triage results in Sheet1 | Normalize AI Output | Check Risk Level | ## 📊 Risk Triage & Doctor Notification … |
| Check Risk Level | if | Routes High risk to Slack | Save Assessment to Sheet | Notify Doctor via Slack (true) | ## 📊 Risk Triage & Doctor Notification … |
| Notify Doctor via Slack | slack | Slack alert for high-risk patients | Check Risk Level (true) |  | ⚠️ **Slack Credentials Required** … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Patient Pre-Arrival Intake Automation”** (inactive by default while configuring credentials).

### A) Appointment Reminder Branch
2. Add **Cron** node named **“Hourly Schedule Trigger”**:
   - Mode: **Every Hour**

3. Add **Google Calendar** node named **“Fetch Calendar Appointments”**:
   - Resource/Operation: **Event → Get All**
   - Calendar: **primary**
   - Return All: **true**
   - Credentials: **Google Calendar OAuth2**

4. Add **Gmail** node named **“Send Pre-Arrival Intake Email”**:
   - Resource: **Message**
   - Operation: **Send**
   - Set **To**, **Subject**, **Message**
   - Credentials: **Gmail OAuth2**
   - (Recommended) Replace static `toList` with an expression from the calendar event (e.g., attendee email) once your event schema is confirmed.

5. Connect:
   - `Hourly Schedule Trigger` → `Fetch Calendar Appointments` → `Send Pre-Arrival Intake Email`

### B) Intake Trigger + AI Analysis Branch
6. Add **Google Sheets Trigger** node named **“Watch Intake Form Responses”**:
   - Polling interval: **Every Minute**
   - Select Spreadsheet (Document ID): your intake responses file
   - Sheet: **Form Responses 1**
   - Credentials: **Google Sheets Trigger OAuth2**

7. Add **Azure OpenAI Chat Model** (LangChain) node named **“Azure OpenAI Chat Model1”**:
   - Model/deployment: **gpt-4o-mini**
   - Credentials: **Azure OpenAI API** (endpoint, key, deployment configured in credential)

8. Add **AI Agent** (LangChain Agent) node named **“AI Risk Assessment Agent”**:
   - Prompt type: **Define**
   - **Text prompt:** build from the intake row fields (ensure the header strings match your sheet exactly, including spaces).
   - **System message:** include the strict JSON schema and safety constraints (no diagnosis, no treatment).
   - Connect **Azure model** to the agent via the **AI Language Model** connection.

9. Add **Code** node named **“Normalize AI Output”**:
   - Paste logic to:
     - read the agent output string,
     - `JSON.parse`,
     - normalize `risk_level`, booleans, `confidence_score`,
     - return a clean JSON object.

10. Add **Google Sheets** node named **“Save Assessment to Sheet”**:
   - Operation: **Append or Update**
   - Spreadsheet: same file (or a separate triage file)
   - Sheet: **Sheet1**
   - Matching column: **Name** (recommended to use a unique ID instead)
   - Map columns to the normalized fields (and Name from the trigger item).

11. Add **IF** node named **“Check Risk Level”**:
   - Condition: String equals
   - Left value: the stored risk field (ensure it exists in the incoming JSON at this point)
   - Right value: `High`
   - (Recommended) Normalize casing/whitespace or use “contains / regex” for robustness.

12. Add **Slack** node named **“Notify Doctor via Slack”**:
   - Authentication: **OAuth2**
   - Channel: replace `YOUR_SLACK_CHANNEL` with your real channel ID
   - Scope: ensure Slack app includes **chat:write**
   - Message text: include patient name, key concerns, allergy flag, doctor notes.

13. Connect:
   - `Watch Intake Form Responses` → `AI Risk Assessment Agent` → `Normalize AI Output` → `Save Assessment to Sheet` → `Check Risk Level`
   - `Check Risk Level (true)` → `Notify Doctor via Slack`
   - `Azure OpenAI Chat Model1 (ai_languageModel)` → `AI Risk Assessment Agent (ai_languageModel)`

### Credential checklist
14. Configure credentials in n8n:
   - Google Calendar OAuth2
   - Gmail OAuth2
   - Google Sheets Trigger OAuth2
   - Google Sheets OAuth2 (write access)
   - Azure OpenAI API (endpoint + key + deployment)
   - Slack OAuth2 (with `chat:write`)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Slack requires a valid channel ID (replace `YOUR_SLACK_CHANNEL`) and `chat:write` scope; missing config may prevent alerts. | From sticky note “⚠️ Warning: Slack Config” |
| Azure OpenAI node requires valid credentials and an active `gpt-4o-mini` deployment; bad config halts intake assessment. | From sticky note “⚠️ Warning: Azure OpenAI” |
| The intake sheet headers include leading/trailing spaces in several field names; expressions must match exactly or values will be empty. | Observed in AI Agent prompt mappings |
| The “Send Pre-Arrival Intake Email” node is currently configured with placeholder recipient/subject/body; it does not yet email the actual appointment attendee. | Observed in Gmail node configuration |
| Risk check/Slack message depend on fields output by the Google Sheets “appendOrUpdate” node; verify that the node returns row fields like `Risk level`, otherwise branch earlier on normalized `risk_level`. | Integration caveat between “Save Assessment to Sheet” → “Check Risk Level” |

