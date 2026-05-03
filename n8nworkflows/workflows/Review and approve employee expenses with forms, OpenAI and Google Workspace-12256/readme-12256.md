Review and approve employee expenses with forms, OpenAI and Google Workspace

https://n8nworkflows.xyz/workflows/review-and-approve-employee-expenses-with-forms--openai-and-google-workspace-12256


# Review and approve employee expenses with forms, OpenAI and Google Workspace

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Review and approve employee expenses with forms, OpenAI and Google Workspace

**Purpose / use case:**  
Automates an employee expense reimbursement flow end-to-end: employees submit an expense via an n8n Form, the expense is logged to Google Sheets, an OpenAI-based agent evaluates whether the amount is fully justified by the description, the manager receives an email with approve/reject buttons (send-and-wait), and the employee is notified. If the manager rejects, the workflow uses Google Calendar + an AI “booking agent” to pick the earliest available 10‑minute slot on the next business day and emails the employee the discussion time.

### Logical blocks
**1.1 Intake & logging**  
Form submission → append record into Google Sheets.

**1.2 AI expense assessment (justification check)**  
AI Agent (OpenAI model + structured output parsing) evaluates whether the full amount is justified → outputs a YES/NO decision.

**1.3 Route to manager approval email**  
An IF node routes to one of two “send-and-wait” manager emails (valid vs needs clarification).

**1.4 Post-manager decision handling**  
Manager approval → employee approval email.  
Manager rejection → calendar availability search (AI agent + calendar tools) → employee discussion email with selected slot.

---

## 2. Block-by-Block Analysis

### 2.1 Intake & logging
**Overview:** Captures expense data from an n8n Form and logs it into a Google Sheet for finance recordkeeping and downstream referencing.  
**Nodes involved:** `On Expense Form Submission`, `Append row in sheet`

#### Node: On Expense Form Submission
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point; hosts and receives a form submission.
- **Configuration (interpreted):**
  - Form title: **Employee Expense Reimbursement Form**
  - Description: prompts employee to submit expense details.
  - Fields (all required):
    - Employee Name (text)
    - Employee Email (text)
    - Department (dropdown: Sales, Marketing, HR, Project Management)
    - Expense Date (date)
    - Expense Amount (number)
    - Currency (dropdown: INR, USD, EU)
    - Expense Description (textarea)
    - Manager Email (text)
- **Key variables produced:** Payload keys match labels, e.g. `$json['Employee Name']`, `$json['Expense Amount']`, `$json.Department`.
- **Outputs:** Main → `Append row in sheet`
- **Edge cases / failures:**
  - Field label changes will break downstream expressions that reference exact labels (e.g., `Employee Name`).
  - Date formatting depends on n8n form output; downstream nodes treat it as string.

#### Node: Append row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — Persists each submission as a new row.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Document: `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - Sheet tab: `gid=0`
  - Column mapping uses expressions from the form payload:
    - Employee Name ← `{{$json['Employee Name']}}`
    - Employee Email ← `{{$json['Employee Email']}}`
    - Department ← `{{$json.Department}}`
    - Expense Date ← `{{$json['Expense Date']}}`
    - Expense Amount ← `{{$json['Expense Amount']}}`
    - Currency ← `{{$json.Currency}}`
    - Expense Description ← `{{$json['Expense Description']}}`
    - Manager Email ← `{{$json['Manager Email']}}`
  - Type conversion disabled (keeps values as-is).
- **Outputs:** Main → `AI Agent`
- **Edge cases / failures:**
  - Google auth/permission errors (Sheet not shared with credential user).
  - Column header mismatch: mapping assumes exact column IDs/names exist.
  - `Expense Amount` is treated as string in schema; numeric handling must be done elsewhere if needed.

---

### 2.2 AI expense assessment (justification check)
**Overview:** Uses an OpenAI chat model to analyze whether the expense description fully justifies the claimed total and outputs a structured decision.  
**Nodes involved:** `AI Agent`, `OpenAI Chat Model`, `Output Parser`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides the LLM for the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini**
  - Default options (no special temperature/tools configured here).
- **Connections:** `ai_languageModel` → `AI Agent`
- **Edge cases / failures:** missing/invalid OpenAI credentials; model availability; rate limits.

#### Node: Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Forces structured JSON output from the agent.
- **Configuration (interpreted):**
  - Schema example:
    ```json
    { "Summary": "", "Opinion": "", "Decesion": "" }
    ```
  - Note the key is spelled **"Decesion"** (typo), and downstream logic depends on this spelling.
- **Connections:** `ai_outputParser` → `AI Agent`
- **Edge cases / failures:**
  - If the model outputs invalid JSON or different keys, parsing fails or produces missing fields.
  - Spelling mismatch (Decision vs Decesion) is a common modification hazard.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Runs the reasoning prompt and returns parsed output.
- **Configuration (interpreted):**
  - Prompt injects form/sheet fields (name, dept, date, amount, description).
  - Requires the agent to:
    - Extract line-item amounts from the description
    - Verify they sum exactly to the total
    - Mark **NO** if any part is unclear/unjustified
    - Produce exact sections: Summary, AI Opinion, Is Expense Appropriate (YES/NO)
  - `hasOutputParser = true` (paired to `Output Parser`)
- **Inputs:** Main from `Append row in sheet`; model input from `OpenAI Chat Model`; parser from `Output Parser`.
- **Outputs:** Main → `If`
- **Edge cases / failures:**
  - If the description includes no explicit amounts, the agent should likely return NO per rules; but LLM variability can still occur.
  - Currency mismatch (e.g., “EU” vs “EUR”) is not validated.
  - The prompt says “Generate the output in the exact format below” but the structured parser expects JSON keys; any divergence can break.

---

### 2.3 Route to manager approval email
**Overview:** Routes the process based on the AI’s YES/NO into separate manager emails: one stating it looks valid, one stating it needs clarification; both are “send-and-wait” approvals.  
**Nodes involved:** `If`, `Send Email to Manager for Approval`, `Send Email to Manager for Approval1`

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — Branching based on AI decision.
- **Configuration (interpreted):**
  - Condition: `{{$json.output.Decesion}}` **equals** `"YES"`
  - True branch → “appears valid” manager email
  - False branch → “may require clarification” manager email
- **Inputs:** Main from `AI Agent`
- **Outputs:** 
  - True → `Send Email to Manager for Approval`
  - False → `Send Email to Manager for Approval1`
- **Edge cases / failures:**
  - If `output.Decesion` is missing/null (parser failure), condition evaluates false → sends clarification path.
  - Decision values must match exactly `"YES"` (case sensitive). Any extra whitespace/newlines can break routing.

#### Node: Send Email to Manager for Approval
- **Type / role:** `n8n-nodes-base.gmail` — Sends an approval email and waits for manager action.
- **Configuration (interpreted):**
  - Operation: **sendAndWait** (approval via email)
  - To: `{{$('On Expense Form Submission').item.json['Manager Email']}}`
  - Subject: “Expense Review Update”
  - Body: HTML summary referencing data via `$('Append row in sheet').item.json[...]`
  - Approval options: `approvalType: "double"` and a reject label “Reject”
- **Input:** True branch from `If`
- **Output:** Main → `If1` (contains approval result under `$json.data.approved`)
- **Edge cases / failures:**
  - Gmail OAuth not configured; sending blocked by domain policy.
  - “sendAndWait” requires n8n to be reachable for the callback; if behind firewall without proper public URL/webhook config, approval links may fail.
  - Expression uses cross-node item references; if multiple items are processed, `.item` selection can mismatch (here it assumes single-item flow).

#### Node: Send Email to Manager for Approval1
- **Type / role:** `n8n-nodes-base.gmail` — Same as above, but wording indicates clarification needed.
- **Configuration differences:**
  - Same recipient and approval mechanism
  - Email content says the expense “may require further clarification”
- **Input:** False branch from `If`
- **Output:** Main → `If1`
- **Edge cases / failures:** same as the previous manager email node.

---

### 2.4 Post-manager decision handling (approve vs reject + scheduling)
**Overview:** If the manager approves, notify the employee. If the manager rejects, automatically find the earliest 10‑minute free slot on the next business day and notify the employee of the scheduled discussion time.  
**Nodes involved:** `If1`, `Send a message`, `Booking Agent`, `OpenAI`, `Get Events`, `Check Availability`, `Output Parser1`, `Send a message2`

#### Node: If1
- **Type / role:** `n8n-nodes-base.if` — Branches based on manager decision returned by “sendAndWait”.
- **Configuration (interpreted):**
  - Condition: `{{$json.data.approved}}` equals `true`
  - True → approved notification to employee
  - False → discussion scheduling path
- **Inputs:** From either manager email node
- **Outputs:**
  - True → `Send a message`
  - False → `Booking Agent`
- **Edge cases / failures:**
  - If manager never responds, the workflow run remains waiting (expected).
  - If email approval payload shape changes, `$json.data.approved` may not exist.

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.gmail` — Notifies the employee their expense is approved.
- **Configuration (interpreted):**
  - To: `{{$('On Expense Form Submission').item.json['Employee Email']}}`
  - Subject: “Expense Approved”
  - HTML body includes expense details from `Append row in sheet`
- **Input:** True branch from `If1`
- **Output:** End of flow
- **Edge cases / failures:** Gmail auth; invalid employee email; cross-node item reference issues.

#### Node: Booking Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Automatically selects the earliest 10-minute free slot on the next business day.
- **Configuration (interpreted):**
  - Prompt is strict: must call tools, must not ask questions, must return JSON `{ "start_time": "YYYY-MM-DD HH:MM:SS" }` or “No available slots found”.
  - Business-day rule: if tomorrow is weekend, use Monday.
  - Tooling strategy:
    1) Call **Get Events**
    2) If events exist, call **Check Availability**
- **Inputs:** False branch from `If1`; model from `OpenAI`; parser from `Output Parser1`; tools from `Get Events` and `Check Availability`.
- **Outputs:** Main → `Send a message2`
- **Edge cases / failures:**
  - The calendar tool nodes are pre-configured with “tomorrow” timeMin/timeMax expressions; these do **not** implement weekend skipping. The prompt *asks the agent* to skip weekends, but the tool parameters may still query Saturday/Sunday. This can cause incorrect “business date” behavior unless adjusted.
  - If the agent fails to output valid JSON, `Output Parser1` may fail and downstream `$json.output.start_time` may be missing.

#### Node: OpenAI
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM for the Booking Agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini**
- **Connections:** `ai_languageModel` → `Booking Agent`
- **Edge cases / failures:** OpenAI credential/rate limits/model availability.

#### Node: Get Events
- **Type / role:** `n8n-nodes-base.googleCalendarTool` — Tool exposed to the agent to list events.
- **Configuration (interpreted):**
  - Operation: `getAll`, return all events
  - Calendar: `YOUR_CALENDAR_EMAIL` (must be replaced)
  - Time window:
    - `timeMin`: tomorrow at 09:00 (string expression)
    - `timeMax`: tomorrow at 18:00
  - `returnAll: true`
- **Connections:** `ai_tool` → `Booking Agent`
- **Edge cases / failures:**
  - Calendar ID invalid; insufficient permissions.
  - Time expressions are convoluted and rely on string manipulation; timezone issues can occur.
  - As noted, weekend skipping is not encoded in these timeMin/timeMax expressions.

#### Node: Check Availability
- **Type / role:** `n8n-nodes-base.googleCalendarTool` — Tool exposed to the agent to compute availability.
- **Configuration (interpreted):**
  - Resource: calendar
  - Output format: `availability`
  - Timezone: Asia/Kolkata
  - Same timeMin/timeMax “tomorrow 09:00–18:00”
  - Calendar: `YOUR_CALENDAR_EMAIL`
- **Connections:** `ai_tool` → `Booking Agent`
- **Edge cases / failures:** same as Get Events; additionally “availability” output format assumptions.

#### Node: Output Parser1
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Parses Booking Agent JSON output.
- **Configuration (interpreted):**
  - Schema example:
    ```json
    { "start_time": "" }
    ```
- **Connections:** `ai_outputParser` → `Booking Agent`
- **Edge cases / failures:** invalid JSON or unexpected key names.

#### Node: Send a message2
- **Type / role:** `n8n-nodes-base.gmail` — Notifies employee that expense is not approved and provides meeting slot.
- **Configuration (interpreted):**
  - To: `{{$('Append row in sheet').item.json['Employee Email']}}`
  - Subject: “Discussion Required Regarding Leave Request” (likely a copy/paste mistake; content is about expense)
  - Message includes meeting details: `{{ $json.output.start_time }}`
  - `appendAttribution: false`
- **Input:** Main from `Booking Agent`
- **Output:** End of flow
- **Edge cases / failures:**
  - If `output.start_time` is missing, email will show blank.
  - Subject mismatch may confuse recipients; should be corrected to an expense-related subject.

---

### 2.5 Documentation notes (sticky notes)
**Overview:** Sticky notes describe the overall intent and the 3-step structure (capture/review, manager decision, notify/schedule).  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note7`  
(They do not execute; they document.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On Expense Form Submission | n8n-nodes-base.formTrigger | Entry point form intake | — | Append row in sheet | ## Expense Approval System\n\nThis workflow automates employee expense reimbursements from submission to approval or clarification scheduling. It uses AI to review expense details, generate clear summaries, and route decisions automatically, while maintaining clean records for finance and management teams.\n\n### How it works\n- Employees submit expenses using a structured form\n- AI reviews the expense and generates a justification summary\n- The summary is emailed to the manager for approve/reject action\n- Approved expenses notify the employee automatically\n- Rejected or unclear expenses trigger a short discussion scheduling\n\n### Setup steps\n- Configure the expense submission form\n- Connect OpenAI for expense review\n- Connect Google Sheets for logging\n- Connect Gmail for approvals and notifications\n- Connect Google Calendar for discussions\n- Activate and test the workflow\n |
| Append row in sheet | n8n-nodes-base.googleSheets | Log submission to Google Sheets | On Expense Form Submission | AI Agent | ## Step 1: Capture, summarize, and request approval\nThe employee submits the expense form, an AI agent reviews the details, and a professional summary is emailed to the manager using a send-and-wait approval step. |
| Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parsing for expense AI output | — (AI parser input) | AI Agent (ai_outputParser) | ## Step 1: Capture, summarize, and request approval\nThe employee submits the expense form, an AI agent reviews the details, and a professional summary is emailed to the manager using a send-and-wait approval step. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for expense review agent | — | AI Agent (ai_languageModel) | ## Step 1: Capture, summarize, and request approval\nThe employee submits the expense form, an AI agent reviews the details, and a professional summary is emailed to the manager using a send-and-wait approval step. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Evaluate justification and decide YES/NO | Append row in sheet | If | ## Step 1: Capture, summarize, and request approval\nThe employee submits the expense form, an AI agent reviews the details, and a professional summary is emailed to the manager using a send-and-wait approval step. |
| If | n8n-nodes-base.if | Route based on AI decision | AI Agent | Send Email to Manager for Approval; Send Email to Manager for Approval1 | ## Step 1: Capture, summarize, and request approval\nThe employee submits the expense form, an AI agent reviews the details, and a professional summary is emailed to the manager using a send-and-wait approval step. |
| Send Email to Manager for Approval | n8n-nodes-base.gmail | Send-and-wait manager approval (valid path) | If (true) | If1 | ## Step 2: Manager reviews and responds\nThe manager reviews the expense summary directly from email and approves or rejects the reimbursement request. |
| Send Email to Manager for Approval1 | n8n-nodes-base.gmail | Send-and-wait manager approval (clarification path) | If (false) | If1 | ## Step 2: Manager reviews and responds\nThe manager reviews the expense summary directly from email and approves or rejects the reimbursement request. |
| If1 | n8n-nodes-base.if | Branch on manager approve/reject | Send Email to Manager for Approval; Send Email to Manager for Approval1 | Send a message; Booking Agent | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Send a message | n8n-nodes-base.gmail | Notify employee of approval | If1 (true) | — | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| OpenAI | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for booking agent | — | Booking Agent (ai_languageModel) | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Get Events | n8n-nodes-base.googleCalendarTool | Calendar tool: list events for next day window | — (tool) | Booking Agent (ai_tool) | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Check Availability | n8n-nodes-base.googleCalendarTool | Calendar tool: compute availability | — (tool) | Booking Agent (ai_tool) | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parsing for booking output | — (AI parser input) | Booking Agent (ai_outputParser) | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Booking Agent | @n8n/n8n-nodes-langchain.agent | Choose earliest 10-min free slot | If1 (false) | Send a message2 | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Send a message2 | n8n-nodes-base.gmail | Notify employee of rejection + meeting slot | Booking Agent | — | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | ## Expense Approval System\n\nThis workflow automates employee expense reimbursements from submission to approval or clarification scheduling. It uses AI to review expense details, generate clear summaries, and route decisions automatically, while maintaining clean records for finance and management teams.\n\n### How it works\n- Employees submit expenses using a structured form\n- AI reviews the expense and generates a justification summary\n- The summary is emailed to the manager for approve/reject action\n- Approved expenses notify the employee automatically\n- Rejected or unclear expenses trigger a short discussion scheduling\n\n### Setup steps\n- Configure the expense submission form\n- Connect OpenAI for expense review\n- Connect Google Sheets for logging\n- Connect Gmail for approvals and notifications\n- Connect Google Calendar for discussions\n- Activate and test the workflow\n |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Step 1: Capture, summarize, and request approval\nThe employee submits the expense form, an AI agent reviews the details, and a professional summary is emailed to the manager using a send-and-wait approval step. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Step 2: Manager reviews and responds\nThe manager reviews the expense summary directly from email and approves or rejects the reimbursement request. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Step 3: Notify employee or schedule discussion\nIf approved, the employee is notified by email. If rejected, the system checks calendar availability and schedules a discussion automatically. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Form Trigger**
   - Add node: **Form Trigger**
   - Title: “Employee Expense Reimbursement Form”
   - Add required fields:
     - Employee Name (text)
     - Employee Email (text)
     - Department (dropdown: Sales, Marketing, HR, Project Management)
     - Expense Date (date)
     - Expense Amount (number)
     - Currency (dropdown: INR, USD, EU)
     - Expense Description (textarea)
     - Manager Email (text)

2) **Add Google Sheets logging**
   - Add node: **Google Sheets**
   - Credentials: Google account with access to the target spreadsheet
   - Operation: **Append**
   - Document ID: set to your spreadsheet ID
   - Sheet: select the correct tab (e.g., first sheet)
   - Map columns to the incoming form fields using expressions matching the field labels.
   - Connect: **Form Trigger → Google Sheets**

3) **Add structured output parser for expense review**
   - Add node: **Structured Output Parser** (LangChain)
   - Schema example:
     - `Summary` (string)
     - `Opinion` (string)
     - `Decesion` (string)  *(keep this exact spelling if you plan to reuse the same IF expression)*
   - Connect parser to the agent later via `ai_outputParser`.

4) **Add OpenAI chat model for expense review**
   - Add node: **OpenAI Chat Model**
   - Credentials: OpenAI API key
   - Model: `gpt-4.1-mini`
   - Connect model to the agent later via `ai_languageModel`.

5) **Add Expense AI Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Prompt: paste the expense-review prompt (include variables for name/department/date/amount/description).
   - Enable “Use Output Parser” / structured output.
   - Connect:
     - **Google Sheets → AI Agent** (main)
     - **OpenAI Chat Model → AI Agent** (`ai_languageModel`)
     - **Output Parser → AI Agent** (`ai_outputParser`)

6) **Route based on AI decision**
   - Add node: **If**
   - Condition (String equals):
     - Left: `{{$json.output.Decesion}}`
     - Right: `YES`
   - Connect: **AI Agent → If**

7) **Manager approval email (valid path)**
   - Add node: **Gmail**
   - Credentials: Gmail OAuth2 for a sender mailbox
   - Operation: **Send and Wait**
   - To: `{{$('On Expense Form Submission').item.json['Manager Email']}}`
   - Subject: “Expense Review Update”
   - Body: include expense details from the sheet node output (or directly from form).
   - Configure approval buttons (approve/reject).
   - Connect: **If (true) → this Gmail node**

8) **Manager approval email (clarification path)**
   - Duplicate the previous Gmail node, adjust wording (“may require further clarification”).
   - Connect: **If (false) → this Gmail node**

9) **Branch on manager response**
   - Add node: **If**
   - Condition (Boolean equals):
     - Left: `{{$json.data.approved}}`
     - Right: `true`
   - Connect both manager email nodes → this IF node.

10) **Employee “approved” notification**
   - Add node: **Gmail** (send)
   - To: employee email (from form or sheet)
   - Subject: “Expense Approved”
   - Body: approval confirmation + expense details
   - Connect: **Manager-response IF (true) → Employee approved email**

11) **Create calendar tools for scheduling (rejection path)**
   - Add node: **Google Calendar Tool** (Get Events)
     - Operation: Get All events
     - Calendar: your calendar ID/email
     - timeMin/timeMax: set a window (09:00–18:00) for the target date
   - Add node: **Google Calendar Tool** (Check Availability)
     - Resource: calendar, output: availability
     - Same calendar and window
     - Set timezone (e.g., Asia/Kolkata)
   - Credentials: Google account with calendar access

12) **Add booking output parser**
   - Add node: **Structured Output Parser**
   - Schema: `{ "start_time": "" }`

13) **Add OpenAI chat model for booking**
   - Add node: **OpenAI Chat Model**
   - Model: `gpt-4.1-mini`
   - Credentials: same OpenAI key (or separate)

14) **Add Booking Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Prompt: paste the availability/slot-selection instructions.
   - Connect:
     - **Manager-response IF (false) → Booking Agent** (main)
     - **OpenAI (booking) → Booking Agent** (`ai_languageModel`)
     - **Booking Output Parser → Booking Agent** (`ai_outputParser`)
     - **Get Events → Booking Agent** (`ai_tool`)
     - **Check Availability → Booking Agent** (`ai_tool`)

15) **Employee “rejected + discussion time” email**
   - Add node: **Gmail** (send)
   - To: employee email
   - Subject: change to an expense-related subject (recommended)
   - Include meeting time with: `{{$json.output.start_time}}`
   - Connect: **Booking Agent → this email**

16) **Activation checklist**
   - Replace placeholders:
     - `YOUR_GOOGLE_SHEET_ID`
     - `YOUR_CALENDAR_EMAIL`
   - Ensure n8n “Public URL” / webhook base is configured so **Gmail send-and-wait** approval links work.
   - Test with a sample submission end-to-end.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Expense Approval System sticky note describing the 3-step flow and setup checklist (OpenAI, Sheets, Gmail, Calendar). | In-canvas documentation (Sticky Note) |
| Step 1 note: capture, summarize, request approval (send-and-wait). | In-canvas documentation (Sticky Note1) |
| Step 2 note: manager reviews and responds via email approval. | In-canvas documentation (Sticky Note2) |
| Step 3 note: notify employee or schedule discussion automatically. | In-canvas documentation (Sticky Note7) |
| Potential inconsistency: Booking tools’ timeMin/timeMax compute “tomorrow” but do not implement weekend skipping; prompt expects weekend skipping. | Design consideration (adjust timeMin/timeMax expressions or add a date-calculation node) |
| Email subject mismatch in rejection email: “Discussion Required Regarding Leave Request” while content is about expenses. | Should be corrected for clarity (Send a message2) |