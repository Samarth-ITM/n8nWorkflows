Automate LinkedIn job search and applications with Claude AI and Google Sheets

https://n8nworkflows.xyz/workflows/automate-linkedin-job-search-and-applications-with-claude-ai-and-google-sheets-13584


# Automate LinkedIn job search and applications with Claude AI and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automate LinkedIn job search and applications with Claude AI and Google Sheets  
**Workflow name (internal):** LinkedIn Job Search Automation with Claude AI

**Purpose:**  
This workflow accepts a job-search request (keywords, location, filters) via a **Webhook**, queries LinkedIn for matching job listings, normalizes and deduplicates them against a **Google Sheet**, uses **Claude (Anthropic)** to score each job against a candidate profile, then either applies (Easy Apply) and sends a recruiter connection request, or skips low-score jobs. All outcomes are logged to Google Sheets and a JSON summary is returned to the webhook caller.

**Primary use cases:**
- Automated sourcing + prioritization of LinkedIn jobs using AI scoring
- Automated “Easy Apply” submissions (where available) and recruiter outreach
- Central tracking/audit trail in Google Sheets

### Logical blocks
1.1 **Trigger & Input Validation**  
Webhook intake → validation/normalization → build LinkedIn filters + candidate profile context.

1.2 **Fetch & Filter Jobs**  
LinkedIn job search → parse/normalize listings → check against Google Sheets to skip already-seen jobs.

1.3 **AI Scoring & Decision**  
Claude scoring agent → parse JSON score → threshold gate.

1.4 **Apply, Outreach & Log**  
Easy Apply request → connection invite → format result rows → append to Google Sheets → respond to webhook.

---

## 2. Block-by-Block Analysis

### 2.1 Block 1 — Trigger & Input Validation

**Overview:**  
Receives a POST request, validates mandatory fields, sets defaults, and builds a unified `searchContext` containing search parameters, candidate profile, LinkedIn filter codes, and request metadata.

**Nodes involved:**
- Receive Job Search Request
- Validate Input and Build Search Params

#### Node: Receive Job Search Request
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point
- **Key configuration:**
  - Method: **POST**
  - Path: `linkedin-job-search`
  - Response mode: **Respond with “Respond to Webhook” node**
- **Input/Output:**
  - No input (trigger)
  - Outputs the incoming request payload (commonly under `body`)
- **Failure/edge cases:**
  - Wrong HTTP method/path → no trigger
  - Large payloads or missing JSON body → downstream validation errors
- **Version notes:** TypeVersion 2 (modern webhook options)

#### Node: Validate Input and Build Search Params
- **Type / role:** `Code` — validation + normalization + context building
- **Key configuration choices (interpreted):**
  - Reads request body from: `const body = $input.item.json.body || $input.item.json;`
  - Enforces required fields: `keywords`, `location` (throws error if missing)
  - Sets defaults:
    - `experienceLevel`: `'mid-senior'`
    - `jobType`: `'full-time'`
    - `datePosted`: `'past-week'` (note: later hard-mapped to `f_TPR: r604800`)
    - `remote`: `false`
    - `scoreThreshold`: `70`
    - `maxApplications`: `10` (declared but **not enforced anywhere else**)
    - `sendConnectionRequest`: Boolean flag (**declared but not used later**)
    - `easyApplyOnly`: default true unless explicitly set to `false`
  - Builds `candidateProfile` (defaults to placeholders like `YOUR_NAME`)
  - Maps filters:
    - Experience level → LinkedIn code `f_E` (internship..executive → '1'..'6')
    - Job type → LinkedIn code `f_JT` (full-time etc → 'F','P','C',...)
    - Date posted hardcoded: `f_TPR: 'r604800'` (past 7 days)
    - Remote filter: `f_LF: '2'` if remote is true
    - Easy Apply: `f_EA: 'true'` if easyApplyOnly is true
  - Creates metadata: `requestId`, `startedAt`, `applicationsCount` (counter is not updated later)
- **Outputs:**
  - `json.searchParams`
  - `json.candidateProfile`
  - `json.linkedInFilters`
  - `json.metadata`
- **Connections:**
  - Outgoing to:
    - Fetch LinkedIn Job Listings
    - Fetch Individual Job Details
    - Parse and Normalize Job Listings
- **Failure/edge cases:**
  - Missing required fields → workflow error
  - Non-string keywords/location → `.trim()` may fail
  - `scoreThreshold` parsing: `parseInt(...) || 70` will convert NaN to 70 (ok)
- **Important design note:**  
  This node connects directly to **Parse and Normalize Job Listings**, but that parsing node actually reads data from **Fetch LinkedIn Job Listings** by name (`$('Fetch LinkedIn Job Listings')...`). If Parse runs before the HTTP request finishes (or if branching executes unexpectedly), parsing can fail.

---

### 2.2 Block 2 — Fetch & Filter Jobs

**Overview:**  
Queries LinkedIn for job listings, converts the response into a normalized per-job item structure, then checks Google Sheets to skip jobs already present in the tracking sheet.

**Nodes involved:**
- Fetch LinkedIn Job Listings
- Fetch Individual Job Details
- Parse and Normalize Job Listings
- Check Already Applied (Google Sheets)
- Skip Already Seen Jobs

#### Node: Fetch LinkedIn Job Listings
- **Type / role:** `HTTP Request` — LinkedIn job search call
- **Key configuration:**
  - URL: `https://api.linkedin.com/v2/jobSearch`
  - Query params:
    - `keywords={{ $json.searchParams.keywords }}`
    - `location={{ $json.searchParams.location }}`
    - `f_E={{ $json.linkedInFilters.f_E }}`
    - `f_JT={{ $json.linkedInFilters.f_JT }}`
    - `f_TPR={{ $json.linkedInFilters.f_TPR }}`
    - `count=25`, `start=0`
  - Headers:
    - `X-Restli-Protocol-Version: 2.0.0`
    - `Content-Type: application/json`
  - Timeout: 15s
  - Auth: **LinkedIn OAuth2 (predefined credential type)** (`linkedInOAuth2Api`)
  - `continueOnFail: true` (errors become output items with `error` structure rather than stopping)
- **Outputs:** Raw LinkedIn response (expected fields like `elements`)
- **Failure/edge cases:**
  - LinkedIn API permissions: many job endpoints are restricted; may return 401/403 even with OAuth2
  - Rate limiting (429), timeout, schema changes
  - If request fails and `continueOnFail` is enabled, downstream parsing may see an error object and throw “No valid job listings found...”

#### Node: Fetch Individual Job Details
- **Type / role:** `HTTP Request` — fetch a single job record
- **Key configuration:**
  - URL template: `https://api.linkedin.com/v2/jobs/{{ $json.jobId }}`
  - Header: `X-Restli-Protocol-Version: 2.0.0`
  - Timeout: 10s
  - Auth: LinkedIn OAuth2 credential
  - `continueOnFail: true`
- **Connections / usage reality:**
  - It is connected from **Validate Input and Build Search Params**, but it expects `jobId` which does not exist at that stage.
  - No downstream nodes consume its output.
- **Likely intended design:**  
  This node should probably run **per parsed job** (after parsing) to enrich descriptions, skills, recruiter, etc.
- **Failure/edge cases:**
  - As currently wired, it will call `/v2/jobs/undefined` (or similar) and fail on most executions.

#### Node: Parse and Normalize Job Listings
- **Type / role:** `Code` — transforms LinkedIn search response into normalized items
- **Key configuration choices:**
  - Reads:
    - `jobsResponse` from **Fetch LinkedIn Job Listings** (by node name): `$('Fetch LinkedIn Job Listings').first().json`
    - `searchContext` from **Validate Input and Build Search Params**
  - Attempts to locate listings in: `elements || jobs || data || []`
  - Produces one item per job: `{ json: { job: parsed, searchContext } }`
  - Normalizes fields: `jobId`, `title`, `companyName`, `companyId`, `location`, `description`, `descriptionSnippet`, `isEasyApply`, `jobUrl`, `postedAt`, `salary`, `skills`, etc.
  - If no jobs parsed → throws: “No valid job listings found...”
- **Connections:**
  - Outgoing to Check Already Applied (Google Sheets)
- **Failure/edge cases:**
  - LinkedIn response schema mismatch → `rawJobs` empty → error
  - `descriptionSnippet` uses `(job.description?.text || '').substring(0, 500)`; if description is in a different field, AI will score with little/no content
  - `continueOnFail` on the HTTP node can feed an error payload here; code does not explicitly detect that

#### Node: Check Already Applied (Google Sheets)
- **Type / role:** `Google Sheets` — lookup/deduplication step
- **Key configuration:**
  - Authentication: **Service Account** (`googleApi`)
  - **Document ID and Sheet Name are blank placeholders** (`value: "="`), so this node is not functional until configured.
  - Operation is not explicitly shown (likely “Read / Lookup” by default depending on node defaults), but the downstream IF node expects an array-like output with `.length`.
  - `continueOnFail: true`
- **Connections:**
  - Outgoing to Skip Already Seen Jobs
- **Failure/edge cases:**
  - Missing documentId/sheetName → node error (but may continue due to continueOnFail)
  - Permissions: service account must have access to the target spreadsheet
  - If operation returns a single object rather than an array, the IF length check may behave incorrectly

#### Node: Skip Already Seen Jobs
- **Type / role:** `IF` — gate to only process jobs not found in the sheet
- **Key configuration:**
  - Condition: `={{ $json.length || 0 }} equals 0`
  - Interpretation: if the Google Sheets lookup returned 0 rows, treat as “not seen” → proceed
- **Connections:**
  - True branch → Score Job with Claude AI
  - False branch → (none; job is effectively dropped)
- **Failure/edge cases:**
  - If Google Sheets node fails and outputs an error object, `$json.length` may be undefined → becomes 0 → **false negatives** (it may process jobs that should have been skipped)

---

### 2.3 Block 3 — AI Scoring & Decision

**Overview:**  
For each job that passes dedupe, the workflow asks Claude to return a strict JSON evaluation (score, pros/cons, messages). It then parses this JSON and checks if the score meets a threshold.

**Nodes involved:**
- Score Job with Claude AI
- Claude AI Model
- Parse AI Score and Decide
- Check Score Threshold

#### Node: Score Job with Claude AI
- **Type / role:** LangChain **Agent** (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model
- **Key configuration:**
  - Prompt includes:
    - Candidate profile from `$json.searchContext.candidateProfile...`
    - Job data from `$json.job...` (notably `descriptionSnippet`)
    - Requires **JSON only** output with fields:
      - `totalScore`, `breakdown`, `matchedSkills`, `missingSkills`, `pros`, `cons`, `shouldApply`, `coverLetterHook`, `connectionMessage`
  - System message enforces “JSON only, no markdown”
  - Prompt type: `define`
- **Model connection:**
  - Uses **Claude AI Model** via `ai_languageModel` connection
- **Failure/edge cases:**
  - Model may still output non-JSON or trailing text → parse step will fail
  - Token limits: long descriptions could truncate (here snippet is 500 chars, mitigating this)
  - If job.skills is empty, prompt becomes less effective

#### Node: Claude AI Model
- **Type / role:** Anthropic chat model node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`)
- **Key configuration:**
  - Model: `claude-sonnet-4-20250514`
  - Temperature: `0.3` (more deterministic)
  - Credential: Anthropic API key
- **Failure/edge cases:**
  - Invalid API key / billing / rate limits
  - Model name availability (must exist in your Anthropic account / region)

#### Node: Parse AI Score and Decide
- **Type / role:** `Code` — robust-ish JSON extraction + decision prep
- **Key configuration choices:**
  - Tries multiple fields for AI output: `response || output || text`, also handles `content[0].text` (Anthropic format)
  - Strips markdown fences if present
  - `JSON.parse()`; throws with the raw response on failure
  - Retrieves upstream job data using:
    - `const jobData = $('Skip Already Seen Jobs').item.json;`
  - Computes:
    - `scoreThreshold` from `jobData.searchContext.searchParams.scoreThreshold` default 70
    - `meetsThreshold = totalScore >= scoreThreshold`
- **Connections:**
  - Outgoing to Check Score Threshold
- **Failure/edge cases:**
  - If upstream node names change, `$('Skip Already Seen Jobs')...` will break
  - If Agent output structure changes, extraction may fail
  - If Claude returns numbers as strings, comparisons still work if `totalScore` is numeric-like, but safest would be explicit coercion

#### Node: Check Score Threshold
- **Type / role:** `IF` — routes to apply vs skip
- **Key configuration:**
  - Condition: `={{ $json.meetsThreshold }}` is true
- **Connections:**
  - True → Submit Easy Apply Application
  - False → Format Skipped Job Result
- **Failure/edge cases:**
  - If `meetsThreshold` missing/undefined, strict boolean check may evaluate false (skips)

---

### 2.4 Block 4 — Apply, Outreach & Log

**Overview:**  
For high-scoring jobs, the workflow attempts an Easy Apply submission and then sends a connection invitation. It formats a final result object (applied or skipped) and appends it to Google Sheets, then responds to the webhook.

**Nodes involved:**
- Submit Easy Apply Application
- Send Recruiter Connection Request
- Format Applied Job Result
- Format Skipped Job Result
- Log All Results to Google Sheets
- Send Summary Response

#### Node: Submit Easy Apply Application
- **Type / role:** `HTTP Request` — LinkedIn apply call
- **Key configuration:**
  - POST `https://api.linkedin.com/v2/jobs/{{ $json.job.jobId }}/apply`
  - JSON body includes:
    - `jobPostingUrn: urn:li:jobPosting:{{jobId}}`
    - `resumeText: {{ candidateProfile.summary }}`
    - `coverLetter: {{ aiScore.coverLetterHook }} ...`
    - `contactInfo.name`
  - Headers: Rest.li protocol + JSON content type
  - Timeout: 15s
  - Auth: LinkedIn OAuth2
  - `continueOnFail: true`
- **Connections:**
  - Outgoing to Send Recruiter Connection Request
- **Failure/edge cases:**
  - LinkedIn API for applying is typically restricted; many accounts/apps cannot submit applications via API
  - If the job is not Easy Apply, endpoint may reject
  - Body fields may not match required schema; could return 400
  - With continueOnFail, failures flow forward and are handled later via `applyResponse.error` checks

#### Node: Send Recruiter Connection Request
- **Type / role:** `HTTP Request` — LinkedIn invitations endpoint
- **Key configuration:**
  - POST `https://api.linkedin.com/v2/invitations`
  - Uses:
    - `profileId: {{ $json.job.companyId }}`
    - `message: {{ $json.aiScore.connectionMessage }}`
    - `trackingId: {{ $json.searchContext.metadata.requestId }}`
  - Timeout: 10s
  - Auth: LinkedIn OAuth2
  - `continueOnFail: true`
- **Critical correctness note:**  
  `profileId` is being set to **companyId**, not a recruiter/person profile ID. This is very likely incorrect for invitations APIs, which generally expect a **member profile identifier**, not a company URN/id.
- **Connections:**
  - Outgoing to Format Applied Job Result
- **Failure/edge cases:**
  - 400/403 due to invalid invitee type, permissions, or ID mismatch
  - LinkedIn anti-abuse limits for invitations

#### Node: Format Applied Job Result
- **Type / role:** `Code` — produce a structured log row for applied path
- **Key configuration choices:**
  - Pulls:
    - `jobData` from `$('Parse AI Score and Decide').item.json`
    - `applyResponse` from `$('Submit Easy Apply Application').item.json`
    - `connectionResponse` from `$('Send Recruiter Connection Request').item.json`
  - Computes success heuristics:
    - Application success: `!applyResponse.error && (applyResponse.status === 200 || applyResponse.id)`
    - Connection success: `!connectionResponse.error && (connectionResponse.id || connectionResponse.createdAt)`
  - Outputs fields: status `APPLIED`, job info, AI score, skills, pros/cons, flags, timestamps, requestId
- **Connections:**
  - Outgoing to Log All Results to Google Sheets
- **Failure/edge cases:**
  - If upstream nodes returned nonstandard structures, success heuristics may misclassify outcomes
  - Node-name dependencies make it brittle to renames

#### Node: Format Skipped Job Result
- **Type / role:** `Code` — produce log row for skipped low-score path
- **Key configuration choices:**
  - Outputs status `SKIPPED_LOW_SCORE`, reason string, scoreThreshold, timestamps, requestId
- **Connections:**
  - Outgoing to Log All Results to Google Sheets

#### Node: Log All Results to Google Sheets
- **Type / role:** `Google Sheets` — append log row
- **Key configuration:**
  - Operation: **Append**
  - Authentication: **Google Sheets OAuth2** (`googleSheetsOAuth2Api`)
  - **Document ID and Sheet Name are blank placeholders** (`value: "="`) → must be set
  - Columns mapping is empty (`mappingMode: defineBelow` but no schema)
  - `continueOnFail: true`
- **Connections:**
  - Outgoing to Send Summary Response
- **Failure/edge cases:**
  - Missing doc/sheet config → node error
  - Column mismatch: if sheet headers exist, append may still work but fields may not land in intended columns without mapping
  - OAuth token expiration/consent issues

#### Node: Send Summary Response
- **Type / role:** `Respond to Webhook` — final HTTP response
- **Key configuration:**
  - Content-Type: `application/json`
  - Respond with: JSON
  - Response body: `={{ JSON.stringify($json, null, 2) }}`
- **Connections:** terminal
- **Edge cases:**
  - Because the workflow processes multiple jobs, “the last item” reaching this node may be returned unless execution is consolidated. As built, it responds with whatever `$json` is at that point (often a single formatted result), not a full multi-job summary.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / overview |  |  | ## LinkedIn Job Search Automation with Claude AI; This AI-powered workflow automatically searches LinkedIn… Full audit log in Google Sheets |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block label |  |  | ## 1. Trigger & Input Validation |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block label |  |  | ## 2. Fetch & Filter Jobs |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block label |  |  | ## 3. AI Scoring & Decision |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block label |  |  | ## 4. Apply, Outreach & Log |
| Receive Job Search Request | n8n-nodes-base.webhook | Trigger: accept search request |  | Validate Input and Build Search Params | ## 1. Trigger & Input Validation |
| Validate Input and Build Search Params | n8n-nodes-base.code | Validate payload; build searchContext, filters, profile | Receive Job Search Request | Fetch LinkedIn Job Listings; Fetch Individual Job Details; Parse and Normalize Job Listings | ## 1. Trigger & Input Validation |
| Fetch LinkedIn Job Listings | n8n-nodes-base.httpRequest | Search LinkedIn jobs | Validate Input and Build Search Params | Parse and Normalize Job Listings | ## 2. Fetch & Filter Jobs |
| Fetch Individual Job Details | n8n-nodes-base.httpRequest | (Intended) enrich each job; currently miswired/unused | Validate Input and Build Search Params |  | ## 2. Fetch & Filter Jobs |
| Parse and Normalize Job Listings | n8n-nodes-base.code | Normalize listing objects into per-job items | Fetch LinkedIn Job Listings *(also directly connected from Validate Input)* | Check Already Applied (Google Sheets) | ## 2. Fetch & Filter Jobs |
| Check Already Applied (Google Sheets) | n8n-nodes-base.googleSheets | Lookup job in tracking sheet (dedupe) | Parse and Normalize Job Listings | Skip Already Seen Jobs | ## 2. Fetch & Filter Jobs |
| Skip Already Seen Jobs | n8n-nodes-base.if | Gate: only new/unseen jobs proceed | Check Already Applied (Google Sheets) | Score Job with Claude AI | ## 2. Fetch & Filter Jobs |
| Score Job with Claude AI | @n8n/n8n-nodes-langchain.agent | Claude-based scoring + message generation | Skip Already Seen Jobs | Parse AI Score and Decide | ## 3. AI Scoring & Decision |
| Claude AI Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic LLM provider |  | Score Job with Claude AI | ## 3. AI Scoring & Decision |
| Parse AI Score and Decide | n8n-nodes-base.code | Parse AI JSON; compute threshold decision | Score Job with Claude AI | Check Score Threshold | ## 3. AI Scoring & Decision |
| Check Score Threshold | n8n-nodes-base.if | Gate: apply vs skip | Parse AI Score and Decide | Submit Easy Apply Application; Format Skipped Job Result | ## 3. AI Scoring & Decision |
| Submit Easy Apply Application | n8n-nodes-base.httpRequest | Submit application via LinkedIn API | Check Score Threshold | Send Recruiter Connection Request | ## 4. Apply, Outreach & Log |
| Send Recruiter Connection Request | n8n-nodes-base.httpRequest | Send LinkedIn invitation with AI message | Submit Easy Apply Application | Format Applied Job Result | ## 4. Apply, Outreach & Log |
| Format Applied Job Result | n8n-nodes-base.code | Produce log row for applied path | Send Recruiter Connection Request | Log All Results to Google Sheets | ## 4. Apply, Outreach & Log |
| Format Skipped Job Result | n8n-nodes-base.code | Produce log row for skipped path | Check Score Threshold | Log All Results to Google Sheets | ## 4. Apply, Outreach & Log |
| Log All Results to Google Sheets | n8n-nodes-base.googleSheets | Append results to tracking sheet | Format Applied Job Result; Format Skipped Job Result | Send Summary Response | ## 4. Apply, Outreach & Log |
| Send Summary Response | n8n-nodes-base.respondToWebhook | Return JSON response to caller | Log All Results to Google Sheets |  | ## 4. Apply, Outreach & Log |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **LinkedIn Job Search Automation with Claude AI**
- Ensure workflow setting “Execution Order” is default (`v1`).

2) **Add Sticky Notes (optional but matching the original)**
- Add one general overview sticky note describing purpose and setup.
- Add four block label sticky notes:
  - “1. Trigger & Input Validation”
  - “2. Fetch & Filter Jobs”
  - “3. AI Scoring & Decision”
  - “4. Apply, Outreach & Log”

3) **Add trigger node: Webhook**
- Node: **Webhook** → name: *Receive Job Search Request*
- Configure:
  - HTTP Method: **POST**
  - Path: `linkedin-job-search`
  - Response: **Using “Respond to Webhook” node** (responseMode: responseNode)

4) **Add Code node: input validation + context builder**
- Node: **Code** → name: *Validate Input and Build Search Params*
- Mode: **Run Once for Each Item**
- Paste logic that:
  - Reads `body` from `incoming.body` or root
  - Validates required `keywords`, `location`
  - Builds:
    - `searchParams` with defaults (`scoreThreshold`, `easyApplyOnly`, etc.)
    - `candidateProfile` with your real information
    - `linkedInFilters` mapping (`f_E`, `f_JT`, `f_TPR`, optional `f_LF`, optional `f_EA`)
    - `metadata` containing `requestId`, timestamps
- Connect: **Webhook → Code**

5) **Add HTTP Request node: LinkedIn job search**
- Node: **HTTP Request** → name: *Fetch LinkedIn Job Listings*
- Configure:
  - URL: `https://api.linkedin.com/v2/jobSearch`
  - Authentication: **Predefined credential type → LinkedIn OAuth2**
  - Query parameters:
    - `keywords = {{$json.searchParams.keywords}}`
    - `location = {{$json.searchParams.location}}`
    - `f_E = {{$json.linkedInFilters.f_E}}`
    - `f_JT = {{$json.linkedInFilters.f_JT}}`
    - `f_TPR = {{$json.linkedInFilters.f_TPR}}`
    - `count = 25`
    - `start = 0`
  - Headers:
    - `X-Restli-Protocol-Version: 2.0.0`
    - `Content-Type: application/json`
  - Timeout: 15000ms
  - Continue On Fail: **true** (to match original)
- Connect: **Validate Input → Fetch LinkedIn Job Listings**

6) **(As in the original) Add HTTP Request node: individual job details**
- Node: **HTTP Request** → name: *Fetch Individual Job Details*
- Configure:
  - URL: `https://api.linkedin.com/v2/jobs/{{$json.jobId}}`
  - Auth: LinkedIn OAuth2
  - Header: `X-Restli-Protocol-Version: 2.0.0`
  - Timeout 10000ms
  - Continue On Fail: true
- Connect (to mirror the original, even if not functional):
  - **Validate Input → Fetch Individual Job Details**
- Note: to make it actually useful, you would instead connect it after parsing per-job items and set `jobId` from the parsed job object.

7) **Add Code node: parse/normalize listings**
- Node: **Code** → name: *Parse and Normalize Job Listings*
- Paste logic that:
  - Reads the raw response from **Fetch LinkedIn Job Listings**
  - Extracts `elements` (or fallback)
  - Emits one item per job with `json.job` + `json.searchContext`
  - Throws if none parsed
- Connect:
  - **Fetch LinkedIn Job Listings → Parse and Normalize Job Listings**
- (To fully match the original wiring) also connect:
  - **Validate Input → Parse and Normalize Job Listings**  
  This is not recommended, but it exists in the provided workflow.

8) **Add Google Sheets node: dedupe check**
- Node: **Google Sheets** → name: *Check Already Applied (Google Sheets)*
- Authentication: **Service Account**
- Configure:
  - Document ID: select your spreadsheet
  - Sheet name/tab: select the log tab (e.g., `Applications`)
  - Operation: choose a lookup/read method that returns matching rows for the current `jobId` (implementation depends on your sheet layout).
- Connect: **Parse and Normalize Job Listings → Check Already Applied**

9) **Add IF node: skip already seen**
- Node: **IF** → name: *Skip Already Seen Jobs*
- Condition (to match original intent):
  - If the Google Sheets output length equals 0 → proceed
  - Expression: `{{$json.length || 0}} == 0`
- Connect: **Check Already Applied → Skip Already Seen Jobs (true path onward)**

10) **Add Anthropic model node**
- Node: **Claude / Anthropic Chat Model** → name: *Claude AI Model*
- Model: `claude-sonnet-4-20250514`
- Temperature: `0.3`
- Credentials: set **Anthropic API** key.

11) **Add LangChain Agent node for scoring**
- Node: **AI Agent** → name: *Score Job with Claude AI*
- Provide the full scoring prompt that:
  - Injects candidate profile + job data
  - Demands JSON-only output
- Set the **system message** to enforce JSON-only.
- Connect:
  - **Skip Already Seen Jobs → Score Job with Claude AI**
  - Connect **Claude AI Model → Score Job with Claude AI** via the model input port.

12) **Add Code node: parse AI output + compute threshold**
- Node: **Code** → name: *Parse AI Score and Decide*
- Implement:
  - Extract text from agent response fields (`content[0].text` etc.)
  - Strip ``` fences if present
  - `JSON.parse`
  - Compute `meetsThreshold`
- Connect: **Score Job with Claude AI → Parse AI Score and Decide**

13) **Add IF node: score gate**
- Node: **IF** → name: *Check Score Threshold*
- Condition: `{{$json.meetsThreshold}} is true`
- Connect: **Parse AI Score and Decide → Check Score Threshold**

14) **Add HTTP Request node: Easy Apply**
- Node: **HTTP Request** → name: *Submit Easy Apply Application*
- Configure:
  - POST `https://api.linkedin.com/v2/jobs/{{$json.job.jobId}}/apply`
  - Auth: LinkedIn OAuth2
  - JSON body containing candidate summary and AI cover letter hook
  - Headers for Rest.li protocol
  - Continue on fail: true
- Connect: **Check Score Threshold (true) → Submit Easy Apply Application**

15) **Add HTTP Request node: invitation**
- Node: **HTTP Request** → name: *Send Recruiter Connection Request*
- Configure:
  - POST `https://api.linkedin.com/v2/invitations`
  - JSON body with invitee identifier, message, trackingId
  - Continue on fail: true
- Connect: **Submit Easy Apply Application → Send Recruiter Connection Request**

16) **Add Code nodes to format results**
- Node A: **Code** → *Format Applied Job Result*
  - Combine jobData + applyResponse + connectionResponse
  - Output a clean log object
- Node B: **Code** → *Format Skipped Job Result*
  - Output status `SKIPPED_LOW_SCORE`, reason, score, threshold
- Connect:
  - **Send Recruiter Connection Request → Format Applied Job Result**
  - **Check Score Threshold (false) → Format Skipped Job Result**

17) **Add Google Sheets node: append log**
- Node: **Google Sheets** → name: *Log All Results to Google Sheets*
- Authentication: **Google Sheets OAuth2**
- Operation: **Append**
- Configure:
  - Document ID and sheet/tab
  - Define column mappings that match your sheet headers (recommended), e.g.:
    - status, jobId, jobTitle, company, location, jobUrl, salary, aiScore, appliedAt, requestId, etc.
- Connect:
  - **Format Applied Job Result → Log All Results**
  - **Format Skipped Job Result → Log All Results**

18) **Add Respond to Webhook node**
- Node: **Respond to Webhook** → name: *Send Summary Response*
- Configure:
  - Respond with: JSON
  - Body: `{{ JSON.stringify($json, null, 2) }}`
  - Header Content-Type: application/json
- Connect: **Log All Results → Send Summary Response**

19) **Credentials to configure**
- **LinkedIn OAuth2** (`linkedInOAuth2Api`):
  - Must be created in n8n credentials and authorized.
  - Note: Many LinkedIn endpoints used here may require special partner access.
- **Anthropic API**:
  - Provide API key with access to the specified model.
- **Google Sheets Service Account** (for dedupe node):
  - Share the spreadsheet with the service account email.
- **Google Sheets OAuth2** (for append node):
  - Authorize account and ensure spreadsheet access.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow description: searches LinkedIn, deduplicates, scores with Claude, applies/connects, logs to Google Sheets | From the large overview sticky note |
| Sample trigger payload includes keywords, location, experienceLevel, jobType, scoreThreshold | From the overview sticky note |
| Setup mentions configuring LinkedIn OAuth2, Anthropic API, Google Sheets; and updating profile text in “Build Search Context” | Note: the actual node is named **“Validate Input and Build Search Params”**, not “Build Search Context” |
| Some nodes contain placeholders for Google Sheet documentId/sheetName (`=`) and require manual configuration | Applies to both Google Sheets nodes |
| LinkedIn endpoints shown (`/v2/jobSearch`, `/v2/jobs/{id}/apply`, `/v2/invitations`) may not work for standard developer apps due to permission restrictions | Integration risk to plan for |

