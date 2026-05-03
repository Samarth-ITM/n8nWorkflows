Send meeting summaries with Mailchimp and MongoDB

https://n8nworkflows.xyz/workflows/send-meeting-summaries-with-mailchimp-and-mongodb-12760


# Send meeting summaries with Mailchimp and MongoDB

## 1. Workflow Overview

**Workflow name:** Meeting Notes Distributor with Mailchimp and MongoDB  
**User-provided title:** Send meeting summaries with Mailchimp and MongoDB

**Purpose:**  
Once per day, the workflow fetches newly available meeting recordings, submits each recording to a transcription service, waits briefly, polls for the transcript, generates a short summary, formats an HTML email, stores everything in MongoDB, then syncs participants to Mailchimp and sends a one-off campaign containing the summary.

**Primary use cases:**
- Automatically distribute meeting summaries to participants
- Maintain an auditable, searchable history of transcripts/summaries in MongoDB
- Keep a Mailchimp list up to date with meeting participant emails

### 1.1 Trigger & Meeting Retrieval
Daily schedule → pull available recordings → stop if none → process meetings one-by-one.

### 1.2 Transcription Polling & Summary Generation
Start transcription job → wait → fetch transcript job result → continue only if completed → summarise transcript → format HTML email.

### 1.3 Database Storage (MongoDB)
Assemble a clean document → insert into `meeting_summaries`.

### 1.4 Mailchimp Sync & Campaign Send
Extract participant emails → upsert list members → create campaign → set content → send → check send status.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Meeting Retrieval

**Overview:**  
Runs every 24 hours, pulls a list of available meeting recordings from a meetings API, exits quietly if there are none, and then processes each meeting sequentially.

**Nodes involved:**
- **Daily Schedule** (Schedule Trigger)
- **Fetch Meetings** (HTTP Request)
- **Any Meetings?** (IF)
- **Split Meetings** (Split In Batches)

#### Node: Daily Schedule
- **Type / role:** `Schedule Trigger` — starts the workflow on a timer.
- **Config choices:** Runs every **24 hours** (`hoursInterval: 24`).
- **Inputs/outputs:** Entry point → outputs to **Fetch Meetings**.
- **Edge cases / failures:** None typical besides instance downtime; missed executions depend on n8n settings.
- **Version notes:** Type version 1.

#### Node: Fetch Meetings
- **Type / role:** `HTTP Request` — retrieves meeting recordings.
- **Config choices:**
  - **GET** `https://api.yourmeetings.com/v1/recordings?status=available`
  - Authentication: **Predefined Credential Type** = `httpHeaderAuth` (expects a header-based credential; sticky note calls it **“Meetings API”**).
- **Key data assumptions:** Response is a JSON array of meetings containing IDs, recording URLs, titles, participants, etc.
- **Inputs/outputs:** Input from **Daily Schedule** → output to **Any Meetings?**
- **Edge cases / failures:**
  - Auth header missing/invalid → 401/403
  - Non-array response (object/string) will break the next IF expression that counts items
  - Pagination not handled (if API pages results)
- **Version notes:** Type version 4.

#### Node: Any Meetings?
- **Type / role:** `IF` — guards against running downstream logic when no meetings exist.
- **Config choices:**
  - Condition (number): `{{ $items("Fetch Meetings").length }}` **larger than** `0`
- **Inputs/outputs:** Input from **Fetch Meetings** → true branch to **Split Meetings**. (False branch is not connected; workflow stops.)
- **Edge cases / failures:**
  - If **Fetch Meetings** returns 1 item whose JSON contains an array rather than being an array itself, `$items("Fetch Meetings").length` may be 1 even if internal list is empty.
  - If **Fetch Meetings** errors, the workflow fails before this node.
- **Version notes:** Type version 2.

#### Node: Split Meetings
- **Type / role:** `Split In Batches` — processes meetings sequentially.
- **Config choices:** Defaults (batch size not explicitly set in parameters shown).
- **Inputs/outputs:** Input from **Any Meetings?** → outputs to **Initiate Transcription**.
- **Edge cases / failures:**
  - As wired, there is **no connection back** to “next batch”; in typical patterns you loop back to keep pulling batches until done. With the current connections, it may only process the first batch depending on n8n’s default batch size/behavior.
- **Version notes:** Type version 3.

---

### Block 2 — Transcription Polling & Summary Generation

**Overview:**  
Submits each meeting recording to a transcription API, waits 2 minutes, polls for completion once, then produces a short summary and an HTML email body + subject.

**Nodes involved:**
- **Initiate Transcription** (HTTP Request)
- **Wait 2 min** (Wait)
- **Get Transcript** (HTTP Request)
- **Transcript Ready?** (IF)
- **Summarise Transcript** (Code)
- **Format Email** (Code)

#### Node: Initiate Transcription
- **Type / role:** `HTTP Request` — creates a transcription job.
- **Config choices:**
  - **POST** `https://api.transcriptionservice.com/v1/jobs`
  - Auth: `httpHeaderAuth` (sticky note calls it **“Transcription API”**)
  - Body is **not defined** in the node configuration shown. Most transcription APIs require sending either `audio_url`/`recordingUrl` or file upload.
- **Inputs/outputs:** From **Split Meetings** → to **Wait 2 min**.
- **Key data assumptions:** Output contains a job identifier in `$json.id`.
- **Edge cases / failures:**
  - Missing POST body → job creation fails (400) unless API accepts defaults
  - If job id field differs (`job_id`, `id`, etc.), downstream `Get Transcript` URL breaks
- **Version notes:** Type version 4.

#### Node: Wait 2 min
- **Type / role:** `Wait` — delays execution to allow transcription to complete.
- **Config choices:** No explicit duration parameters visible; node name implies **2 minutes**, but the parameters are empty in the JSON excerpt.
- **Inputs/outputs:** From **Initiate Transcription** → to **Get Transcript**.
- **Edge cases / failures:**
  - If the wait duration isn’t actually configured, it may not delay as intended.
  - Long recordings likely won’t finish in 2 minutes; since there is no loop/polling cycle, many jobs will fail the “completed” check and stop.
- **Version notes:** Type version 1.

#### Node: Get Transcript
- **Type / role:** `HTTP Request` — polls the transcription job.
- **Config choices:**
  - GET `https://api.transcriptionservice.com/v1/jobs/{{ $json.id }}`
  - Auth: `httpHeaderAuth`
- **Inputs/outputs:** From **Wait 2 min** → to **Transcript Ready?**
- **Edge cases / failures:**
  - `$json.id` missing or not the job id → 404/invalid URL
  - API may require different endpoint or query parameters
- **Version notes:** Type version 4.

#### Node: Transcript Ready?
- **Type / role:** `IF` — continues only if transcript is complete.
- **Config choices:** String condition: `{{ $json.status }}` equals `completed`.
- **Inputs/outputs:** From **Get Transcript** → true branch to **Summarise Transcript**. (False branch not connected; processing ends for that meeting.)
- **Edge cases / failures:**
  - Status values differ (`done`, `finished`) → false negatives
  - No retry/backoff; incomplete jobs are dropped
- **Version notes:** Type version 2.

#### Node: Summarise Transcript
- **Type / role:** `Code` — creates a simple executive summary.
- **Config choices (interpreted):**
  - Reads `transcript_text` from the current item: `const transcript = $json.transcript_text || ''`
  - Splits on `.` and takes first 5 “sentences”
  - Outputs same JSON plus `summary`
- **Inputs/outputs:** From **Transcript Ready?** → to **Format Email**
- **Edge cases / failures:**
  - If transcript has few/no periods (or uses newlines), summary quality is poor
  - If transcript is empty, output summary becomes `"."` (because it appends `'.'` even when empty)
  - Potential mismatch: later nodes reference `$json.transcript` (not `transcript_text`)
- **Version notes:** Type version 2.

#### Node: Format Email
- **Type / role:** `Code` — builds HTML content and email subject.
- **Config choices (interpreted):**
  - `meetingTitle = $json.meetingTitle || 'Meeting Summary'`
  - Generates basic HTML with `<h2>` and `<p>` containing the summary
  - Adds `emailSubject: Summary – ${meetingTitle}` and `htmlContent`
- **Inputs/outputs:** From **Summarise Transcript** → to **Prepare Mongo Document**
- **Edge cases / failures:**
  - Uses `meetingTitle` but Mongo insert uses `$json.title` (field mismatch risk)
  - No escaping/sanitization for HTML (if summary/title contain `<`/`&`)
- **Version notes:** Type version 2.

---

### Block 3 — Database Storage (MongoDB)

**Overview:**  
Builds a MongoDB document and inserts it into `meeting_summaries` before any emails are sent, enabling durable storage and later auditing.

**Nodes involved:**
- **Prepare Mongo Document** (Set)
- **Insert into MongoDB** (MongoDB)

#### Node: Prepare Mongo Document
- **Type / role:** `Set` — intended to assemble/normalize fields for MongoDB.
- **Config choices:** No explicit fields are shown; only `options: {}`.
- **Inputs/outputs:** From **Format Email** → to **Insert into MongoDB**
- **Edge cases / failures:**
  - As configured, it likely doesn’t add fields like `timestamp`, `meetingId`, etc. (but the Mongo node expects them). This can lead to `createdAt`, `meetingId`, etc. being `null`/empty.
- **Version notes:** Type version 3.

#### Node: Insert into MongoDB
- **Type / role:** `MongoDB` — inserts the meeting summary record.
- **Config choices:**
  - Operation: **insert**
  - Collection: `meeting_summaries`
  - Fields inserted via expressions:
    - `title = {{ $json.title }}`
    - `summary = {{ $json.summary }}`
    - `mailSent = false`
    - `createdAt = {{ $json.timestamp }}`
    - `meetingId = {{ $json.meetingId }}`
    - `transcript = {{ $json.transcript }}`
    - `htmlContent = {{ $json.htmlContent }}`
    - `emailSubject = {{ $json.emailSubject }}`
- **Inputs/outputs:** From **Prepare Mongo Document** → to **Extract Participant Emails**
- **Edge cases / failures:**
  - Field mismatches are likely: earlier code uses `transcript_text`, `meetingTitle`; here it expects `transcript`, `title`, `timestamp`, `meetingId`
  - Auth/connection errors to MongoDB
  - Schema/validation rules on the collection could reject inserts
- **Version notes:** Type version 1.

---

### Block 4 — Mailchimp Sync & Campaign Send

**Overview:**  
Creates one Mailchimp audience member per participant (add/update), then creates a campaign, sets the HTML content, sends it immediately, and checks whether it was sent.

**Nodes involved:**
- **Extract Participant Emails** (Code)
- **Split Participants** (Split In Batches)
- **Upsert Member** (Mailchimp)
- **Merge For Campaign** (Merge)
- **Create Campaign** (Mailchimp)
- **Set Campaign Content** (Mailchimp)
- **Send Campaign** (Mailchimp)
- **Mail Sent?** (IF)

#### Node: Extract Participant Emails
- **Type / role:** `Code` — converts participant list into items Mailchimp can process.
- **Config choices (interpreted):**
  - Hardcodes `listId = '{{YOUR_MAILCHIMP_LIST_ID}}'` (placeholder)
  - Reads `participants = $json.participants || []`
  - Emits one item per participant: `{ email, listId, htmlContent, emailSubject }`
- **Inputs/outputs:** From **Insert into MongoDB** → to **Split Participants**
- **Edge cases / failures:**
  - If `participants` isn’t present (common if meeting data wasn’t merged into transcript result), zero items result → downstream campaign creation may still run depending on merge behavior (but likely not reached because Split In Batches won’t output)
  - No deduplication, despite sticky note claiming dedupe; duplicates may cause repeated upserts
- **Version notes:** Type version 2.

#### Node: Split Participants
- **Type / role:** `Split In Batches` — processes participants sequentially.
- **Config choices:** Defaults (batch size not explicit).
- **Inputs/outputs:** From **Extract Participant Emails** → to **Upsert Member**
- **Edge cases / failures:**
  - Like “Split Meetings”, there is **no explicit loop-back** to fetch subsequent batches; may only process first batch.
- **Version notes:** Type version 3.

#### Node: Upsert Member
- **Type / role:** `Mailchimp` — adds or updates a list member.
- **Config choices:**
  - Operation: `addUpdate`
  - (List/Audience id and email are typically required by the node; not visible here, likely relies on incoming JSON fields but may actually require mapping in parameters.)
- **Inputs/outputs:** From **Split Participants** → to **Merge For Campaign**
- **Edge cases / failures:**
  - Missing required parameters (audience/list id, email) in node config will error even if present in JSON
  - Mailchimp requires member hash for some endpoints; node handles internally, but email must be valid
  - Rate limiting if large participant lists
- **Version notes:** Type version 1.

#### Node: Merge For Campaign
- **Type / role:** `Merge` — “wait” mode used to synchronize before creating the campaign.
- **Config choices:**
  - Mode: `wait` (typically waits for both inputs; however only one input is connected in this workflow)
- **Inputs/outputs:** Input from **Upsert Member** → output to **Create Campaign**
- **Edge cases / failures:**
  - In “wait” mode, with only one incoming branch connected, behavior can be confusing; it may still pass through but does not truly “wait for all participants” unless wired with a proper completion signal/second input.
- **Version notes:** Type version 2.

#### Node: Create Campaign
- **Type / role:** `Mailchimp` — creates a new campaign.
- **Config choices:** Resource `campaign`, operation `create`. Sender, list, subject settings are not shown (usually required).
- **Inputs/outputs:** From **Merge For Campaign** → to **Set Campaign Content**
- **Edge cases / failures:**
  - If audience/list and campaign settings (from name, reply-to, etc.) are not configured, creation fails
- **Version notes:** Type version 1.

#### Node: Set Campaign Content
- **Type / role:** `Mailchimp` — sets campaign HTML content.
- **Config choices:** Resource `campaign`, operation `setContent`. Requires campaign id and HTML; not shown how mapped.
- **Inputs/outputs:** From **Create Campaign** → to **Send Campaign**
- **Edge cases / failures:**
  - If HTML content isn’t mapped from `$json.htmlContent`, content may be blank
- **Version notes:** Type version 1.

#### Node: Send Campaign
- **Type / role:** `Mailchimp` — sends the created campaign.
- **Config choices:**
  - Resource `campaign`, operation `send`
  - `campaignId = {{ $json.id }}` (assumes previous node outputs campaign id as `id`)
- **Inputs/outputs:** From **Set Campaign Content** → to **Mail Sent?**
- **Edge cases / failures:**
  - `$json.id` might refer to another id depending on node output structure
  - Mailchimp can reject sends if list is empty, not compliant, missing required settings, etc.
- **Version notes:** Type version 1.

#### Node: Mail Sent?
- **Type / role:** `IF` — checks send success.
- **Config choices:**
  - Condition: `{{ $json.status || 'sent' }}` equals `sent`
  - Note: The `|| 'sent'` default makes the check **pass even if status is missing**, which can mask failures.
- **Inputs/outputs:** From **Send Campaign**. No outgoing connections shown.
- **Edge cases / failures:**
  - False positives due to defaulting to `'sent'`
  - No follow-up action to update Mongo `mailSent` to true/false
- **Version notes:** Type version 2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📒  Meeting Notes Distributor – Overview | Sticky Note | Documentation / overview | — | — | ## How it works; Setup steps 1–7 (Meetings API, Transcription API, MongoDB, Mailchimp, replace placeholders, schedule cadence, manual test) |
| Trigger & Retrieval (Info) | Sticky Note | Documentation for trigger/retrieval block | — | — | ## Trigger & Retrieval (schedule → fetch → IF empty → split in batches) |
| Transcription & Summary (Info) | Sticky Note | Documentation for transcription/summarization | — | — | ## Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Database Storage (Info) | Sticky Note | Documentation for MongoDB storage | — | — | ## Database Storage (Set doc → insert → track mailSent flag) |
| Mailchimp Notifications (Info) | Sticky Note | Documentation for Mailchimp sending | — | — | ## Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Daily Schedule | Schedule Trigger | Runs workflow every 24h | — | Fetch Meetings | Trigger & Retrieval (schedule → fetch → IF empty → split in batches) |
| Fetch Meetings | HTTP Request | Pull available recordings list | Daily Schedule | Any Meetings? | Trigger & Retrieval (schedule → fetch → IF empty → split in batches) |
| Any Meetings? | IF | Continue only if meetings exist | Fetch Meetings | Split Meetings | Trigger & Retrieval (schedule → fetch → IF empty → split in batches) |
| Split Meetings | Split In Batches | Process meetings serially | Any Meetings? | Initiate Transcription | Trigger & Retrieval (schedule → fetch → IF empty → split in batches) |
| Initiate Transcription | HTTP Request | Create transcription job | Split Meetings | Wait 2 min | Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Wait 2 min | Wait | Delay before polling transcript | Initiate Transcription | Get Transcript | Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Get Transcript | HTTP Request | Fetch job status + transcript | Wait 2 min | Transcript Ready? | Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Transcript Ready? | IF | Continue only if status=completed | Get Transcript | Summarise Transcript | Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Summarise Transcript | Code | Generate short summary from transcript | Transcript Ready? | Format Email | Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Format Email | Code | Build HTML body + subject | Summarise Transcript | Prepare Mongo Document | Transcription & Summary (post job → wait → poll → IF completed → summary → HTML template) |
| Prepare Mongo Document | Set | Normalize fields for DB insert | Format Email | Insert into MongoDB | Database Storage (Set doc → insert → track mailSent flag) |
| Insert into MongoDB | MongoDB | Insert record into `meeting_summaries` | Prepare Mongo Document | Extract Participant Emails | Database Storage (Set doc → insert → track mailSent flag) |
| Extract Participant Emails | Code | Map participants to Mailchimp items | Insert into MongoDB | Split Participants | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Split Participants | Split In Batches | Process participants serially | Extract Participant Emails | Upsert Member | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Upsert Member | Mailchimp | Add/update each participant | Split Participants | Merge For Campaign | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Merge For Campaign | Merge | Attempt to wait before campaign creation | Upsert Member | Create Campaign | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Create Campaign | Mailchimp | Create one-off campaign | Merge For Campaign | Set Campaign Content | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Set Campaign Content | Mailchimp | Set campaign HTML content | Create Campaign | Send Campaign | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Send Campaign | Mailchimp | Send campaign immediately | Set Campaign Content | Mail Sent? | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |
| Mail Sent? | IF | Check send status | Send Campaign | — | Mailchimp Notifications (dedupe + upsert members → create campaign → set content → send → IF success/fail) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Create **HTTP Header Auth** credential named **“Meetings API”** containing the required authorization header(s) for your meetings platform.
   2. Create **HTTP Header Auth** credential named **“Transcription API”** containing the API key/header(s) for the transcription service.
   3. Create **MongoDB** credential with insert/write permission to the target database.
   4. Create **Mailchimp** credential with permission to manage audience members and campaigns.

2) **Add the trigger**
   1. Add node **Schedule Trigger** named **Daily Schedule**.
   2. Set interval to **Every 24 hours** (or your preferred cadence).

3) **Fetch meetings**
   1. Add **HTTP Request** named **Fetch Meetings**.
   2. Method: **GET**
   3. URL: `https://api.yourmeetings.com/v1/recordings?status=available` (replace with real endpoint)
   4. Authentication: **Predefined Credential Type** → `httpHeaderAuth` → select **Meetings API** credential.
   5. Connect **Daily Schedule → Fetch Meetings**.

4) **Guard: only proceed if meetings exist**
   1. Add **IF** node named **Any Meetings?**
   2. Condition: Number → Value1: `{{ $items("Fetch Meetings").length }}`; Operation: **larger**; Value2: `0`
   3. Connect **Fetch Meetings → Any Meetings?**

5) **Process meetings in sequence**
   1. Add **Split In Batches** named **Split Meetings** (choose batch size explicitly, e.g., 1).
   2. Connect **Any Meetings? (true) → Split Meetings**

6) **Create transcription job**
   1. Add **HTTP Request** named **Initiate Transcription**
   2. Method: **POST**
   3. URL: `https://api.transcriptionservice.com/v1/jobs`
   4. Auth: `httpHeaderAuth` → select **Transcription API**
   5. Configure request body to include the meeting recording URL (example conceptually: send `$json.recordingUrl`), matching your transcription provider’s API.
   6. Connect **Split Meetings → Initiate Transcription**

7) **Wait**
   1. Add **Wait** node named **Wait 2 min**
   2. Configure an actual delay of **2 minutes** in the node parameters.
   3. Connect **Initiate Transcription → Wait 2 min**

8) **Poll transcription result**
   1. Add **HTTP Request** named **Get Transcript**
   2. Method: **GET**
   3. URL: `https://api.transcriptionservice.com/v1/jobs/{{ $json.id }}` (adapt `id` field to whatever the create-job response returns)
   4. Auth: `httpHeaderAuth` → **Transcription API**
   5. Connect **Wait 2 min → Get Transcript**

9) **Continue only if completed**
   1. Add **IF** node named **Transcript Ready?**
   2. Condition: String → Value1 `{{ $json.status }}` equals `completed` (adapt to provider statuses).
   3. Connect **Get Transcript → Transcript Ready?**

10) **Summarise**
   1. Add **Code** node named **Summarise Transcript**
   2. Paste the provided JS logic (or replace with your own/LLM).
   3. Ensure it reads the correct transcript field (currently expects `transcript_text`).
   4. Connect **Transcript Ready? (true) → Summarise Transcript**

11) **Format email**
   1. Add **Code** node named **Format Email**
   2. Paste the provided HTML/subject generation JS.
   3. Ensure the title field matches your meeting data (`meetingTitle` vs `title`).
   4. Connect **Summarise Transcript → Format Email**

12) **Prepare MongoDB document**
   1. Add **Set** node named **Prepare Mongo Document**
   2. Add/normalize fields required for DB insert (at minimum: `title`, `meetingId`, `timestamp/createdAt`, `transcript`, `participants`, plus `summary`, `htmlContent`, `emailSubject`).
   3. Connect **Format Email → Prepare Mongo Document**

13) **Insert into MongoDB**
   1. Add **MongoDB** node named **Insert into MongoDB**
   2. Operation: **insert**
   3. Collection: `meeting_summaries`
   4. Map fields (title, summary, transcript, htmlContent, emailSubject, meetingId, createdAt, mailSent=false).
   5. Select your **MongoDB credential**
   6. Connect **Prepare Mongo Document → Insert into MongoDB**

14) **Extract participant emails**
   1. Add **Code** node named **Extract Participant Emails**
   2. Use code that:
      - reads `$json.participants`
      - emits items with at least `{ email, listId }`
      - carries `htmlContent` and `emailSubject`
   3. Replace `{{YOUR_MAILCHIMP_LIST_ID}}` with your real Mailchimp audience/list id (or inject via environment variable).
   4. Connect **Insert into MongoDB → Extract Participant Emails**

15) **Process participants & upsert into Mailchimp**
   1. Add **Split In Batches** named **Split Participants** (batch size 1 recommended).
   2. Connect **Extract Participant Emails → Split Participants**
   3. Add **Mailchimp** node named **Upsert Member**, operation **addUpdate**
   4. Configure it to use:
      - Audience/List ID from incoming item (or fixed in node)
      - Email from incoming item
   5. Connect **Split Participants → Upsert Member**

16) **Gate before campaign creation**
   1. Add **Merge** node named **Merge For Campaign** with mode **wait**
   2. Connect **Upsert Member → Merge For Campaign**
   3. (If you truly need “wait until all participants processed”, add a proper completion path/second input; current single-input wiring does not guarantee that.)

17) **Create, populate, and send campaign**
   1. Add **Mailchimp** node **Create Campaign** (resource: `campaign`, operation: `create`)
   2. Configure required campaign settings: audience/list, from name/email, subject (can use `$json.emailSubject`), etc.
   3. Connect **Merge For Campaign → Create Campaign**
   4. Add **Mailchimp** node **Set Campaign Content** (resource: `campaign`, operation: `setContent`)
   5. Map HTML to `$json.htmlContent`
   6. Connect **Create Campaign → Set Campaign Content**
   7. Add **Mailchimp** node **Send Campaign** (resource: `campaign`, operation: `send`)
   8. Set Campaign ID to `{{ $json.id }}` (or adapt to actual output)
   9. Connect **Set Campaign Content → Send Campaign**

18) **Check send status**
   1. Add **IF** node **Mail Sent?**
   2. Condition: check a reliable “sent” indicator returned by Mailchimp (avoid defaulting to `'sent'` if missing).
   3. Connect **Send Campaign → Mail Sent?**
   4. (Optional but recommended) Add a MongoDB **update** step to set `mailSent=true` on success / store error details on failure.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow runs every 24 hours, fetches new recordings, transcribes, generates a summary, stores in MongoDB, syncs participants to Mailchimp, creates a campaign, and sends it. Failures/empty lists are handled via IF branches. | Sticky note: “📒 Meeting Notes Distributor – Overview” |
| Setup steps: create “Meetings API” HTTP credential; “Transcription API” HTTP credential; MongoDB credential; Mailchimp credential; replace placeholder URLs/list IDs/sender info; adjust schedule; run once manually to validate. | Sticky note: “📒 Meeting Notes Distributor – Overview” |
| Trigger & Retrieval block description (schedule → HTTP fetch → IF list empty stops → Split in Batches for serial processing). | Sticky note: “Trigger & Retrieval (Info)” |
| Transcription & Summary block description (submit recording URL → wait → poll once → IF completed → code summary → code HTML template). | Sticky note: “Transcription & Summary (Info)” |
| Database Storage block description (assemble clean doc including metadata/transcript/summary/timestamp/mailSent flag → insert into MongoDB). | Sticky note: “Database Storage (Info)” |
| Mailchimp Notifications description (dedupe + upsert members; create campaign; set content; send; IF success/failure; can swap ESP by replacing a few nodes). | Sticky note: “Mailchimp Notifications (Info)” |