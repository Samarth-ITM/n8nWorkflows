Automated resume screening & interview scheduling with Gmail, GPT & Airtable

https://n8nworkflows.xyz/workflows/automated-resume-screening---interview-scheduling-with-gmail--gpt---airtable-11912


# Automated resume screening & interview scheduling with Gmail, GPT & Airtable

## 1. Workflow Overview

**Workflow name:** Automated Hiring Assistant  
**Stated title:** Automated resume screening & interview scheduling with Gmail, GPT & Airtable  
**Purpose:** Automatically detect job-application emails in Gmail, extract and analyze attached resumes with an OpenAI model against predefined roles, store candidate evaluation in Airtable, and (for strong candidates) schedule an interview in Google Calendar and email confirmation to the candidate.

### 1.1 Intake & Classification (Gmail → AI “is this a job application?”)
- Watches Gmail for new messages (with attachments downloaded).
- Sends subject/body to an LLM classifier that returns **ONLY** `YES` or `NO`.
- Proceeds only if not `NO` (note: current logic accepts anything other than exact `NO`).

### 1.2 Resume Retrieval, Storage, and Text Extraction
- Fetches the full Gmail message and downloads attachments.
- Uploads the first attachment binary (`data0`) to Google Drive.
- Extracts text from the PDF attachment.

### 1.3 Candidate Fit Evaluation (Roles → LLM JSON)
- Adds a fixed list of “Available Positions”.
- Sends extracted resume text + available roles to an LLM that outputs **structured JSON** (recommended role, score, strengths, gaps, etc.).

### 1.4 Qualification Gate & Interview Slot Selection (Score ≥ 8)
- If fit score ≥ 8:
  - Computes next business day and a 09:00–18:00 time window.
  - Uses an AI Agent with Google Calendar “Get Events” + “Check Availability” tools to return the earliest free 1-hour slot (structured output).

### 1.5 Scheduling, Notification, and Persistence
- Creates a Google Calendar interview event with attendees.
- Emails the candidate a confirmation message with formatted date/time.
- Creates an Airtable record (candidate evaluation) after sending the email.
- Additionally, a second Airtable “Create record3” is connected on the **false** branch of the score check (see notes in Block 3/4).

---

## 2. Block-by-Block Analysis

### Block 1 — Detect and collect job-application data
**Overview:** Monitors Gmail, classifies incoming emails as job applications, and if accepted retrieves the email and attachments for processing.  
**Nodes involved:** Gmail Trigger1, Message a model2, If2, Get a message1, Upload file1, Extract from File1

#### 2.1 Gmail Trigger1
- **Type / role:** `gmailTrigger` — polling trigger for new Gmail messages.
- **Key configuration:**
  - Polling every minute.
  - `downloadAttachments: true` and attachments stored under binary properties prefixed with `data` (e.g., `data0`, `data1`).
  - “Simple” mode disabled (so you get richer payload, including `textAsHtml`, address objects, etc.).
- **Input/Output:**
  - **Entry point** of workflow.
  - Output → Message a model2.
- **Failure/edge cases:**
  - Gmail OAuth token expiry / missing scopes (read access).
  - High volume inbox can lead to missed/duplicated polling windows depending on Gmail trigger settings.
  - Attachments might not be present; downstream assumes `data0` exists.

#### 2.2 Message a model2
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — sends a single prompt to OpenAI to classify the email.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`.
  - Prompt forces strict output: **ONLY** `YES` or `NO`.
  - Uses:
    - `{{ $json.subject }}`
    - `{{ $json.textAsHtml }}`
- **Input/Output:**
  - Input from Gmail Trigger1.
  - Output → If2.
- **Failure/edge cases:**
  - If the model returns whitespace, punctuation, or `"YES "` the next IF may behave unexpectedly.
  - If `subject` or `textAsHtml` is missing (some messages), the classifier may degrade.

#### 2.3 If2
- **Type / role:** `if` — gates processing based on classifier output.
- **Configuration choices:**
  - Condition: `{{ $json.message.content }} != "NO"`
  - **Important logic implication:** Anything not exactly `NO` passes (including `YES`, `Yes`, empty string, or an error message).
- **Input/Output:**
  - Input from Message a model2.
  - True output → Get a message1.
  - False output: not connected (workflow ends).
- **Failure/edge cases:**
  - Misclassification acceptance: If OpenAI returns something like `"NO."`, the condition passes and triggers full pipeline.
  - Consider switching to equals `YES` with trimming/lowercasing.

#### 2.4 Get a message1
- **Type / role:** `gmail` — fetch a specific Gmail message by ID, including attachments.
- **Configuration choices:**
  - Operation: **Get** message.
  - Message ID: `{{ $('Gmail Trigger1').item.json.id }}`
  - Downloads attachments with prefix `data`.
  - Requires Gmail OAuth2 credentials.
- **Input/Output:**
  - Input from If2.
  - Output → Upload file1 and Extract from File1 (parallel).
- **Failure/edge cases:**
  - Attachment download may fail for large files or restricted content.
  - If Trigger output doesn’t include `id` (rare), expression fails.

#### 2.5 Upload file1
- **Type / role:** `googleDrive` — uploads resume attachment to Drive for storage/audit.
- **Configuration choices:**
  - Input binary field: `data0` (first attachment).
  - File name: `Resume - {{ $json.from.value[0].address }}`
  - Drive: “My Drive”
  - Folder: “Attachments” (must be selected/valid).
- **Input/Output:**
  - Input from Get a message1.
  - No downstream connection (storage side-effect only).
- **Failure/edge cases:**
  - If the resume is not the first attachment, `data0` may be something else.
  - If email has no attachments → node errors.
  - Google Drive permissions / folder ID mismatch.

#### 2.6 Extract from File1
- **Type / role:** `extractFromFile` — extracts text from a PDF binary.
- **Configuration choices:**
  - Operation: `pdf`
  - Binary property: `data0`
- **Input/Output:**
  - Input from Get a message1.
  - Output → Available Positions1.
- **Failure/edge cases:**
  - Non-PDF resumes (DOCX) will fail.
  - Scanned PDFs may yield poor text unless OCR is used (this node is not OCR).

**Sticky note applied (Block 1):**  
“## Step 1: Detect and collect job-application data …” (from Sticky Note6)  
Also general workflow context from Sticky Note5 applies.

---

### Block 2 — Analyze the resume and evaluate candidate fit with AI
**Overview:** Adds predefined open roles, then prompts an LLM to pick the best role match and score with strict JSON output.  
**Nodes involved:** Available Positions1, Message a model3

#### 2.7 Available Positions1
- **Type / role:** `set` — defines the list of roles to evaluate against.
- **Configuration choices:**
  - Creates fields:
    - `Position 1 = "Automation Engineer"`
    - `position 2 = "Full Stack Developer"` (note lowercase “position”)
    - `Position 3 = "Javascript Developer"`
- **Input/Output:**
  - Input from Extract from File1.
  - Output → Message a model3.
- **Failure/edge cases:**
  - Inconsistent naming (`Position 1` vs `position 2`) is handled because the prompt references both, but it’s easy to break later.
  - Hard-coded roles; changing requires editing this node.

#### 2.8 Message a model3
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — resume evaluator returning JSON.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - `jsonOutput: true` enabled.
  - Prompt injects:
    - Resume text: `{{ $('Extract from File1').item.json.text }}`
    - Positions from current item: `{{ $json['Position 1'] }}`, `{{ $json['position 2'] }}`, `{{ $json['Position 3'] }}`
  - Output format demanded:
    - `recommended_role` (must match available roles exactly)
    - `fit_score` (numeric)
    - `key_strengths`, `gaps`, `years_of_experience`, `top_skills`, `reasoning`
- **Input/Output:**
  - Input from Available Positions1.
  - Output → If3.
- **Failure/edge cases:**
  - Model may return invalid JSON; `jsonOutput: true` helps but isn’t foolproof.
  - `years_of_experience` could come back as string; downstream IF expects number.
  - If resume extraction returns empty text, scoring becomes unreliable.

**Sticky note applied (Block 2):**  
“## Step 2: Analyze the resume and evaluate candidate fit with AI …” (Sticky Note7)

---

### Block 3 — Qualification gate and Airtable persistence branching
**Overview:** Splits candidates by score. High-scoring candidates proceed to scheduling. Low-scoring candidates are currently routed to an Airtable create node (Create a record3).  
**Nodes involved:** If3, Create a record3

#### 2.9 If3
- **Type / role:** `if` — checks whether candidate meets threshold.
- **Configuration choices:**
  - Condition: `{{ $json.message.content.fit_score }} >= 8`
- **Input/Output:**
  - Input from Message a model3.
  - **True branch (index 0)** → Get Next Business Day1
  - **False branch (index 1)** → Create a record3
- **Failure/edge cases:**
  - If `fit_score` is missing or not numeric, strict validation may fail or evaluate unexpectedly.
  - Threshold is hard-coded (8).

#### 2.10 Create a record3
- **Type / role:** `airtable` — creates a candidate record in Airtable (currently on the **false** branch).
- **Configuration choices:**
  - Base/table are filled with concrete IDs in this node (`app8c...`, `tblYA...`).
  - Fields map from the current JSON (coming from Message a model3 on false branch):
    - `Gaps = {{ $json.message.content.gaps }}`
    - `Score = {{ $json.message.content.fit_score }}`
    - etc.
  - `Name` / `Email` come from `Get a message1` sender data.
- **Input/Output:**
  - Input from If3 false branch.
  - No outputs connected.
- **Failure/edge cases:**
  - Airtable auth/base/table mismatch.
  - Field type mismatch (Score/Experience expect numbers).
  - **Design inconsistency:** High-scoring candidates are recorded later via Create a record2, but low-scoring via Create a record3; these may duplicate or diverge.

---

### Block 4 — Check availability on the next business day (AI Agent + Calendar tools)
**Overview:** Computes the next business day and uses an AI Agent (with calendar tools) to pick the earliest free 1-hour slot between 09:00–18:00.  
**Nodes involved:** Get Next Business Day1, AI Agent1, OpenAI Chat Model2, Get Events1, Check Availability1, Structured Output Parser1, OpenAI Chat Model3

#### 2.11 Get Next Business Day1
- **Type / role:** `code` — computes next business day and time window strings.
- **Configuration choices:**
  - Adds 1 day, skips weekend (Sat/Sun → Monday).
  - Intended outputs:
    - `nextBusinessDay` (YYYY-MM-DD)
    - `windowStart` / `windowEnd` (YYYY-MM-DD HH:MM:SS)
    - `defaultSlotStart` / `defaultSlotEnd`
- **Critical issue (as written):**
  - The code constructs strings without quotes, e.g.  
    `const windowStart = ${yyyy}-${mm}-${dd} 09:00:00;`  
    This is **invalid JavaScript** (template interpolation only works in backticks, and the value must be a quoted string).
  - As-is, this node will fail at runtime unless n8n has auto-corrected it elsewhere.
- **Input/Output:**
  - Input from If3 true branch.
  - Output → AI Agent1 (main input).
- **Failure/edge cases:**
  - Timezone: uses server/runtime timezone for date calculation, but Calendar nodes use `Asia/Kolkata` (mismatch risk).
  - Business-day logic ignores holidays.

#### 2.12 AI Agent1
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool calls and returns a structured slot.
- **Configuration choices:**
  - Prompt instructs agent to:
    1) call “Get Events”
    2) call “Check Availability”
    3) output only the earliest 1-hour slot
  - Requires strict output schema:
    ```json
    { "start_time": "", "end_time": "" }
    ```
  - `hasOutputParser: true` and connected to Structured Output Parser1.
- **Input/Output:**
  - Main input from Get Next Business Day1.
  - Tools:
    - Get Events1 → connected as `ai_tool`
    - Check Availability1 → connected as `ai_tool`
  - Language model:
    - OpenAI Chat Model2 connected as `ai_languageModel`
  - Output parser:
    - Structured Output Parser1 connected as `ai_outputParser`
  - Output → Create an event1.
- **Failure/edge cases:**
  - If no slots available, prompt says to return a message, but parser expects JSON; could fail unless autoFix resolves.
  - If tool outputs are large (many events), LLM context limits may degrade performance.

#### 2.13 OpenAI Chat Model2
- **Type / role:** `lmChatOpenAi` — chat model provider for AI Agent1.
- **Configuration choices:** model `gpt-4.1-mini`.
- **Input/Output:** feeds AI Agent1 as the agent’s LM.
- **Failure/edge cases:** OpenAI auth, rate limits.

#### 2.14 Get Events1
- **Type / role:** `googleCalendarTool` — tool for the AI agent to list events.
- **Configuration choices:**
  - Operation: `getAll`, `returnAll: true`
  - `timeMin = {{ $json.windowStart }}`
  - `timeMax = {{ $json.windowEnd }}`
  - Calendar: `user@example.com`
- **Input/Output:**
  - Exposed to AI Agent1 via `ai_tool`.
- **Failure/edge cases:**
  - If `windowStart/windowEnd` are not valid RFC3339 or acceptable format, Calendar API may reject.
  - Calendar permissions.

#### 2.15 Check Availability1
- **Type / role:** `googleCalendarTool` — tool to compute free/busy availability.
- **Configuration choices:**
  - Resource: `calendar`
  - Timezone: `Asia/Kolkata`
  - `timeMin/timeMax` from Get Next Business Day1 (`windowStart/windowEnd`)
  - Calendar: `user@example.com`
- **Input/Output:**
  - Exposed to AI Agent1 via `ai_tool`.
- **Failure/edge cases:**
  - Same datetime format concerns as above.
  - Timezone mismatch with code node date generation.

#### 2.16 Structured Output Parser1
- **Type / role:** `outputParserStructured` — forces/repairs agent output to match schema.
- **Configuration choices:**
  - `autoFix: true`
  - Schema example: `{ "start_time": "", "end_time": "" }`
- **Input/Output:**
  - Connected to AI Agent1 as `ai_outputParser`.
  - Takes its language model from OpenAI Chat Model3 (see next).
- **Failure/edge cases:**
  - If agent returns non-JSON text (“No available slots…”), parser may fail or “fix” into empty fields.

#### 2.17 OpenAI Chat Model3
- **Type / role:** `lmChatOpenAi` — model used by the structured output parser (to auto-fix).
- **Configuration choices:** model `gpt-4.1-mini`.
- **Input/Output:** feeds Structured Output Parser1 as its LM.
- **Failure/edge cases:** same OpenAI concerns; also increases token usage.

**Sticky note applied (Block 4):**  
“## Step 3: Check availability on the next business day …” (Sticky Note8)

---

### Block 5 — Schedule the interview, notify the candidate, store results
**Overview:** Creates a Google Calendar event for the selected slot, emails the candidate details, then stores the evaluation to Airtable.  
**Nodes involved:** Create an event1, Send a message1, Create a record2

#### 2.18 Create an event1
- **Type / role:** `googleCalendar` — creates an event in Google Calendar.
- **Configuration choices:**
  - `start = {{ $json.output.start_time }}`
  - `end = {{ $json.output.end_time }}`
  - Summary: “Interview Scheduled with Dev Doshi”
  - Location: “iTechNotion Pvt Ltd Office, Makarba”
  - Attendees:
    - `{{ $('Get a message1').item.json.to.value[0].address }}`
    - `{{ $('Get a message1').item.json.from.value[0].address }}`
  - Description: `Interview for Role - {{ $('If3').item.json.message.content.recommended_role }}`
  - Default reminders disabled.
- **Input/Output:**
  - Input from AI Agent1 output.
  - Output → Send a message1.
- **Failure/edge cases:**
  - **Attendee bug risk:** `to.value[0].address` is typically the company mailbox, not the interviewer. If the inbound email was sent to a group alias, this may be wrong.
  - If AI Agent output is missing `output.start_time/end_time` (schema mismatch), event creation fails.
  - Calendar permissions/timezone issues.

#### 2.19 Send a message1
- **Type / role:** `gmail` — sends interview confirmation email to the candidate.
- **Configuration choices:**
  - `sendTo = {{ $('Gmail Trigger1').item.json.from.value[0].address }}`
  - Subject: `Interview Scheduled for - {{ $json.start.dateTime.split('T')[0] }}`
  - HTML body includes:
    - Candidate name from Get a message1
    - Role from If3
    - Date from calendar event start
    - Time formatting using inline JS to compute `h(e.getHours())` etc.
    - Location from created event output (`$json.location`)
  - `appendAttribution: false`
- **Input/Output:**
  - Input from Create an event1 (calendar event response).
  - Output → Create a record2.
- **Failure/edge cases:**
  - If Calendar node returns `start.dateTime` in a different structure (all-day event uses `start.date`), expression fails.
  - The inline time formatting expression is complex; can throw if `dateTime` missing.
  - Gmail send permission/scopes.

#### 2.20 Create a record2
- **Type / role:** `airtable` — creates a record for the candidate evaluation (connected after sending email).
- **Configuration choices:**
  - Base/table are placeholders (“enter your value here”) in this node.
  - Maps fields primarily from `Message a model3`:
    - `Score = {{ $('Message a model3').item.json.message.content.fit_score }}`
    - `Recommended Role = ...recommended_role`
    - plus strengths, gaps, skills, experience, reasoning
  - Name/Email from Get a message1 sender.
- **Input/Output:**
  - Input from Send a message1.
  - No outputs connected.
- **Failure/edge cases:**
  - Base/table not configured → will fail until user selects valid resources.
  - Type mismatch for number fields if model returns strings.

**Sticky note applied (Block 5):**  
“## Step 4: Schedule the interview and notify the candidate …” (Sticky Note9)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger1 | gmailTrigger | Poll Gmail for new emails + download attachments | — | Message a model2 | ## AI-Powered Resume Screening and Interview Scheduling Agent… (Sticky Note5) ; ## Step 1: Detect and collect job-application data… (Sticky Note6) |
| Message a model2 | @n8n/n8n-nodes-langchain.openAi | Classify email as job application (YES/NO) | Gmail Trigger1 | If2 | Sticky Note5 ; Sticky Note6 |
| If2 | if | Gate: proceed if classifier output is not “NO” | Message a model2 | Get a message1 | Sticky Note5 ; Sticky Note6 |
| Get a message1 | gmail | Fetch full email by ID + download attachments | If2 | Upload file1; Extract from File1 | Sticky Note5 ; Sticky Note6 |
| Upload file1 | googleDrive | Upload first attachment (resume) to Drive | Get a message1 | — | Sticky Note5 ; Sticky Note6 |
| Extract from File1 | extractFromFile | Extract text from PDF attachment | Get a message1 | Available Positions1 | Sticky Note5 ; Sticky Note6 |
| Available Positions1 | set | Provide list of open roles | Extract from File1 | Message a model3 | Sticky Note5 ; ## Step 2: Analyze the resume… (Sticky Note7) |
| Message a model3 | @n8n/n8n-nodes-langchain.openAi | Analyze resume vs roles, return JSON | Available Positions1 | If3 | Sticky Note5 ; Sticky Note7 |
| If3 | if | Gate: fit_score ≥ 8 | Message a model3 | (true) Get Next Business Day1; (false) Create a record3 | Sticky Note5 |
| Create a record3 | airtable | Persist candidate evaluation (currently for score < 8) | If3 (false) | — | Sticky Note5 |
| Get Next Business Day1 | code | Compute next business day & 9–18 window | If3 (true) | AI Agent1 | Sticky Note5 ; ## Step 3: Check availability… (Sticky Note8) |
| AI Agent1 | @n8n/n8n-nodes-langchain.agent | Select earliest available 1-hour slot using Calendar tools | Get Next Business Day1 | Create an event1 | Sticky Note5 ; Sticky Note8 |
| OpenAI Chat Model2 | lmChatOpenAi | LLM backend for AI Agent | — (wired as ai_languageModel) | AI Agent1 | Sticky Note5 ; Sticky Note8 |
| Get Events1 | googleCalendarTool | Tool: fetch events within window | — (tool connection) | AI Agent1 | Sticky Note5 ; Sticky Note8 |
| Check Availability1 | googleCalendarTool | Tool: compute availability within window | — (tool connection) | AI Agent1 | Sticky Note5 ; Sticky Note8 |
| Structured Output Parser1 | outputParserStructured | Enforce `{start_time,end_time}` output | — (wired as ai_outputParser) | AI Agent1 | Sticky Note5 ; Sticky Note8 |
| OpenAI Chat Model3 | lmChatOpenAi | LLM backend for structured output auto-fix | — (wired as ai_languageModel) | Structured Output Parser1 | Sticky Note5 ; Sticky Note8 |
| Create an event1 | googleCalendar | Create interview calendar event | AI Agent1 | Send a message1 | Sticky Note5 ; ## Step 4: Schedule the interview… (Sticky Note9) |
| Send a message1 | gmail | Email candidate with interview details | Create an event1 | Create a record2 | Sticky Note5 ; Sticky Note9 |
| Create a record2 | airtable | Persist candidate evaluation (currently after email) | Send a message1 | — | Sticky Note5 ; Sticky Note9 |
| Sticky Note5 | stickyNote | Documentation (canvas note) | — | — | (self) |
| Sticky Note6 | stickyNote | Documentation (canvas note) | — | — | (self) |
| Sticky Note7 | stickyNote | Documentation (canvas note) | — | — | (self) |
| Sticky Note8 | stickyNote | Documentation (canvas note) | — | — | (self) |
| Sticky Note9 | stickyNote | Documentation (canvas note) | — | — | (self) |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

1) **Create workflow**
   - Name it **Automated Hiring Assistant**.
   - Keep execution order setting default (`v1`).

2) **Add Gmail Trigger**
   - Node: **Gmail Trigger**
   - Polling: every minute
   - Enable **Download Attachments**
   - Set attachments prefix to **data**
   - Connect Gmail OAuth2 credentials with inbox read access.

3) **Add OpenAI email classifier**
   - Node: **OpenAI (LangChain) → Message a model**
   - Model: `gpt-4.1-mini`
   - Prompt: classifier prompt that outputs ONLY `YES`/`NO`
   - Use expressions:
     - `EMAIL SUBJECT: {{ $json.subject }}`
     - `EMAIL BODY: {{ $json.textAsHtml }}`
   - Connect: Gmail Trigger → Message a model

4) **Add IF gate for job applications**
   - Node: **IF**
   - Condition (string): `{{ $json.message.content }}` **notEquals** `NO`
   - Connect: Message a model → IF

   *Recommended improvement when rebuilding:* check **equals** `YES` with trimming/lowercasing (e.g., `{{ $json.message.content.trim().toUpperCase() }}`).

5) **Fetch full Gmail message**
   - Node: **Gmail**
   - Operation: **Get**
   - Message ID: `{{ $('Gmail Trigger1').item.json.id }}`
   - Enable **Download Attachments** with prefix `data`
   - Connect: IF (true) → Gmail(Get)

6) **Upload attachment to Google Drive (optional storage)**
   - Node: **Google Drive**
   - Operation: Upload
   - Binary field: `data0`
   - File name: `Resume - {{ $json.from.value[0].address }}`
   - Choose Drive (e.g., My Drive) and target Folder (e.g., Attachments)
   - Connect: Gmail(Get) → Google Drive(Upload)

7) **Extract resume text**
   - Node: **Extract From File**
   - Operation: **PDF**
   - Binary property: `data0`
   - Connect: Gmail(Get) → Extract From File

8) **Set available roles**
   - Node: **Set**
   - Add 3 string fields:
     - `Position 1` = `Automation Engineer`
     - `position 2` = `Full Stack Developer`
     - `Position 3` = `Javascript Developer`
   - Connect: Extract From File → Set

9) **Resume analysis with OpenAI (JSON output)**
   - Node: **OpenAI (LangChain) → Message a model**
   - Model: `gpt-4.1-mini`
   - Enable **JSON Output**
   - Prompt includes:
     - `{{ $('Extract from File1').item.json.text }}` for resume text
     - positions from Set node (`$json['Position 1']`, `$json['position 2']`, `$json['Position 3']`)
   - Connect: Set → Resume analysis node

10) **Add score threshold IF**
   - Node: **IF**
   - Condition (number): `{{ $json.message.content.fit_score }}` **gte** `8`
   - Connect: Resume analysis → IF

11) **Compute next business day (fix code)**
   - Node: **Code**
   - Ensure the timestamps are valid **strings**. Use backticks:
     - `const windowStart = \`${yyyy}-${mm}-${dd} 09:00:00\`;` etc.
   - Output fields: `nextBusinessDay`, `windowStart`, `windowEnd`, `defaultSlotStart`, `defaultSlotEnd`
   - Connect: IF (true) → Code

12) **Create AI Agent for slot selection**
   - Node: **AI Agent (LangChain)**
   - Prompt: availability-checking instructions (as in workflow) and enforce output schema.
   - Connect: Code → AI Agent

13) **Add AI Agent language model**
   - Node: **OpenAI Chat Model**
   - Model: `gpt-4.1-mini`
   - Connect this node to AI Agent via **ai_languageModel**.

14) **Add Google Calendar Tools**
   - Node A: **Google Calendar Tool** = “Get Events”
     - Operation: getAll, return all
     - timeMin: `{{ $json.windowStart }}`
     - timeMax: `{{ $json.windowEnd }}`
     - Calendar: your calendar (e.g., `user@example.com`)
   - Node B: **Google Calendar Tool** = “Check Availability”
     - Resource: calendar (free/busy)
     - timeMin/timeMax from the code node
     - Timezone: `Asia/Kolkata` (or your choice, but keep consistent)
   - Connect both tool nodes to the AI Agent via **ai_tool** connections.
   - Configure Google Calendar OAuth credentials with read + write scopes (for later event creation).

15) **Add Structured Output Parser**
   - Node: **Structured Output Parser**
   - Schema example:
     ```json
     { "start_time": "", "end_time": "" }
     ```
   - Enable **autoFix**
   - Add another **OpenAI Chat Model** node (or reuse one) for the parser’s `ai_languageModel`.
   - Connect Parser → AI Agent via **ai_outputParser**.

16) **Create calendar event**
   - Node: **Google Calendar**
   - Operation: Create event
   - Start: `{{ $json.output.start_time }}`
   - End: `{{ $json.output.end_time }}`
   - Summary/location/description as desired
   - Attendees:
     - Candidate email: `{{ $('Get a message1').item.json.from.value[0].address }}`
     - Interviewer email: set a fixed address or a field (avoid using `to.value[0]` unless intentional)
   - Connect: AI Agent → Create event

17) **Send confirmation email**
   - Node: **Gmail**
   - Operation: Send
   - SendTo: `{{ $('Gmail Trigger1').item.json.from.value[0].address }}`
   - Subject and HTML body referencing the created event output (`$json.start.dateTime`, `$json.end.dateTime`, `$json.location`)
   - Connect: Create event → Gmail(Send)
   - Ensure Gmail OAuth2 credentials include send scope.

18) **Persist candidate evaluation to Airtable**
   - Node: **Airtable**
   - Operation: Create record
   - Select Base and Table
   - Map fields from the resume analysis output:
     - Name/Email from `Get a message1`
     - Score/Role/Strengths/Gaps/Skills/Experience/Reasoning from `Message a model3`
   - Connect: Gmail(Send) → Airtable(Create)

19) **(Optional) Store low-scoring candidates too**
   - If you want both high and low scoring stored consistently:
     - Either connect IF (false) to the same Airtable node,
     - Or keep a second Airtable node but ensure both are configured identically.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Powered Resume Screening and Interview Scheduling Agent… Setup steps…” | Sticky Note5 (high-level workflow description + setup checklist) |
| “Step 1: Detect and collect job-application data…” | Sticky Note6 |
| “Step 2: Analyze the resume and evaluate candidate fit with AI…” | Sticky Note7 |
| “Step 3: Check availability on the next business day…” | Sticky Note8 |
| “Step 4: Schedule the interview and notify the candidate…” | Sticky Note9 |
| **Key implementation concern:** the Code node for next business day contains invalid JS string construction and must be corrected to return valid strings/datetimes. | Affects Get Next Business Day1 and downstream Calendar tools |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.