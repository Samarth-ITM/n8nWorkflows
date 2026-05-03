Schedule interviews and send GPT-4 reminders with Google Calendar, Gmail, Slack and Recrutei

https://n8nworkflows.xyz/workflows/schedule-interviews-and-send-gpt-4-reminders-with-google-calendar--gmail--slack-and-recrutei-12043


# Schedule interviews and send GPT-4 reminders with Google Calendar, Gmail, Slack and Recrutei

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Interview Scheduler with AI Reminders & Slack Approval*  
**Title (given):** *Schedule interviews and send GPT-4 reminders with Google Calendar, Gmail, Slack and Recrutei*

This workflow automates an interview lifecycle triggered by Recrutei ATS data: it schedules a Google Calendar event with a Google Meet link, drafts a reminder email using OpenAI, sends that reminder exactly 24 hours before the interview via Gmail, logs observations back into Recrutei, then after the interview time asks for Slack approval to decide whether the candidate proceeds or is rejected (and executes the corresponding ATS/email actions).

### 1.1 Input Reception (Recrutei → n8n)
Receives interview payload via Webhook and prepares standardized date/time fields.

### 1.2 Interview Scheduling (Google Calendar)
Creates the Calendar event, invites attendee(s), and generates a Google Meet conference URL.

### 1.3 AI Email Generation (OpenAI → parsing)
Uses meeting + candidate context to draft reminder email body and extracts the returned text into a simple field.

### 1.4 Timed Delivery (Wait → Gmail)
Waits until 24 hours before the interview, sends the reminder email, and logs an observation in Recrutei.

### 1.5 Post-Interview Decision (Wait → Slack approval → If)
Waits until the interview time, requests approval in Slack, then routes to “approved” vs “rejected”.

### 1.6 Approved Path (Recrutei observations → vacancy fetch → move pipeline stage)
Logs approval, fetches vacancy to determine last pipeline stage ID, then moves the candidate to the last stage.

### 1.7 Rejected Path (Gmail + Recrutei rating)
Sends a rejection email and marks/rates the application as not approved in Recrutei.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Date/Time Formatting
**Overview:** Accepts the Recrutei interview payload and normalizes the interview start datetime into a single ISO-like string used downstream.  
**Nodes involved:** `Webhook`, `Formatting the start date/time`

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point; receives HTTP request from Recrutei.
- **Config choices:**
  - Path: `interview-calendar-meet`
  - Uses standard Webhook trigger behavior (expects JSON payload).
- **Key fields expected (from pinned data):**
  - `when.marked_at`, `when.start_at`, `candidate.email`, `participants[0].email`, `vacancy`, `application_id`, `id` (talent id).
- **Connections:**
  - Output → `Formatting the start date/time`
- **Failure/edge cases:**
  - Payload mismatch (missing `when.*`, `candidate`, `participants`) will break expressions later.
  - If Recrutei sends times in a timezone that differs from the hardcoded offset, scheduling may be wrong.

#### Node: Formatting the start date/time
- **Type / role:** `n8n-nodes-base.set` — enriches incoming JSON with a `when` datetime string.
- **Config choices (interpreted):**
  - Adds/overwrites field `when` as a string composed from:
    - date: `$json.when.marked_at`
    - time: `$json.when.start_at`
    - fixed seconds/millis and timezone offset: `:00.000-05:00`
  - Keeps all other fields (`includeOtherFields: true`).
- **Key expression:**
  - `when = {{ $json.when.marked_at }}T{{ $json.when.start_at }}:00.000-05:00`
- **Connections:**
  - Input ← `Webhook`
  - Output → `Create the interview and invites the candidate`
- **Failure/edge cases:**
  - If `when.start_at` is not `HH:mm`, resulting ISO string may be invalid.
  - Hardcoded `-05:00` may conflict with later nodes using other timezones.

---

### Block 2 — Create Calendar Event + Meet Link
**Overview:** Creates a Google Calendar event and attaches a Google Meet link, inviting the candidate and the first participant.  
**Nodes involved:** `Create the interview and invites the candidate`

#### Node: Create the interview and invites the candidate
- **Type / role:** `n8n-nodes-base.googleCalendar` — creates an event.
- **Config choices (interpreted):**
  - **Start** and **End** both set to the same `when` value (creates a zero-duration event unless Google applies defaults).
  - Summary: `Interview for {{ $json.vacancy }}`
  - Attendees: candidate email + first participant email.
  - Conference: requests Google Meet (`hangoutsMeet`).
  - Calendar selection: set to “list” mode but `value` is empty → requires user to choose a calendar in UI to function reliably.
- **Key expressions:**
  - `start = {{ $json.when }}`
  - `end = {{ $json.when }}`
  - `attendees = {{ $json.candidate.email }}, {{ $json.participants[0].email }}`
- **Connections:**
  - Input ← `Formatting the start date/time`
  - Output → `Creates the e-mail content`
- **Credentials:**
  - Google Calendar OAuth2 (`Google Calendar Account`)
- **Failure/edge cases:**
  - Missing/empty calendar ID will fail.
  - Conference creation can fail if the Google Workspace account disallows Meet creation or lacks permissions.
  - Same start/end may create an instant event; usually you want an interview duration (e.g., +30/60 minutes).

---

### Block 3 — AI Reminder Drafting & Output Extraction
**Overview:** Builds a prompt with meeting details and candidate name, asks OpenAI to draft the reminder email body, then extracts plain text for Gmail.  
**Nodes involved:** `Creates the e-mail content`, `Separates the text`

#### Node: Creates the e-mail content
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat generation (LangChain-based node).
- **Config choices (interpreted):**
  - Model: `gpt-4.o-mini`
  - Prompt content includes:
    - interview start time (`$json.start.dateTime` from Google Calendar event output)
    - event summary
    - Meet URL (`$json.conferenceData.entryPoints[0].uri`)
    - candidate name from the earlier node via node reference.
  - System message instructs: EN-US, professional, body only, no subject, no signing.
- **Key expressions / references:**
  - Candidate name: `{{ $('Formatting the start date/time').item.json.candidate.name }}`
  - Calendar fields: `{{ $json.start.dateTime }}`, `{{ $json.summary }}`, `{{ $json.conferenceData.entryPoints[0].uri }}`
- **Connections:**
  - Input ← `Create the interview and invites the candidate`
  - Output → `Separates the text`
- **Credentials:**
  - OpenAI API (`OpenAI Account`)
- **Failure/edge cases:**
  - If `conferenceData.entryPoints[0].uri` is absent (Meet not created), prompt will break or produce incomplete email.
  - OpenAI rate limits / quota / auth errors.
  - Output structure differences across node versions/models can break downstream extraction.

#### Node: Separates the text
- **Type / role:** `n8n-nodes-base.set` — extracts the generated email body into `text`.
- **Config choices (interpreted):**
  - Creates `text = $json.output[0].content[0].text`
- **Key expression:**
  - `text = {{ $json.output[0].content[0].text }}`
- **Connections:**
  - Input ← `Creates the e-mail content`
  - Output → `Wait until 1 day before the interview` **and** also directly to `Send the message to the candidate` (parallel wiring; see edge case below)
- **Failure/edge cases:**
  - If OpenAI node returns a different structure, `output[0].content[0].text` may be undefined.
  - The workflow connections include a direct path to Gmail *in addition* to the Wait path; if both execute, you may send immediately and again later (see Block 4).

---

### Block 4 — Timed Reminder Delivery + ATS Logging
**Overview:** Waits until 24 hours before the interview, sends the AI-generated reminder via Gmail, then records an observation in Recrutei.  
**Nodes involved:** `Wait until 1 day before the interview`, `Send the message to the candidate`, `Adding observation in candidate`

#### Node: Wait until 1 day before the interview
- **Type / role:** `n8n-nodes-base.wait` — pauses execution until a calculated datetime.
- **Config choices (interpreted):**
  - Resume at a specific time computed from the Google Calendar event start time minus 1 day.
  - Note: “You can personalize as you need”.
- **Key expression (Luxon DateTime):**
  - `{{ DateTime.fromISO($('Create the interview and invites the candidate').item.json.start.dateTime).minus({ days: 1 }).toISO() }}`
- **Connections:**
  - Input ← `Separates the text`
  - Output → `Send the message to the candidate`
- **Failure/edge cases:**
  - If the calendar start time is missing/invalid, Luxon conversion fails.
  - If the computed time is in the past, n8n may resume immediately.

#### Node: Send the message to the candidate
- **Type / role:** `n8n-nodes-base.gmail` — sends the reminder email.
- **Config choices (interpreted):**
  - To: candidate email from the original webhook payload.
  - Subject: the calendar event summary.
  - Body: extracted AI text.
  - Sender name: participant[0].name (recruiter).
  - Attribution disabled.
  - Note: “The source node of the data is specified to prevent errors.”
- **Key expressions (explicit node sourcing):**
  - `sendTo = {{ $('Formatting the start date/time').item.json.candidate.email }}`
  - `subject = {{ $('Create the interview and invites the candidate').item.json.summary }}`
  - `message = {{ $('Separates the text').item.json.text }}`
  - `senderName = {{ $('Formatting the start date/time').item.json.participants[0].name }}`
- **Connections:**
  - Input ← `Wait until 1 day before the interview`
  - Output → `Adding observation in candidate`
  - Additionally, due to connections, it can also be triggered directly from `Separates the text` (see “edge case” below).
- **Credentials:**
  - Gmail OAuth2 (`Gmail Account`)
- **Failure/edge cases:**
  - Gmail OAuth permission issues (send scope), quota, or “From” identity restrictions.
  - If the direct connection from `Separates the text` remains, the email may send immediately (unintended).

#### Node: Adding observation in candidate
- **Type / role:** `n8n-nodes-base.httpRequest` — logs an observation in Recrutei ATS that the interview is scheduled and provides the Meet URL.
- **Config choices (interpreted):**
  - POST `https://api.recrutei.com.br/api/v2/talents-observations`
  - Multipart form-data body with:
    - `application_id` from Webhook
    - `talent_id` from Webhook (`id`)
    - `talent_observation_type_id = 7`
    - description includes Meet URL from Calendar node
    - `action = 1`
  - Headers include bearer token placeholder and multipart content type.
- **Key expressions:**
  - `application_id = {{ $('Webhook').item.json.application_id }}`
  - `talent_id = {{ $('Webhook').item.json.id }}`
  - `description = Candidate interview already set on Google Meet:\n{{ $('Create the interview and invites the candidate').item.json.conferenceData.entryPoints[0].uri }}`
- **Connections:**
  - Input ← `Send the message to the candidate`
  - Output → `Wait` (post-interview wait)
- **Failure/edge cases:**
  - Token missing/expired → 401/403.
  - `Content-Type: multipart/form-data` is manually set; depending on n8n behavior, boundary handling can be problematic. Prefer letting n8n set multipart boundaries automatically when using form-data options.

**Important edge case (wiring):**  
`Separates the text` connects to both **Wait until 1 day before** and **Send the message to the candidate**. If left as-is, Gmail may execute immediately (via the direct branch) and then again after the Wait node. If the intent is “send only 24h before”, remove the direct connection to Gmail and keep only the Wait path.

---

### Block 5 — Post-Interview Wait + Slack Approval + Routing
**Overview:** After logging the scheduled interview, the workflow waits until the interview time, requests an approval decision in Slack, then routes based on the approval payload.  
**Nodes involved:** `Wait`, `Request for candidate aproval`, `If`

#### Node: Wait
- **Type / role:** `n8n-nodes-base.wait` — pauses until the interview datetime (as built in Block 1).
- **Config choices:**
  - Resume at specific time: `{{ $('Formatting the start date/time').item.json.when }}`
- **Connections:**
  - Input ← `Adding observation in candidate`
  - Output → `Request for candidate aproval`
- **Failure/edge cases:**
  - If `when` is not valid ISO, Wait scheduling fails.
  - If interview time is in the past, it resumes immediately.

#### Node: Request for candidate aproval
- **Type / role:** `n8n-nodes-base.slack` — sends a Slack approval message and waits for response.
- **Config choices (interpreted):**
  - Operation: `sendAndWait`
  - Approval type: `double` (requires double confirmation/interaction per Slack node’s approval mode).
  - Message: “The candidate X was approved and can proceed to the next fase?”
  - Channel: configured in UI, but `value` is empty in the JSON → must be set.
  - Limit wait time: resume amount set to 4 (node-specific option controlling how long it waits/polls/handles responses).
- **Key expressions:**
  - Candidate name: `{{ $('Webhook').item.json.candidate.name }}`
- **Connections:**
  - Input ← `Wait`
  - Output → `If`
- **Credentials:**
  - Slack API (`Slack Account`)
- **Failure/edge cases:**
  - Missing channel configuration prevents sending.
  - Slack app permissions missing (chat:write, interactive components, etc.).
  - If approvals are not completed, downstream `If` may never run.

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — branches on Slack approval result.
- **Config choices (interpreted):**
  - Condition checks `{{ $json.data.approved }}` is boolean true.
  - `executeOnce: true` (ensures it only evaluates once even if multiple items).
- **Connections:**
  - **True** → `Adding observation in candidate1` (approved path)
  - **False** → `Reproval e-mail` (rejected path)
- **Failure/edge cases:**
  - If Slack node output does not include `data.approved`, the condition fails (routes to false branch).

---

### Block 6 — Approved Path: Log + Fetch Vacancy + Move Pipeline Stage
**Overview:** When approved, the workflow logs an approval observation, fetches vacancy data to locate the last pipeline stage ID, then moves the application to that stage in Recrutei.  
**Nodes involved:** `Adding observation in candidate1`, `Getting the vacancy`, `Selecting pipe stage id`, `Moving candidate to last stage`

#### Node: Adding observation in candidate1
- **Type / role:** `n8n-nodes-base.httpRequest` — adds approval observation in Recrutei.
- **Config choices:**
  - POST `/api/v2/talents-observations`
  - Multipart body: application_id, talent_id, observation_type_id=7, description “approved…”, action=1
  - Bearer token placeholder.
- **Connections:**
  - Input ← `If` (true branch)
  - Output → `Getting the vacancy`
- **Failure/edge cases:**
  - Same multipart/header caveats as earlier HTTP node.
  - Token/auth errors.

#### Node: Getting the vacancy
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves vacancy details from Recrutei.
- **Config choices:**
  - GET `https://api.recrutei.com.br/api/v1/vacancies/{{ vacancy.id }}`
  - Vacancy ID is referenced from **Adding observation in candidate** output:  
    `{{ $('Adding observation in candidate').item.json.data.vacancy.id }}`
- **Connections:**
  - Input ← `Adding observation in candidate1`
  - Output → `Selecting pipe stage id`
- **Failure/edge cases:**
  - Potential logical fragility: vacancy id is taken from a different node (`Adding observation in candidate`) rather than from the current approved-path context. If that node didn’t run (or data differs), this may be undefined.
  - Token/auth errors.

#### Node: Selecting pipe stage id
- **Type / role:** `n8n-nodes-base.set` — selects the “last stage” ID from vacancy pipeline stages.
- **Config choices:**
  - Sets numeric field `last_stage` to `pipe_stages[7].id`
  - Note: “You must personalize this as you need”
- **Key expression:**
  - `last_stage = {{ $json.pipe_vacancy.pipe.pipe_stages[7].id }}`
- **Connections:**
  - Input ← `Getting the vacancy`
  - Output → `Moving candidate to last stage`
- **Failure/edge cases:**
  - Hardcoded index `[7]` assumes at least 8 stages; if fewer, expression fails.
  - “Last stage” may not be index 7; should compute last element dynamically.

#### Node: Moving candidate to last stage
- **Type / role:** `n8n-nodes-base.httpRequest` — moves the application to a pipeline stage.
- **Config choices:**
  - POST `https://api.recrutei.com.br/api/v1/applications/{{ application_id }}/pipe`
  - Body: `pipe_stage_id = {{ $json.last_stage }}`
- **Connections:**
  - Input ← `Selecting pipe stage id`
  - Output: none (end)
- **Failure/edge cases:**
  - Token/auth errors.
  - Invalid stage ID or application state constraints.

---

### Block 7 — Rejected Path: Email + Recrutei Rating
**Overview:** When not approved, the workflow sends a rejection email to the candidate and updates the application rating/status in Recrutei.  
**Nodes involved:** `Reproval e-mail`, `Reproving candidate`

#### Node: Reproval e-mail
- **Type / role:** `n8n-nodes-base.gmail` — sends rejection email.
- **Config choices:**
  - To: candidate email from Webhook.
  - Subject: vacancy name from Webhook.
  - Body: fixed template in English.
  - `executeOnce: true`
- **Key expressions:**
  - `sendTo = {{ $('Webhook').item.json.candidate.email }}`
  - `subject = {{ $('Webhook').item.json.vacancy }}`
  - Message interpolates candidate name and vacancy.
- **Connections:**
  - Input ← `If` (false branch)
  - Output → `Reproving candidate`
- **Credentials:**
  - Gmail OAuth2 (`Gmail Account`)
- **Failure/edge cases:**
  - Same Gmail credential/sending constraints as the reminder email.

#### Node: Reproving candidate
- **Type / role:** `n8n-nodes-base.httpRequest` — rates/rejects application in Recrutei.
- **Config choices:**
  - PUT `https://api.recrutei.com.br/api/v1/applications/rate`
  - Body: `application_id` + `rated = 0`
- **Connections:**
  - Input ← `Reproval e-mail`
  - Output: none (end)
- **Failure/edge cases:**
  - Token/auth errors.
  - API semantics: `rated=0` must match Recrutei’s expected values for rejection.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point from Recrutei ATS | — | Formatting the start date/time | ## Overview: AI-Powered Interview Scheduling & Reminders… (steps 1–6 incl. Recrutei token) |
| Formatting the start date/time | Set | Build ISO datetime string for interview | Webhook | Create the interview and invites the candidate | ## Ingestion & Scheduling … formats the date/time into ISO strings |
| Create the interview and invites the candidate | Google Calendar | Create calendar event + Meet link + invites | Formatting the start date/time | Creates the e-mail content | ## Ingestion & Scheduling … creates event and video conference URL |
| Creates the e-mail content | OpenAI (LangChain) | Generate reminder email body | Create the interview and invites the candidate | Separates the text | ## AI-Driven Copywriting … humanized reminder email in English |
| Separates the text | Set | Extract AI text into a field for Gmail | Creates the e-mail content | Wait until 1 day before the interview; Send the message to the candidate | ## AI-Driven Copywriting … (same block context) |
| Wait until 1 day before the interview | Wait | Pause until 24h before interview | Separates the text | Send the message to the candidate | ## Smart Delivery … calculates optimal time (24 hours before) |
| Send the message to the candidate | Gmail | Send reminder email | Wait until 1 day before the interview *(also connected from Separates the text)* | Adding observation in candidate | ## Smart Delivery … triggers Gmail node to send AI-generated content |
| Adding observation in candidate | HTTP Request | Log scheduled interview + Meet link in Recrutei | Send the message to the candidate | Wait | ## Smart Delivery … “register in the observations on the ATS.” |
| Wait | Wait | Pause until interview time | Adding observation in candidate | Request for candidate aproval | ## Approving/Declining candidate … sends message in Slack after interview |
| Request for candidate aproval | Slack | Send approval request and wait for decision | Wait | If | ## Approving/Declining candidate … approval/decline in chosen channel |
| If | If | Route approved vs rejected | Request for candidate aproval | Adding observation in candidate1; Reproval e-mail | ## Approving/Declining candidate … decision routing |
| Adding observation in candidate1 | HTTP Request | Log approval observation in Recrutei | If (true) | Getting the vacancy | ## Moving the candidate to the last pipe stage … adds observation informing decision |
| Getting the vacancy | HTTP Request | Fetch vacancy pipeline info | Adding observation in candidate1 | Selecting pipe stage id | ## Moving the candidate to the last pipe stage … gets vacancy |
| Selecting pipe stage id | Set | Choose pipeline stage ID to move candidate | Getting the vacancy | Moving candidate to last stage | ## Moving the candidate to the last pipe stage … separate the pipe_stage_id of the last pipe stage |
| Moving candidate to last stage | HTTP Request | Move application to selected pipeline stage | Selecting pipe stage id | — | ## Moving the candidate to the last pipe stage … moves the candidate to the last stage |
| Reproval e-mail | Gmail | Send rejection email | If (false) | Reproving candidate | ## Reproving candidate … sends reproval e-mail then informs ATS |
| Reproving candidate | HTTP Request | Mark/rate application as rejected in Recrutei | Reproval e-mail | — | ## Reproving candidate … creates ATS-side decision record (via API) |
| Sticky Note | Sticky Note | Documentation | — | — |  |
| Sticky Note1 | Sticky Note | Documentation | — | — |  |
| Sticky Note2 | Sticky Note | Documentation | — | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — |  |
| Sticky Note4 | Sticky Note | Documentation | — | — |  |
| Sticky Note5 | Sticky Note | Documentation | — | — |  |
| Sticky Note6 | Sticky Note | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: *Interview Scheduler with AI Reminders & Slack Approval* (or your preferred name).

2) **Add `Webhook` (Trigger)**
   - Node type: **Webhook**
   - Path: `interview-calendar-meet`
   - Method: typically **POST**
   - Activate “Production URL” and configure Recrutei ATS to send interview JSON to it.

3) **Add `Set` → “Formatting the start date/time”**
   - Add field `when` (String) with:
     - `{{ $json.when.marked_at }}T{{ $json.when.start_at }}:00.000-05:00`
   - Enable **Include Other Fields**.

4) **Add `Google Calendar` → “Create the interview and invites the candidate”**
   - Operation: **Create Event**
   - **Start**: `{{ $json.when }}`
   - **End**: `{{ $json.when }}` (recommended to adjust to a duration, e.g. add 1 hour)
   - Additional fields:
     - Summary: `Interview for {{ $json.vacancy }}`
     - Attendees: `{{ $json.candidate.email }}, {{ $json.participants[0].email }}`
     - Conference / video meeting: select **Google Meet (hangoutsMeet)**
   - Credentials: connect **Google Calendar OAuth2**
   - Select an actual **Calendar** in the node UI (do not leave it blank).

5) **Add `OpenAI (LangChain)` → “Creates the e-mail content”**
   - Node type: **OpenAI** (LangChain node in n8n)
   - Model: `gpt-4.o-mini` (or equivalent available)
   - Provide messages:
     - System message instructing professional EN-US body-only reminder without signature.
     - User/content message that includes:
       - Start time: `{{ $json.start.dateTime }}`
       - Summary: `{{ $json.summary }}`
       - Meet URL: `{{ $json.conferenceData.entryPoints[0].uri }}`
       - Candidate name: `{{ $('Formatting the start date/time').item.json.candidate.name }}`
   - Credentials: **OpenAI API key** in n8n credentials.

6) **Add `Set` → “Separates the text”**
   - Add field `text` with:
     - `{{ $json.output[0].content[0].text }}`
   - (If your OpenAI node outputs a different structure, adjust this mapping accordingly.)

7) **Add `Wait` → “Wait until 1 day before the interview”**
   - Resume: **At a specific time**
   - Datetime expression:
     - `{{ DateTime.fromISO($('Create the interview and invites the candidate').item.json.start.dateTime).minus({ days: 1 }).toISO() }}`
   - Optionally adjust timing policy (e.g., minus 2 days, minus hours, etc.).

8) **Add `Gmail` → “Send the message to the candidate”**
   - Operation: **Send**
   - To: `{{ $('Formatting the start date/time').item.json.candidate.email }}`
   - Subject: `{{ $('Create the interview and invites the candidate').item.json.summary }}`
   - Message: `{{ $('Separates the text').item.json.text }}`
   - Sender name: `{{ $('Formatting the start date/time').item.json.participants[0].name }}`
   - Disable attribution if desired.
   - Credentials: **Gmail OAuth2**
   - **Connection order:** connect **Wait until 1 day before** → **Gmail**  
     - Recommended: do **not** connect “Separates the text” directly to Gmail if you only want timed sending.

9) **Add `HTTP Request` → “Adding observation in candidate” (Recrutei)**
   - Method: **POST**
   - URL: `https://api.recrutei.com.br/api/v2/talents-observations`
   - Send body: **Yes** (multipart/form-data style fields)
   - Body fields:
     - `application_id`: `{{ $('Webhook').item.json.application_id }}`
     - `talent_id`: `{{ $('Webhook').item.json.id }}`
     - `talent_observation_type_id`: `7`
     - `description`: `Candidate interview already set on Google Meet:\n{{ $('Create the interview and invites the candidate').item.json.conferenceData.entryPoints[0].uri }}`
     - `action`: `1`
   - Header:
     - `Authorization: Bearer <YOUR_RECRUTEI_TOKEN>`
   - (Prefer configuring credentials or environment variable for the token.)

10) **Add `Wait` → “Wait” (until interview time)**
   - Resume: **At a specific time**
   - Datetime: `{{ $('Formatting the start date/time').item.json.when }}`

11) **Add `Slack` → “Request for candidate aproval”**
   - Operation: **Send and Wait (approval)**
   - Channel: select your target channel in UI
   - Message: `The candidate {{ $('Webhook').item.json.candidate.name }} was approved and can proceed to the next fase?`
   - Approval type: **double**
   - Credentials: **Slack API** (ensure interactive approvals are enabled and permitted).

12) **Add `If`**
   - Condition: Boolean is true
   - Left value: `{{ $json.data.approved }}`
   - True branch = approved; False branch = rejected.

13) **Approved branch nodes**
   1. `HTTP Request` → “Adding observation in candidate1”
      - POST `https://api.recrutei.com.br/api/v2/talents-observations`
      - Body: application_id, talent_id, observation_type_id=7, description “approved…”, action=1
      - Auth header bearer token
   2. `HTTP Request` → “Getting the vacancy”
      - GET `https://api.recrutei.com.br/api/v1/vacancies/{{ <vacancy_id> }}`
      - In the provided workflow the vacancy id is referenced from:  
        `{{ $('Adding observation in candidate').item.json.data.vacancy.id }}`  
        (Recommended: source vacancy id directly from Webhook payload if available to reduce coupling.)
   3. `Set` → “Selecting pipe stage id”
      - Field `last_stage` (Number): `{{ $json.pipe_vacancy.pipe.pipe_stages[7].id }}`
      - Adjust to your pipeline structure.
   4. `HTTP Request` → “Moving candidate to last stage”
      - POST `https://api.recrutei.com.br/api/v1/applications/{{ $('Webhook').item.json.application_id }}/pipe`
      - Body: `pipe_stage_id = {{ $json.last_stage }}`
      - Auth header bearer token

14) **Rejected branch nodes**
   1. `Gmail` → “Reproval e-mail”
      - To: `{{ $('Webhook').item.json.candidate.email }}`
      - Subject: `{{ $('Webhook').item.json.vacancy }}`
      - Message: fixed rejection template (as in workflow)
      - Credentials: Gmail OAuth2
   2. `HTTP Request` → “Reproving candidate”
      - PUT `https://api.recrutei.com.br/api/v1/applications/rate`
      - Body:
        - `application_id = {{ $('Webhook').item.json.application_id }}`
        - `rated = 0`
      - Auth header bearer token

15) **Credentials checklist**
   - Google Calendar OAuth2: must allow creating events + conference data.
   - Gmail OAuth2: must allow sending email as the account.
   - OpenAI API: valid key, model access.
   - Slack API: permissions to post messages and handle interactive approvals.
   - Recrutei API token: store securely (credential/env var), apply as `Authorization: Bearer ...`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Copy the Production URL and configure it in your Recrutei ATS to send interview JSON data.” | Sticky note “Overview” setup step for Webhook |
| “Google Calendar node is pre-configured to generate a ‘hangoutsMeet’ conference link.” | Sticky note “Overview” |
| “OpenAI requires a valid API Key… draft a professional email body.” | Sticky note “Overview” |
| “Wait Node currently set to ‘1 day before’… adjust timing.” | Sticky note “Overview” + Wait node note |
| “Recrutei’s API: Inserts your Recrutei token in the Authorization header” | Sticky note “Overview” |
| “Selecting pipe stage id… You must personalize this as you need” | Note attached to “Selecting pipe stage id” node |
| “Smart Delivery… Then, it register in the observations on the ATS.” | Sticky note “Smart Delivery” (note: grammar preserved) |

