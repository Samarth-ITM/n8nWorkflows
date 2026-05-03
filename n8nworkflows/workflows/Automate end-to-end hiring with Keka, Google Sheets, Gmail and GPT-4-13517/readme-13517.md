Automate end-to-end hiring with Keka, Google Sheets, Gmail and GPT-4

https://n8nworkflows.xyz/workflows/automate-end-to-end-hiring-with-keka--google-sheets--gmail-and-gpt-4-13517


# Automate end-to-end hiring with Keka, Google Sheets, Gmail and GPT-4

## 1. Workflow Overview

**Workflow title:** *Automate end-to-end hiring with Keka, Google Sheets, Gmail and GPT-4*  
**Workflow name (in JSON):** *Overall Recruitment & Improvement Process Automation*

This workflow automates an end-to-end recruitment pipeline by synchronizing **jobs and candidates from Keka**, managing the **master recruitment tracker in Google Sheets**, sending **stage-based notifications via Gmail**, and using **GPT‑4 (OpenAI via LangChain node)** to assist with screening / feedback processing. It also integrates with an external system referred to as **“Jia”** (token managed via sub-workflows), used for interviews/reports/status updates.

### Logical blocks (based on dependencies)

1.1 **Job & candidate ingestion from Keka → Master Sheet upsert**  
1.2 **Recruiter notification for newly added candidates + sheet state update**  
1.3 **Invite/interview handling: resume fetch + recruiter notification + sheet update**  
1.4 **AI screening (OpenAI) → update stage → notify recruiters + rejection email branch**  
1.5 **Jia interview report polling → score extraction → final status update**  
1.6 **Stage progression nudges (Tech / HR / Leadership / Approval Pending) + sheet updates**  
1.7 **Interview scheduling emails (interviewer + candidate) + sheet update**  
1.8 **HR final decision notifications + hired notifications + final sheet status update**  
1.9 **Jia status synchronization (update Jia + update sheet)**  
1.10 **Cross-cutting: token management via Execute Workflow nodes + Wait/rate limiting**

**Entry points:** multiple **Schedule Trigger** nodes (11 total), each starting a different “maintenance” lane.

---

## 2. Block-by-Block Analysis

### 2.1 Block 1 — Job & candidate ingestion from Keka → Master Sheet upsert

**Overview:** Pulls open jobs from Keka, iterates job-by-job, paginates applications/candidates, normalizes/deduplicates them, maps recruiters/hiring team, and prepares rows for Google Sheets comparison and insertion.

**Nodes involved:**
- Schedule Trigger 1
- Init Run Storage
- Keka Token Manager (Execute Workflow)
- Get Keka Jobs (HTTP Request)
- Normalize Jobs (Code)
- Accumulate Jobs + Hiring Team (Code)
- Loop Over Items (SplitInBatches)
- Deduplicate Candidates (Code)
- Resolve Job Recruiters (Code)
- Prepare Candidate Rows (Code)
- Read Master Recruitment Sheet (Google Sheets)
- Filter Sourced Candidates (Code)
- Compare With Sheet (Code)
- Loop Over Items1 (SplitInBatches)
- Append row in sheet (Google Sheets)
- Wait1 (Wait)
- Send a message5 (Gmail)
- (Candidate pagination chain)
  - Init Candidate Pagination (Code)
  - Fetch job applications (HTTP Request)
  - Normalize Candidates (Code)
  - Accumulate Candidates (Code)
  - More Candidate Pages? (IF)
  - Next Candidate Page (Code)

**Node details (key points):**

- **Schedule Trigger 1 (scheduleTrigger v1.3)**  
  *Role:* periodic start of ingestion lane.  
  *Failure modes:* misconfigured schedule/timezone; overlapping runs if schedule too frequent.

- **Init Run Storage (Code v2)**  
  *Role:* initializes runtime memory/state (typically maps/arrays used downstream, like “notified candidates”).  
  *Edge cases:* if it assumes certain env vars or prior execution state, may throw.

- **Keka Token Manager (Execute Workflow v1.3)**  
  *Role:* calls a sub-workflow that returns/sets a valid Keka access token (exact interface not included in JSON).  
  *Integration risk:* sub-workflow not present, wrong output shape, token expiration not handled.

- **Get Keka Jobs (HTTP Request v4.3, retryOnFail=true)**  
  *Role:* fetch job list from Keka API.  
  *Failure modes:* 401/403 (token invalid), pagination not handled here, request limits/timeouts.

- **Normalize Jobs (Code v2)**  
  *Role:* transforms raw Keka jobs into a normalized schema used in later blocks.  
  *Edge cases:* missing fields from Keka API; null recruiter/hiring team fields.

- **Accumulate Jobs + Hiring Team (Code v2)**  
  *Role:* enrich/accumulate job items and attach hiring team context for candidate routing.

- **Loop Over Items (SplitInBatches v3)**  
  *Role:* iterates over jobs (batch loop).  
  *Connections:* outputs to both **Deduplicate Candidates** and **Init Candidate Pagination** (parallel lanes from the batch).  
  *Failure modes:* incorrect batch size can cause memory pressure or too many API calls.

- **Init Candidate Pagination (Code v2) → Fetch job applications (HTTP) → Normalize Candidates (Code) → Accumulate Candidates (Code)**  
  *Role:* sets pagination params (page cursor/page number), requests job applications per job, normalizes candidates, accumulates them across pages.  
  *More Candidate Pages? (IF v2.3):* if more pages exist → **Next Candidate Page** which loops back into **Init Candidate Pagination**. Otherwise returns to **Loop Over Items** to proceed with next job.  
  *Typical failure:* infinite loop if “hasMore” logic is wrong; duplicates if page cursor repeats.

- **Deduplicate Candidates (Code v2)**  
  *Role:* remove duplicates after accumulation and/or per-job overlaps.

- **Resolve Job Recruiters (Code v2)**  
  *Role:* map job → recruiter email(s) / recruiter metadata (likely used later for grouping and emailing).  
  *Edge cases:* recruiter not found, invalid email format.

- **Prepare Candidate Rows (Code v2)**  
  *Role:* shape items into sheet row payload format (columns aligned to master tracker).

- **Read Master Recruitment Sheet (Google Sheets v4.7, alwaysOutputData=true)**  
  *Role:* reads existing rows for dedupe/upsert.  
  *Failure modes:* sheet permissions, wrong spreadsheet/tab IDs, API quota.

- **Filter Sourced Candidates (Code v2)**  
  *Role:* keeps only candidates in “sourced” or desired stage (based on sheet/Keka attributes).

- **Compare With Sheet (Code v2)**  
  *Role:* compares normalized candidates vs sheet rows to identify “new rows to append” and “existing rows to update/notify”.  
  *Connections:* feeds **Loop Over Items1** (append lane) and **Group by Recruiter + job id** (notify lane).

- **Loop Over Items1 (SplitInBatches v3) → Append row in sheet (Google Sheets v4.7) → Wait1 (Wait v1.1) → Send a message5 (Gmail v2.2)**  
  *Role:* append new candidates to sheet, then throttle via Wait, then send a Gmail message (likely confirmation/notification).  
  *Edge cases:* duplicates if compare logic fails; row append failures; Gmail auth errors.

**Sub-workflow references:** Keka Token Manager (called here).

---

### 2.2 Block 2 — Recruiter notification for newly added candidates + “notified” memory + sheet update

**Overview:** Groups new candidates by recruiter + job, emails recruiters, tracks which candidates were notified (in workflow memory), then updates corresponding sheet rows.

**Nodes involved:**
- Group by Recruiter + job id
- Send a message to recruiter (Gmail)
- Mark Candidates Notified (Memory) (Code)
- Prepare Rows For Sheet Update (Code)
- Loop Over Items2 (SplitInBatches)
- Wait (Wait)
- Update row in sheet5 (Google Sheets)

**Node details:**
- **Group by Recruiter + job id (Code v2)**  
  *Role:* create groups to send one email per recruiter/job rather than per candidate.  
  *Failure modes:* missing recruiter email leads to empty groups.

- **Send a message to recruiter (Gmail v2.2, retryOnFail=true)**  
  *Role:* send summary email for a recruiter/job group.  
  *Failure modes:* Gmail OAuth misconfiguration; invalid “to” addresses; rate limiting.

- **Mark Candidates Notified (Memory) (Code v2)**  
  *Role:* stores notified candidate identifiers in memory for later sheet updates (“notified=true”, timestamp, etc.).  
  *Edge case:* memory is per-execution; if workflow relies on persistence across runs, it will not persist unless using external store.

- **Prepare Rows For Sheet Update (Code v2) → Loop Over Items2 (SplitInBatches v3) → Wait (Wait v1.1) → Update row in sheet5 (Google Sheets v4.7)**  
  *Role:* transforms memory into sheet update payload and applies it with throttling.  
  *Failure modes:* wrong row identifiers; sheet update conflicts if rows moved/filtered.

---

### 2.3 Block 3 — Resume fetch + invited candidate accumulation + recruiter notification

**Overview:** Reads master sheet entries, extracts role/YOE, fetches resumes from Keka, optionally triggers “Jia interview”, accumulates invited candidates, groups by recruiter/job, notifies recruiters, and updates sheet.

**Nodes involved:**
- Jia Token Manager (Execute Workflow)
- Read Master Recruitment Sheet1 (Google Sheets)
- Loop Over Items3 (SplitInBatches)
- Extract Role + YOE (Code)
- Fetch Candidate Resume (HTTP Request)
- Download Resume (HTTP Request)
- Jia Inteview Sent (HTTP Request, **disabled**)
- Dummy data for jia inteview (Set)
- Accumulate Invited Candidates (Code)
- Group by Recruiter + job id1 (Code)
- Notify Recruiters (Gmail)
- Prepare Sheet Updates from Memory (Code)
- Update row in sheet6 (Google Sheets)
- Jia Token Manager3 (Execute Workflow) (upstream of Jia Token Manager)

**Node details:**
- **Jia Token Manager3 → Jia Token Manager (Execute Workflow v1.3)**  
  *Role:* chained token setup (Keka token manager triggers Jia manager elsewhere too). Exact contract unknown.  
  *Risk:* missing referenced workflows.

- **Read Master Recruitment Sheet1 (Google Sheets v4.7)**  
  *Role:* obtains candidate rows for this stage (likely “invite pending”).

- **Loop Over Items3 (SplitInBatches v3)**  
  *Role:* per-candidate processing. Outputs to both **Group by Recruiter + job id1** and **Extract Role + YOE** (parallel).

- **Extract Role + YOE (Code v2)**  
  *Role:* parse role name and years-of-experience from sheet fields or candidate profile.

- **Fetch Candidate Resume (HTTP v4.3) → Download Resume (HTTP v4.3)**  
  *Role:* obtain resume URL then download binary.  
  *Failure modes:* expired URLs, file too large, content-type mismatch.

- **Jia Inteview Sent (HTTP v4.3, disabled)**  
  *Role:* would send invite to Jia interview platform. Disabled means current execution skips it.  
  *Replacement path:* **Dummy data for jia inteview (Set v3.4)** produces a stand-in payload.

- **Accumulate Invited Candidates (Code v2)**  
  *Role:* memory accumulation for later recruiter notification + sheet update.

- **Group by Recruiter + job id1 (Code) → Notify Recruiters (Gmail)**  
  *Role:* one recruiter email per job group listing invited candidates.

- **Prepare Sheet Updates from Memory (Code) → Update row in sheet6 (Google Sheets)**  
  *Role:* mark invite/interview sent status in sheet.

---

### 2.4 Block 4 — AI screening (GPT‑4) → update stage → recruiter email + rejection branch

**Overview:** Reads candidates requiring AI screening, sends them to GPT‑4, parses the model output, merges back, updates stage status in Sheets, groups results, notifies recruiters, and separately sends rejection emails after a wait.

**Nodes involved:**
- Schedule Trigger 2
- Read Master Recruitment For Feedback (Google Sheets)
- Filter Candidates for AI Screening (Code)
- Message a model (OpenAI LangChain)
- PARSE the OpenAI output (Code)
- Merge (Merge)
- Update Stage Status (Google Sheets)
- Group by recruiter and job id (Code)
- Notify Recruiters1 (Gmail)
- Keka Token Manager3 (Execute Workflow) → Jia Token Manager (Execute Workflow)
- Filter rejected candidates (Code)
- Loop Over Items7 (SplitInBatches)
- Wait2 (Wait)
- Send Rejection Mails to candidates (Gmail)

**Node details:**
- **Filter Candidates for AI Screening (Code v2)**  
  *Role:* choose candidates where AI screening is needed; also provides a second input to Merge (it connects to Merge input 1).  
  *Edge cases:* if it outputs zero items, downstream OpenAI calls do nothing.

- **Message a model (@n8n/n8n-nodes-langchain.openAi v2.1)**  
  *Role:* calls OpenAI (intended GPT‑4) with candidate context.  
  *Requirements:* OpenAI credentials; model availability; token limits.  
  *Failure modes:* rate limiting, prompt too large, invalid JSON responses.

- **PARSE the OpenAI output (Code v2)**  
  *Role:* parse structured data out of model response (likely score/recommendation/stage).  
  *Failure modes:* model output format drift; JSON parse errors.

- **Merge (Merge v3.2)**  
  *Role:* merges parsed AI results with original candidate items so sheet updates have full context.  
  *Risk:* wrong merge mode can mismatch items.

- **Update Stage Status (Google Sheets v4.7)**  
  *Role:* updates candidate stage/status columns.

- **Group by recruiter and job id (Code v2) → Notify Recruiters1 (Gmail v2.2)**  
  *Role:* notify recruiters of screened outcomes.  
  *Next hop:* **Notify Recruiters1** triggers **Keka Token Manager3**, which then triggers Jia-related processing lane (token chaining).

- **Filter rejected candidates (Code v2) → Loop Over Items7 → Wait2 → Send Rejection Mails to candidates (Gmail)**  
  *Role:* send rejection emails to candidates identified as rejected by AI/logic, throttled by Wait.  
  *Edge cases:* accidental rejection if threshold logic incorrect; email deliverability.

---

### 2.5 Block 5 — Jia interview report polling → score extraction → final status update (sheet)

**Overview:** Reads candidates expected to have completed AI interview (Jia), fetches report metadata, downloads the report, extracts text, parses overall score, merges with candidate records, sets final status, and updates sheet.

**Nodes involved:**
- Schedule Trigger 3
- Jia Token Manager1 (Execute Workflow)
- Read Master Recruitment For Feedback1 (Google Sheets)
- If (IF)
- Filter candidates (Code)
- Fetch the AI interview Report (HTTP, executeOnce=true)
- Filter interview completed candidates (Code)
- Compare Candidates (Code)
- Loop Over Items4 (SplitInBatches)
- Download Report (HTTP)
- Extract from File (ExtractFromFile)
- Parse Overall Score (Code)
- Merge1 (Merge)
- Final Status (Code)
- Update row in sheet (Google Sheets)

**Node details:**
- **If (IF v2.3)**  
  *Role:* acts as a gate; only “true” path continues to processing (false path unused).  
  *Risk:* if condition incorrect, whole lane never runs.

- **Fetch the AI interview Report (HTTP v4.3, executeOnce=true)**  
  *Role:* fetch report list or report status from Jia. `executeOnce` means it runs once per workflow execution (not per item).  
  *Failure modes:* stale data if multiple candidates require different reports; API auth.

- **Loop Over Items4**  
  *Role:* per candidate: download report → extract → parse score.

- **Extract from File (extractFromFile v1.1)**  
  *Role:* extract text from downloaded file (likely PDF).  
  *Failure modes:* unsupported file types, corrupted PDFs, large file timeouts.

- **Parse Overall Score (Code v2)**  
  *Role:* parse numeric score/summary from extracted text.  
  *Edge case:* report format changes break regex/parser.

- **Merge1 → Final Status → Update row in sheet**  
  *Role:* merge parsed score back with candidate record; compute final status; update sheet.  
  *Connection note:* Update row in sheet loops back to Loop Over Items4 per graph; ensure batch loop termination is correct to avoid unintended loops.

---

### 2.6 Block 6 — “Approval Pending” notifications + sheet update

**Overview:** Reads master sheet for items needing “Approval Pending” handling, groups by recruiter/job, emails recruiters, prepares update payload, and marks sheet rows.

**Nodes involved:**
- Schedule Trigger 4
- Read Master Recruitment For Feedback2 (Google Sheets)
- If1 (IF)
- Filter Candidates (Code)
- Group  by recruiter + job id (Code)
- Notify Recruiters2 (Gmail)
- Prepare Update Payload (Code)
- Update row As Approval Pending (Google Sheets)

**Key risks:** sheet row matching; email routing; if-condition misgating.

---

### 2.7 Block 7 — Stage nudges: Technical interview, HR discussion, Leadership connect + sheet updates

**Overview:** Three similar lanes trigger on schedules, read master sheet, filter candidates by stage, group by recruiter/job, notify recruiters to schedule next round, and update sheet statuses accordingly.

**Nodes involved (Tech lane):**
- Schedule Trigger 6
- Read Master Recruitment For Feedback3
- If3
- Filter Candidates1
- Group  by recruiter + job id1
- Notify Recruiters to Technical Interview (Gmail)
- Prepare Update Payload1 (Code)
- Update row As TR interview Pending (Google Sheets)

**Nodes involved (HR discussion lane):**
- Schedule Trigger 7
- Read Master Recruitment For Feedback4
- If7
- Filter Candidates2
- Group  by recruiter + job id2
- Notify Recruiters to schedule HR Discussion Round (Gmail)
- Prepare Update Payload2 (Code)
- Update row As Hr interview Pending (Google Sheets)

**Nodes involved (Leadership lane):**
- Schedule Trigger 8
- Read Master Recruitment For Feedback5
- If8
- Filter Candidates3
- Group  by recruiter + job id3
- Notify Recruiters to schedule Leadership Connect Round (Gmail)
- Prepare Update Payload3 (Code)
- Update row As LeaderShip round pending (Google Sheets)

**Failure modes:** incorrect stage filters leading to spam; Gmail quota; wrong sheet update ranges.

---

### 2.8 Block 8 — Interview scheduling emails (interviewer + candidate) + sheet update

**Overview:** Reads sheet rows where interview is pending, filters/normalizes, resolves interviewer emails, sends interview emails to interviewer and candidate, then updates sheet status.

**Nodes involved:**
- Schedule Trigger 9
- Read sheet with intetrview pending (Google Sheets)
- Filter Candidates4 (Code)
- If9 (IF)
- Filter rows (Code)
- Loop Over Items6 (SplitInBatches)
- Resolve emails (Code)
- Send email to interviewer (Gmail)
- Pass the data (Code)
- Send email to Candidate (Gmail)
- Prepare payload for sheets (Code)
- Update Status1 (Google Sheets)

**Key risks:** sending emails to wrong recipients due to resolution logic; missing calendar links; sheet updates failing after email sends (partial completion).

---

### 2.9 Block 9 — HR interview pending sheet → final HR decision notifications + final status update

**Overview:** Reads sheet rows with HR interview pending, optionally fetches feedback report (disabled), merges normalized feedback into candidate items, filters final HR decisions, groups by recruiter/job, emails status updates, updates final status in Sheets. Also has a rejection email branch for rejected candidates.

**Nodes involved:**
- Schedule Trigger 10
- Keka Token Manager2 (Execute Workflow)
- Read sheet with HR intetrview pending (Google Sheets)
- Filter Candidates5 (Code)
- If4 (IF)
- Normalize Rows (Code)
- Merge2 (Merge)
- Loop Over Items5 (SplitInBatches)
- Filter Final HR Decisions (Code)
- Group by Recruiter + Job ID (Code)
- Send a message (Gmail)
- Prepare Google Sheets Update Payload (Code)
- Update Final Status (Google Sheets)
- (feedback fetch sub-lane)
  - Dummy (Code)
  - Fetch feedback report (HTTP, **disabled**)
  - Normalize Feedback (Code)
- (rejection sub-lane)
  - Filter rejected candidates1 (Code)
  - Loop Over Items8 (SplitInBatches)
  - Wait3 (Wait)
  - Send Rejection Mails to candidates1 (Gmail)

**Important connection note:** `Dummy → Fetch feedback report` exists but HTTP node is disabled; if left disabled, Merge2 receives only Normalize Rows side (depending on merge mode it may block or pass-through). Validate Merge2 configuration in UI.

---

### 2.10 Block 10 — Hired status notifications + sheet update

**Overview:** Reads sheet rows with “hired” status, groups by recruiter/job, emails relevant parties, and updates sheet status.

**Nodes involved:**
- Schedule Trigger 11
- Read sheet with Interview status as hired (Google Sheets)
- If6 (IF)
- Pass (Code)
- Group by recruiter + job id (Code)
- Send a message1 (Gmail)
- Prepare Google Sheets Update Payload1 (Code)
- Update Status (Google Sheets)
- Prepare Google Sheets Update Payload1 (Code)
- Pass (Code)
- 882358bc-… “Pass” is the code node named “Pass” (already listed)

**Failure modes:** repeated notifications if “already notified” flag not used; sheet rows not uniquely identified.

---

### 2.11 Block 11 — Jia status synchronization (update Jia + update sheet)

**Overview:** Periodically checks candidates requiring Jia status updates, updates status in Jia via API, and then writes result back to sheet.

**Nodes involved:**
- Schedule Trigger 5
- Jia Token Manager2 (Execute Workflow)
- Read Master Recruitment For Feedback6 (Google Sheets)
- If2 (IF)
- Filter Candidates6 (Code)
- Loop Over Items9 (SplitInBatches)
- Update Jia status (HTTP)
- Update row in sheet1 (Google Sheets)

**Failure modes:** Jia API auth/status mapping errors; partial updates; inconsistent states between Jia and sheet.

---

## 3. Summary Table (all nodes)

> Sticky notes exist but their content is empty (`""`) in the JSON, so the **Sticky Note** column is blank for all rows.

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger 1 | scheduleTrigger | Start Keka ingestion lane | — | Init Run Storage | |
| Init Run Storage | code | Initialize in-run memory/state | Schedule Trigger 1 | Keka Token Manager | |
| Keka Token Manager | executeWorkflow | Obtain Keka token via sub-workflow | Init Run Storage | Get Keka Jobs | |
| Get Keka Jobs | httpRequest | Fetch jobs from Keka | Keka Token Manager | Normalize Jobs | |
| Normalize Jobs | code | Normalize job objects | Get Keka Jobs | Accumulate Jobs + Hiring Team | |
| Accumulate Jobs + Hiring Team | code | Enrich jobs with hiring team | Normalize Jobs | Loop Over Items | |
| Loop Over Items | splitInBatches | Iterate jobs in batches | Accumulate Jobs + Hiring Team; More Candidate Pages? (false) | Deduplicate Candidates; Init Candidate Pagination | |
| Deduplicate Candidates | code | Remove duplicate candidates | Loop Over Items | Resolve Job Recruiters | |
| Resolve Job Recruiters | code | Map job→recruiter(s) | Deduplicate Candidates | Prepare Candidate Rows | |
| Prepare Candidate Rows | code | Build sheet row payloads | Resolve Job Recruiters | Read Master Recruitment Sheet | |
| Read Master Recruitment Sheet | googleSheets | Read master tracker rows | Prepare Candidate Rows | Filter Sourced Candidates | |
| Filter Sourced Candidates | code | Keep candidates of interest | Read Master Recruitment Sheet | Compare With Sheet | |
| Compare With Sheet | code | Diff candidates vs sheet | Filter Sourced Candidates | Loop Over Items1; Group by Recruiter + job id | |
| Loop Over Items1 | splitInBatches | Iterate rows to append | Compare With Sheet; Send a message5 | Append row in sheet | |
| Append row in sheet | googleSheets | Append new candidate rows | Loop Over Items1 | Wait1 | |
| Wait1 | wait | Throttle before email | Append row in sheet | Send a message5 | |
| Send a message5 | gmail | Post-append email action | Wait1 | Loop Over Items1 | |
| Group by Recruiter + job id | code | Group candidates per recruiter/job | Compare With Sheet | Send a message to recruiter | |
| Send a message to recruiter | gmail | Notify recruiter about candidates | Group by Recruiter + job id | Mark Candidates Notified (Memory) | |
| Mark Candidates Notified (Memory) | code | Track notified candidates in memory | Send a message to recruiter | Prepare Rows For Sheet Update | |
| Prepare Rows For Sheet Update | code | Build update payload for sheet | Mark Candidates Notified (Memory) | Loop Over Items2 | |
| Loop Over Items2 | splitInBatches | Iterate sheet updates | Prepare Rows For Sheet Update; Update row in sheet5 | Wait | |
| Wait | wait | Throttle updates | Loop Over Items2 | Update row in sheet5 | |
| Update row in sheet5 | googleSheets | Update notification flags/status | Wait | Loop Over Items2 | |
| Init Candidate Pagination | code | Set pagination params per job | Loop Over Items; Next Candidate Page | Fetch job applications | |
| Fetch job applications | httpRequest | Fetch job applications page | Init Candidate Pagination | Normalize Candidates | |
| Normalize Candidates | code | Normalize candidate/application records | Fetch job applications | Accumulate Candidates | |
| Accumulate Candidates | code | Accumulate candidates across pages | Normalize Candidates | More Candidate Pages? | |
| More Candidate Pages? | if | Pagination gate | Accumulate Candidates | Next Candidate Page (true); Loop Over Items (false) | |
| Next Candidate Page | code | Increment cursor/page | More Candidate Pages? (true) | Init Candidate Pagination | |
| Jia Token Manager3 | executeWorkflow | Token setup chaining | Notify Recruiters1 | Jia Token Manager | |
| Jia Token Manager | executeWorkflow | Obtain Jia token via sub-workflow | Jia Token Manager3 | Read Master Recruitment Sheet1 | |
| Read Master Recruitment Sheet1 | googleSheets | Read candidates for resume/invite lane | Jia Token Manager | Loop Over Items3 | |
| Loop Over Items3 | splitInBatches | Iterate candidates for resume/invite | Read Master Recruitment Sheet1; Accumulate Invited Candidates | Group by Recruiter + job id1; Extract Role + YOE | |
| Extract Role + YOE | code | Parse role + experience | Loop Over Items3 | Fetch Candidate Resume | |
| Fetch Candidate Resume | httpRequest | Fetch resume metadata/url | Extract Role + YOE | Download Resume | |
| Download Resume | httpRequest | Download resume file | Fetch Candidate Resume | Jia Inteview Sent | |
| Jia Inteview Sent | httpRequest (disabled) | Trigger Jia interview invite | Download Resume | Dummy data for jia inteview | |
| Dummy data for jia inteview | set | Stand-in payload | Jia Inteview Sent | Accumulate Invited Candidates | |
| Accumulate Invited Candidates | code | Collect invited candidates for grouping | Dummy data for jia inteview | Loop Over Items3 | |
| Group by Recruiter + job id1 | code | Group invited candidates | Loop Over Items3 | Notify Recruiters | |
| Notify Recruiters | gmail | Email recruiters with invite summary | Group by Recruiter + job id1 | Prepare Sheet Updates from Memory | |
| Prepare Sheet Updates from Memory | code | Build sheet updates for invited | Notify Recruiters | Update row in sheet6 | |
| Update row in sheet6 | googleSheets | Apply invited/interview sent updates | Prepare Sheet Updates from Memory | — | |
| Schedule Trigger 2 | scheduleTrigger | Start AI screening lane | — | Read Master Recruitment For Feedback | |
| Read Master Recruitment For Feedback | googleSheets | Read candidates needing AI screening | Schedule Trigger 2 | Filter Candidates for AI Screening | |
| Filter Candidates for AI Screening | code | Select candidates to send to GPT | Read Master Recruitment For Feedback | Message a model; Merge (input 2) | |
| Message a model | openAi (langchain) | GPT‑4 screening | Filter Candidates for AI Screening | PARSE the OpenAI output | |
| PARSE the OpenAI output | code | Parse model response | Message a model | Merge | |
| Merge | merge | Combine original + AI output | PARSE the OpenAI output; Filter Candidates for AI Screening | Update Stage Status | |
| Update Stage Status | googleSheets | Update stage columns | Merge | Group by recruiter and job id | |
| Group by recruiter and job id | code | Group outcomes for recruiter emails | Update Stage Status | Notify Recruiters1; Filter rejected candidates | |
| Notify Recruiters1 | gmail | Notify recruiters of AI outcomes | Group by recruiter and job id | Keka Token Manager3 | |
| Filter rejected candidates | code | Identify rejected candidates | Group by recruiter and job id | Loop Over Items7 | |
| Loop Over Items7 | splitInBatches | Iterate rejection emails | Filter rejected candidates; Send Rejection Mails to candidates | Wait2 | |
| Wait2 | wait | Throttle | Loop Over Items7 | Send Rejection Mails to candidates | |
| Send Rejection Mails to candidates | gmail | Rejection emails | Wait2 | Loop Over Items7 | |
| Schedule Trigger 3 | scheduleTrigger | Start Jia report polling lane | — | Jia Token Manager1 | |
| Jia Token Manager1 | executeWorkflow | Obtain Jia token | Schedule Trigger 3 | Read Master Recruitment For Feedback1 | |
| Read Master Recruitment For Feedback1 | googleSheets | Read candidates for report polling | Jia Token Manager1 | If | |
| If | if | Gate processing | Read Master Recruitment For Feedback1 | Filter candidates (true path) | |
| Filter candidates | code | Select candidates | If | Fetch the AI interview Report | |
| Fetch the AI interview Report | httpRequest | Get report status/list | Filter candidates | Filter interview completed candidates | |
| Filter interview completed candidates | code | Keep completed interviews | Fetch the AI interview Report | Compare Candidates | |
| Compare Candidates | code | Diff vs existing sheet values | Filter interview completed candidates | Loop Over Items4 | |
| Loop Over Items4 | splitInBatches | Iterate report download/parse | Compare Candidates; Update row in sheet | Download Report; Merge1 (input 2) | |
| Download Report | httpRequest | Download report file | Loop Over Items4 | Extract from File | |
| Extract from File | extractFromFile | Extract text from report | Download Report | Parse Overall Score | |
| Parse Overall Score | code | Extract score fields | Extract from File | Merge1 | |
| Merge1 | merge | Merge score + candidate | Parse Overall Score; Loop Over Items4 | Final Status | |
| Final Status | code | Compute final status | Merge1 | Update row in sheet | |
| Update row in sheet | googleSheets | Write final status/score | Final Status | Loop Over Items4 | |
| Schedule Trigger 4 | scheduleTrigger | Start approval pending lane | — | Read Master Recruitment For Feedback2 | |
| Read Master Recruitment For Feedback2 | googleSheets | Read candidates for approval | Schedule Trigger 4 | If1 | |
| If1 | if | Gate processing | Read Master Recruitment For Feedback2 | Filter Candidates (true path) | |
| Filter Candidates | code | Select candidates | If1 | Group  by recruiter + job id | |
| Group  by recruiter + job id | code | Group for approvals | Filter Candidates | Notify Recruiters2 | |
| Notify Recruiters2 | gmail | Email recruiters re approval | Group  by recruiter + job id | Prepare Update Payload | |
| Prepare Update Payload | code | Build sheet update | Notify Recruiters2 | Update row As Approval Pending | |
| Update row As Approval Pending | googleSheets | Mark approval pending | Prepare Update Payload | — | |
| Schedule Trigger 6 | scheduleTrigger | Start tech interview nudge | — | Read Master Recruitment For Feedback3 | |
| Read Master Recruitment For Feedback3 | googleSheets | Read tech-interview-needed | Schedule Trigger 6 | If3 | |
| If3 | if | Gate | Read Master Recruitment For Feedback3 | Filter Candidates1 | |
| Filter Candidates1 | code | Select candidates | If3 | Group  by recruiter + job id1 | |
| Group  by recruiter + job id1 | code | Group tech interview | Filter Candidates1 | Notify Recruiters to Technical Interview | |
| Notify Recruiters to Technical Interview | gmail | Ask to schedule tech round | Group  by recruiter + job id1 | Prepare Update Payload1 | |
| Prepare Update Payload1 | code | Build sheet update | Notify Recruiters to Technical Interview | Update row As TR interview Pending | |
| Update row As TR interview Pending | googleSheets | Mark TR pending | Prepare Update Payload1 | — | |
| Schedule Trigger 7 | scheduleTrigger | Start HR discussion nudge | — | Read Master Recruitment For Feedback4 | |
| Read Master Recruitment For Feedback4 | googleSheets | Read HR-discussion-needed | Schedule Trigger 7 | If7 | |
| If7 | if | Gate | Read Master Recruitment For Feedback4 | Filter Candidates2 | |
| Filter Candidates2 | code | Select candidates | If7 | Group  by recruiter + job id2 | |
| Group  by recruiter + job id2 | code | Group HR discussion | Filter Candidates2 | Notify Recruiters to schedule HR Discussion Round | |
| Notify Recruiters to schedule HR Discussion Round | gmail | Ask to schedule HR round | Group  by recruiter + job id2 | Prepare Update Payload2 | |
| Prepare Update Payload2 | code | Build sheet update | Notify Recruiters to schedule HR Discussion Round | Update row As Hr interview Pending | |
| Update row As Hr interview Pending | googleSheets | Mark HR pending | Prepare Update Payload2 | — | |
| Schedule Trigger 8 | scheduleTrigger | Start leadership nudge | — | Read Master Recruitment For Feedback5 | |
| Read Master Recruitment For Feedback5 | googleSheets | Read leadership-needed | Schedule Trigger 8 | If8 | |
| If8 | if | Gate | Read Master Recruitment For Feedback5 | Filter Candidates3 | |
| Filter Candidates3 | code | Select candidates | If8 | Group  by recruiter + job id3 | |
| Group  by recruiter + job id3 | code | Group leadership connect | Filter Candidates3 | Notify Recruiters to schedule Leadership Connect Round | |
| Notify Recruiters to schedule Leadership Connect Round | gmail | Ask to schedule leadership round | Group  by recruiter + job id3 | Prepare Update Payload3 | |
| Prepare Update Payload3 | code | Build sheet update | Notify Recruiters to schedule Leadership Connect Round | Update row As LeaderShip round pending | |
| Update row As LeaderShip round pending | googleSheets | Mark leadership pending | Prepare Update Payload3 | — | |
| Schedule Trigger 9 | scheduleTrigger | Start interview scheduling emails | — | Read sheet with intetrview pending | |
| Read sheet with intetrview pending | googleSheets | Read interview-pending rows | Schedule Trigger 9 | Filter Candidates4 | |
| Filter Candidates4 | code | Pre-filter | Read sheet with intetrview pending | If9 | |
| If9 | if | Gate | Filter Candidates4 | Filter rows | |
| Filter rows | code | Normalize/filter rows | If9 | Loop Over Items6 | |
| Loop Over Items6 | splitInBatches | Iterate interview scheduling | Filter rows; Update Status1 | Resolve emails | |
| Resolve emails | code | Determine interviewer email | Loop Over Items6 | Send email to interviewer | |
| Send email to interviewer | gmail | Email interviewer | Resolve emails | Pass the data | |
| Pass the data | code | Pass/reshape payload | Send email to interviewer | Send email to Candidate | |
| Send email to Candidate | gmail | Email candidate | Pass the data | Prepare payload for sheets | |
| Prepare payload for sheets | code | Build sheet update | Send email to Candidate | Update Status1 | |
| Update Status1 | googleSheets | Update interview scheduling status | Prepare payload for sheets | Loop Over Items6 | |
| Schedule Trigger 10 | scheduleTrigger | Start HR interview pending lane | — | Keka Token Manager2 | |
| Keka Token Manager2 | executeWorkflow | Obtain Keka token | Schedule Trigger 10 | Read sheet with HR intetrview pending | |
| Read sheet with HR intetrview pending | googleSheets | Read HR pending rows | Keka Token Manager2 | Filter Candidates5 | |
| Filter Candidates5 | code | Filter relevant rows | Read sheet with HR intetrview pending | If4 | |
| If4 | if | Gate | Filter Candidates5 | Normalize Rows | |
| Normalize Rows | code | Normalize sheet rows | If4 | Loop Over Items5 | |
| Dummy | code | Trigger feedback fetch path | Loop Over Items5 | Fetch feedback report | |
| Fetch feedback report | httpRequest (disabled) | Fetch feedback report | Dummy | Normalize Feedback | |
| Normalize Feedback | code | Normalize feedback | Fetch feedback report | Merge2 | |
| Merge2 | merge | Merge rows + feedback | Normalize Feedback; Loop Over Items5 | Loop Over Items5 | |
| Loop Over Items5 | splitInBatches | Iterate HR decisions | Normalize Rows; Merge2 | Filter Final HR Decisions; Dummy | |
| Filter Final HR Decisions | code | Select final decisions | Loop Over Items5 | Group by Recruiter + Job ID | |
| Group by Recruiter + Job ID | code | Group decisions | Filter Final HR Decisions | Send a message; Filter rejected candidates1 | |
| Send a message | gmail | Notify on HR decision | Group by Recruiter + Job ID | Prepare Google Sheets Update Payload | |
| Prepare Google Sheets Update Payload | code | Build final status updates | Send a message | Update Final Status | |
| Update Final Status | googleSheets | Apply final status | Prepare Google Sheets Update Payload | — | |
| Filter rejected candidates1 | code | Identify rejected | Group by Recruiter + Job ID | Loop Over Items8 | |
| Loop Over Items8 | splitInBatches | Iterate rejection emails | Filter rejected candidates1; Send Rejection Mails to candidates1 | Wait3 | |
| Wait3 | wait | Throttle | Loop Over Items8 | Send Rejection Mails to candidates1 | |
| Send Rejection Mails to candidates1 | gmail | Rejection emails | Wait3 | Loop Over Items8 | |
| Schedule Trigger 11 | scheduleTrigger | Start hired notifications | — | Read sheet with Interview status as hired | |
| Read sheet with Interview status as hired | googleSheets | Read hired rows | Schedule Trigger 11 | If6 | |
| If6 | if | Gate | Read sheet with Interview status as hired | Pass | |
| Pass | code | Pass/normalize | If6 | Group by recruiter + job id | |
| Group by recruiter + job id | code | Group hired notifications | Pass | Send a message1 | |
| Send a message1 | gmail | Email hired notification | Group by recruiter + job id | Prepare Google Sheets Update Payload1 | |
| Prepare Google Sheets Update Payload1 | code | Build sheet update | Send a message1 | Update Status | |
| Update Status | googleSheets | Update hired status | Prepare Google Sheets Update Payload1 | — | |
| Schedule Trigger 5 | scheduleTrigger | Start Jia status sync | — | Jia Token Manager2 | |
| Jia Token Manager2 | executeWorkflow | Obtain Jia token | Schedule Trigger 5 | Read Master Recruitment For Feedback6 | |
| Read Master Recruitment For Feedback6 | googleSheets | Read rows requiring Jia update | Jia Token Manager2 | If2 | |
| If2 | if | Gate | Read Master Recruitment For Feedback6 | Filter Candidates6 | |
| Filter Candidates6 | code | Select candidates | If2 | Loop Over Items9 | |
| Loop Over Items9 | splitInBatches | Iterate Jia updates | Filter Candidates6; Update row in sheet1 | Update Jia status | |
| Update Jia status | httpRequest | Update candidate status in Jia | Loop Over Items9 | Update row in sheet1 | |
| Update row in sheet1 | googleSheets | Persist Jia status in sheet | Update Jia status | Loop Over Items9 | |
| Fetch Candidate Resume | httpRequest | (already listed above) | Extract Role + YOE | Download Resume | |
| Read Master Recruitment For Feedback (etc.) | googleSheets | (covered above) | — | — | |
| Sticky Note* | stickyNote | Visual comment | — | — | |

*(All Sticky Note nodes have empty content in this JSON.)*

---

## 4. Reproducing the Workflow from Scratch (step-by-step)

> Because the JSON omits nearly all node parameters (URLs, sheet IDs, prompts, conditions), the steps below describe **what must be configured** per node to reproduce the behavior.

### 4.1 Prerequisites / credentials
1. **Google Sheets OAuth2 credential** in n8n with access to the master recruitment spreadsheet.
2. **Gmail OAuth2 credential** (or Gmail service account setup if applicable) capable of sending emails.
3. **OpenAI credential** for the LangChain OpenAI node (ensure GPT‑4 model access).
4. **Keka API access** (base URL + endpoints + auth method). Typically Bearer token.
5. **Jia API access** (base URL + endpoints + auth method).

### 4.2 Create required sub-workflows (called via Execute Workflow)
Create these as separate workflows and ensure they return token(s) in a predictable JSON shape:
1. **“Keka Token Manager” sub-workflow**
   - Input: optional.
   - Output: `{ token: "..." }` (or set it in a field used by HTTP Request expressions).
   - Responsibilities: refresh token if expired, handle secrets, return token.
2. **“Jia Token Manager” sub-workflow**
   - Same concept for Jia.
3. Ensure the main workflow’s **Execute Workflow** nodes point to these workflows and map returned token to a field used in downstream HTTP nodes (e.g., `{{$json.token}}` or a pinned variable strategy).

### 4.3 Build Lane A — Keka ingestion → append to sheet → notify recruiter
1. Add **Schedule Trigger 1** (set cadence).
2. Add **Code** node “Init Run Storage” to initialize arrays/maps (e.g., `notifiedCandidateIds=[]`).
3. Add **Execute Workflow** “Keka Token Manager”.
4. Add **HTTP Request** “Get Keka Jobs”
   - Method: GET
   - URL: Keka jobs endpoint
   - Headers: `Authorization: Bearer {{$json.token}}` (or wherever your token is)
   - Turn on retries (matches `retryOnFail`).
5. Add **Code** “Normalize Jobs” to map API response into job items.
6. Add **Code** “Accumulate Jobs + Hiring Team” to enrich each job with recruiters/hiring team.
7. Add **SplitInBatches** “Loop Over Items” to iterate jobs (choose batch size).
8. Add **Code** “Init Candidate Pagination” to set initial page/cursor for the current job.
9. Add **HTTP Request** “Fetch job applications” to retrieve applications for job + page cursor.
10. Add **Code** “Normalize Candidates”.
11. Add **Code** “Accumulate Candidates”.
12. Add **IF** “More Candidate Pages?” checking a flag like `hasMore`/`nextPage`.
13. Add **Code** “Next Candidate Page” to advance cursor; connect back to **Init Candidate Pagination**.
14. From “Loop Over Items” also connect to **Code** “Deduplicate Candidates”.
15. Add **Code** “Resolve Job Recruiters”.
16. Add **Code** “Prepare Candidate Rows” to produce sheet-row shaped objects.
17. Add **Google Sheets** “Read Master Recruitment Sheet” (operation: Read/Get Many; configure Spreadsheet ID + Sheet name).
18. Add **Code** “Filter Sourced Candidates”.
19. Add **Code** “Compare With Sheet” that outputs:
   - New rows for append
   - Rows for recruiter notification / update
20. Add **SplitInBatches** “Loop Over Items1” to process “new rows”.
21. Add **Google Sheets** “Append row in sheet” (operation: Append; map columns).
22. Add **Wait** “Wait1” for throttling.
23. Add **Gmail** “Send a message5” (configure recipients/subject/body; likely internal notification).
24. For recruiter grouping: add **Code** “Group by Recruiter + job id”.
25. Add **Gmail** “Send a message to recruiter”.
26. Add **Code** “Mark Candidates Notified (Memory)”.
27. Add **Code** “Prepare Rows For Sheet Update”.
28. Add **SplitInBatches** “Loop Over Items2”.
29. Add **Wait** “Wait”.
30. Add **Google Sheets** “Update row in sheet5” (operation: Update; must have a stable row key strategy—e.g., store sheet row ID in a column).

### 4.4 Build Lane B — Resume + invite processing + recruiter notification
1. Add **Execute Workflow** “Jia Token Manager” (optionally preceded by “Keka Token Manager3” if you need chained auth setup).
2. Add **Google Sheets** “Read Master Recruitment Sheet1” (filter rows by stage, e.g., “Invite Pending”).
3. Add **SplitInBatches** “Loop Over Items3”.
4. Add **Code** “Extract Role + YOE”.
5. Add **HTTP Request** “Fetch Candidate Resume” then **HTTP Request** “Download Resume” (binary download enabled).
6. Add **HTTP Request** “Jia Inteview Sent” (keep **disabled** if you don’t want to call Jia yet).
7. Add **Set** “Dummy data for jia inteview” to emulate Jia response when above is disabled.
8. Add **Code** “Accumulate Invited Candidates”.
9. Add **Code** “Group by Recruiter + job id1”.
10. Add **Gmail** “Notify Recruiters”.
11. Add **Code** “Prepare Sheet Updates from Memory”.
12. Add **Google Sheets** “Update row in sheet6”.

### 4.5 Build Lane C — AI screening (OpenAI) + stage update + recruiter notify + rejections
1. Add **Schedule Trigger 2**.
2. Add **Google Sheets** “Read Master Recruitment For Feedback”.
3. Add **Code** “Filter Candidates for AI Screening”.
4. Add **OpenAI (LangChain)** “Message a model”
   - Choose GPT‑4 model
   - Prompt: include candidate profile, role, rubric; request strict JSON output.
5. Add **Code** “PARSE the OpenAI output” (JSON.parse with guardrails).
6. Add **Merge** node “Merge” to combine original candidate item + AI result (configure merge by index or key).
7. Add **Google Sheets** “Update Stage Status”.
8. Add **Code** “Group by recruiter and job id”.
9. Add **Gmail** “Notify Recruiters1”.
10. Add **Code** “Filter rejected candidates”.
11. Add **SplitInBatches** “Loop Over Items7” → **Wait2** → **Gmail** “Send Rejection Mails to candidates”.

### 4.6 Build Lane D — Jia report polling → score parse → sheet update
1. Add **Schedule Trigger 3**.
2. Add **Execute Workflow** “Jia Token Manager1”.
3. Add **Google Sheets** “Read Master Recruitment For Feedback1”.
4. Add **IF** “If” to ensure there are candidates requiring polling.
5. Add **Code** “Filter candidates”.
6. Add **HTTP Request** “Fetch the AI interview Report” (note `executeOnce=true` in JSON; enable “Execute Once” if you want one call per run).
7. Add **Code** “Filter interview completed candidates”.
8. Add **Code** “Compare Candidates”.
9. Add **SplitInBatches** “Loop Over Items4”.
10. Add **HTTP Request** “Download Report” (binary).
11. Add **Extract From File** “Extract from File” to text.
12. Add **Code** “Parse Overall Score”.
13. Add **Merge** “Merge1” with candidate data.
14. Add **Code** “Final Status”.
15. Add **Google Sheets** “Update row in sheet”.

### 4.7 Build Lanes E/F/G — Approval + Tech/HR/Leadership nudges
For each lane:
1. Add a **Schedule Trigger**.
2. Add **Google Sheets** read node for the relevant “Feedback” dataset.
3. Add an **IF** gate.
4. Add a **Code** filter selecting the stage.
5. Add a **Code** grouping by recruiter/job.
6. Add **Gmail** notify node with stage-specific email copy.
7. Add **Code** “Prepare Update PayloadX”.
8. Add **Google Sheets** update node to mark stage pending.

### 4.8 Build Lane H — Interview scheduling emails
1. Add **Schedule Trigger 9**.
2. Add **Google Sheets** “Read sheet with intetrview pending”.
3. Add **Code** filter + **IF9** gate.
4. Add **SplitInBatches** “Loop Over Items6”.
5. Add **Code** “Resolve emails”.
6. Add **Gmail** send to interviewer.
7. Add **Code** “Pass the data”.
8. Add **Gmail** send to candidate.
9. Add **Code** “Prepare payload for sheets”.
10. Add **Google Sheets** “Update Status1”.

### 4.9 Build Lane I — HR final decision notifications + update final status + rejection emails
1. Add **Schedule Trigger 10**.
2. Add **Execute Workflow** “Keka Token Manager2” (if needed).
3. Add **Google Sheets** “Read sheet with HR intetrview pending”.
4. Add **Code** filter + **IF4** gate.
5. Add **Code** “Normalize Rows”.
6. (Optional feedback fetch) Keep HTTP “Fetch feedback report” disabled or configure it; ensure **Merge2** mode handles missing second input.
7. Add **SplitInBatches** “Loop Over Items5”.
8. Add **Code** “Filter Final HR Decisions”.
9. Add **Code** “Group by Recruiter + Job ID”.
10. Add **Gmail** “Send a message”.
11. Add **Code** “Prepare Google Sheets Update Payload”.
12. Add **Google Sheets** “Update Final Status”.
13. Add rejection branch: **Filter rejected candidates1 → Loop Over Items8 → Wait3 → Send Rejection Mails to candidates1**.

### 4.10 Build Lane J — Hired notifications
1. Add **Schedule Trigger 11**.
2. Add **Google Sheets** “Read sheet with Interview status as hired”.
3. Add **IF6** gate.
4. Add **Code** “Pass”.
5. Add **Code** “Group by recruiter + job id”.
6. Add **Gmail** “Send a message1”.
7. Add **Code** “Prepare Google Sheets Update Payload1”.
8. Add **Google Sheets** “Update Status”.

### 4.11 Build Lane K — Jia status sync
1. Add **Schedule Trigger 5**.
2. Add **Execute Workflow** “Jia Token Manager2”.
3. Add **Google Sheets** “Read Master Recruitment For Feedback6”.
4. Add **IF2** gate.
5. Add **Code** “Filter Candidates6”.
6. Add **SplitInBatches** “Loop Over Items9”.
7. Add **HTTP Request** “Update Jia status”.
8. Add **Google Sheets** “Update row in sheet1”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided by user; indicates content is from an n8n workflow and compliant with policies. |
| Sticky notes exist but are empty in this workflow export. | All sticky note nodes have `content: ""`. |

--- 

If you can share (even partially) the missing node parameters (HTTP URLs, IF conditions, Code contents, sheet column mappings, email bodies, OpenAI prompt), I can refine this into an exact, reproducible specification including the precise expressions and data schemas used at each hop.