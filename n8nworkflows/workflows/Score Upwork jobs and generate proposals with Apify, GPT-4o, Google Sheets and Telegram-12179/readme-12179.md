Score Upwork jobs and generate proposals with Apify, GPT-4o, Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/score-upwork-jobs-and-generate-proposals-with-apify--gpt-4o--google-sheets-and-telegram-12179


# Score Upwork jobs and generate proposals with Apify, GPT-4o, Google Sheets and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow automatically scrapes new Upwork job posts via Apify, deduplicates them against a Google Sheet, uses OpenAI (GPT-4o) to score job fit (0–100), generates proposal drafts (GPT-4o-mini) for jobs above a threshold (default **60**), logs qualified leads to Google Sheets, and sends a run summary to Telegram. It also includes an error alert path.

**Target use cases:**
- Freelancers/agencies doing continuous lead sourcing on Upwork
- Automated triage of job posts against a niche/service offering
- Semi-automated proposal drafting to speed up outreach

### Logical Blocks
1.1 **Scheduling & Scraping (Apify)**  
1.2 **Deduplication (Google Sheets + Code)**  
1.3 **Normalization & AI Fit Scoring (GPT-4o)**  
1.4 **Filtering, Proposal Generation & Logging (GPT-4o-mini + Sheets)**  
1.5 **Metrics & Telegram Summary**  
1.6 **Global Error Handling (Error Trigger + Telegram)**

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Scraping (Apify)

**Overview:** Runs every 6 hours, triggers an Apify Actor run, then fetches dataset items (job listings) produced by that run.

**Nodes involved:**
- Schedule Trigger
- Run Apify Scraper
- Get Dataset Items

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point; time-based execution.
- **Configuration:** Every **6 hours** (`hoursInterval: 6`).
- **Inputs / outputs:** No input. Outputs to **Run Apify Scraper**.
- **Version notes:** Node typeVersion `1.2`.
- **Failure modes / edge cases:**
  - Workflow not active → never runs.
  - n8n instance downtime → missed schedules (depends on n8n scheduling behavior).

#### Node: Run Apify Scraper
- **Type / role:** `HTTP Request` — starts an Apify Actor run.
- **Configuration choices:**
  - **POST** to: `https://api.apify.com/v2/acts/{{ $env.APIFY_ACTOR_ID }}/runs?waitForFinish=300`
  - Waits up to **300s** for actor completion (`waitForFinish=300`).
  - Timeout set to **300000 ms** (300s).
  - Sends JSON body defining filters:
    - hourly min/max: 25–150
    - fixed min/max: 200–50000
    - categories: Web Development, AI Apps & Integration
    - limit: 100
  - **Auth:** `httpHeaderAuth` via generic credentials (expects Apify token in headers).
- **Key expressions / variables:**
  - `$env.APIFY_ACTOR_ID`
- **Inputs / outputs:** From Schedule Trigger → outputs actor run metadata to **Get Dataset Items**.
- **Version notes:** HTTP node typeVersion `4.2`.
- **Failure modes / edge cases:**
  - Missing/incorrect `APIFY_ACTOR_ID` env var.
  - Invalid Apify API token header → 401/403.
  - Actor run exceeds 300s → may return before dataset ready or fail depending on Apify response.
  - Rate limits / network timeouts.

#### Node: Get Dataset Items
- **Type / role:** `HTTP Request` — retrieves scraped jobs from Apify dataset.
- **Configuration choices:**
  - GET: `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items?clean=true`
  - `clean=true` returns simplified dataset items.
  - Auth: same `httpHeaderAuth` generic credential.
- **Key expressions / variables:**
  - `$json.data.defaultDatasetId` comes from **Run Apify Scraper** response.
- **Inputs / outputs:** From Run Apify Scraper → outputs list of job items to **Read Existing Job IDs**.
- **Failure modes / edge cases:**
  - Actor response does not include `defaultDatasetId` (actor misconfigured) → expression fails.
  - Dataset empty → downstream handles “no new jobs” path.
  - Auth/rate-limit errors.

---

### 2.2 Deduplication (Google Sheets + Code)

**Overview:** Loads existing job IDs from a Google Sheet and filters out jobs already processed.

**Nodes involved:**
- Read Existing Job IDs
- Filter Duplicates
- Has New Jobs?
- No New Jobs

#### Node: Read Existing Job IDs
- **Type / role:** `Google Sheets` — reads rows to build a set of existing Job IDs.
- **Configuration choices:**
  - Document ID from env: `{{$env.GOOGLE_SHEETS_DOC_ID}}`
  - Sheet/tab name: **“Upwork Jobs”**
  - `alwaysOutputData: true` (ensures node outputs even if no rows are returned).
- **Key expressions / variables:** `$env.GOOGLE_SHEETS_DOC_ID`
- **Inputs / outputs:** From Get Dataset Items → outputs rows to **Filter Duplicates**.
- **Version notes:** typeVersion `4.5`.
- **Failure modes / edge cases:**
  - Missing/incorrect Google credentials → auth failure.
  - Wrong sheet name → “sheet not found”.
  - Column header mismatch: the code expects a column named **`Job ID`**.

#### Node: Filter Duplicates
- **Type / role:** `Code` — compares scraped items against existing “Job ID” values.
- **Configuration choices (logic):**
  - `scraped = $('Get Dataset Items').all()`
  - `existing = $('Read Existing Job IDs').all()`
  - Builds `ids` as a Set of `r.json['Job ID']` values.
  - Filters scraped items where `j.json.uid` is not in `ids`.
  - If none remain, returns a single item `{ _noNewJobs: true }`.
- **Key variables:** `uid` from Apify dataset items; `Job ID` from Sheets rows.
- **Inputs / outputs:** Receives current execution context; outputs to **Has New Jobs?**.
- **Version notes:** Code node typeVersion `2`.
- **Failure modes / edge cases:**
  - If Apify items don’t contain `uid`, dedupe may incorrectly pass/fail.
  - If Sheet column contains extra whitespace or different header, `Job ID` reads undefined → duplicates won’t be detected.
  - Large sheets could slow execution (Sheets read + in-memory set).

#### Node: Has New Jobs?
- **Type / role:** `IF` — checks if the current item represents a real job (has `uid`) vs the `_noNewJobs` sentinel.
- **Configuration choices:**
  - Condition: `{{$json.uid}}` **not empty**
- **Routing:**
  - **True** → Normalize Fields
  - **False** → No New Jobs
- **Version notes:** typeVersion `2`.
- **Failure modes / edge cases:**
  - If upstream jobs use a different id field, condition fails and everything goes to “No New Jobs”.

#### Node: No New Jobs
- **Type / role:** `NoOp` — terminates the “nothing to do” path.
- **Inputs / outputs:** From IF false branch; no output connections.
- **Failure modes / edge cases:** None (placeholder node).

---

### 2.3 Normalization & AI Fit Scoring (GPT-4o)

**Overview:** Normalizes fields into a consistent schema, sends them to GPT-4o for scoring, then parses the model output into structured fields.

**Nodes involved:**
- Normalize Fields
- AI Scoring
- Parse AI Score
- Filter Score >= 60

#### Node: Normalize Fields
- **Type / role:** `Code` — converts Apify job objects into a consistent record used for scoring and downstream logging.
- **Configuration choices (output schema):**
  - `jobId: j.uid`
  - `title, description, url, postedAt`
  - `skills`: comma-joined
  - `budget`: derived as:
    - hourly range `$min-max/hr` if present
    - else fixed `$fixedBudget`
    - else `N/A`
  - `clientVerified`: Yes/No based on `j.client.paymentVerified`
  - `clientSpent`: `j.client.totalSpent || 0`
- **Inputs / outputs:** From IF true branch → outputs items to **AI Scoring**.
- **Failure modes / edge cases:**
  - Missing nested fields in Apify payload are guarded with optional chaining, but budget formats may vary (some jobs could appear as `N/A`).

#### Node: AI Scoring
- **Type / role:** `OpenAI (LangChain)` — calls OpenAI model to evaluate the job.
- **Configuration choices:**
  - Model: **`gpt-4o`**
  - No explicit prompt is shown in JSON (this is important): the node as provided will run with whatever prompt/messages are configured in the node UI; the JSON here only shows model and response settings.
  - `responses.values: [{}]` indicates a response is expected but does not document the schema.
- **Inputs / outputs:** From Normalize Fields → outputs LLM response to **Parse AI Score**.
- **Version notes:** `@n8n/n8n-nodes-langchain.openAi` typeVersion `2` (requires LangChain-enabled OpenAI node in n8n).
- **Failure modes / edge cases:**
  - Missing OpenAI credentials / model access (gpt-4o).
  - Prompt not configured → output may be unusable or generic.
  - Non-JSON output breaks parsing downstream.
  - Token limits if descriptions are long.

#### Node: Parse AI Score
- **Type / role:** `Code` — parses model output JSON and merges with original normalized fields.
- **Configuration choices (logic):**
  - `orig = $('Normalize Fields').all()` and aligns by index `i`.
  - Attempts to parse JSON from `item.json.message?.content`
  - Strips Markdown code fences ```json ... ```
  - Outputs merged object:
    - `score` default 0
    - `decision` default "SKIP"
    - `reasoning` default ""
- **Inputs / outputs:** From AI Scoring → outputs to **Filter Score >= 60**.
- **Failure modes / edge cases:**
  - If OpenAI returns fewer/more items or order changes, index-based merge can mismatch jobs (rare but possible depending on node behavior).
  - Invalid JSON → caught; defaults applied → job likely filtered out due to score 0.

#### Node: Filter Score >= 60
- **Type / role:** `Filter` — keeps only jobs with score ≥ 60.
- **Configuration choices:**
  - Strict number comparison: `{{$json.score}} >= 60`
- **Inputs / outputs:** From Parse AI Score → outputs to **Loop Over Items**.
- **Failure modes / edge cases:**
  - Score returned as string → strict validation may fail; ensure scoring prompt returns numeric `score`.

---

### 2.4 Filtering, Proposal Generation & Logging (GPT-4o-mini + Sheets)

**Overview:** Iterates through qualified jobs, generates a tailored proposal draft, then appends the result to Google Sheets.

**Nodes involved:**
- Loop Over Items
- Generate Proposal
- Log to Google Sheet
- Loop Complete

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — processes items in a loop (batch size defaults if not set in parameters).
- **Configuration choices:** No batch size explicitly configured (n8n default applies).
- **Routing:**
  - Output 0 → Loop Complete (when done)
  - Output 1 → Generate Proposal (per batch/item)
- **Inputs / outputs:** From Filter Score >= 60 → to Generate Proposal and Loop Complete.
- **Version notes:** typeVersion `3`.
- **Failure modes / edge cases:**
  - If many items, proposal generation can be slow/costly; batch sizing may need tuning.

#### Node: Generate Proposal
- **Type / role:** `OpenAI (LangChain)` — drafts a proposal for each high-scoring job.
- **Configuration choices:**
  - Model: **`gpt-4o-mini`** (cost-saving per sticky note)
  - As with scoring, the actual prompt/messages are not visible in the provided JSON; it must be configured in node UI.
- **Inputs / outputs:** From SplitInBatches (per item) → outputs to Log to Google Sheet.
- **Failure modes / edge cases:**
  - Prompt not configured → proposal may be low quality.
  - Token limits for long job descriptions.
  - Missing personalization variables if prompt expects fields not present.

#### Node: Log to Google Sheet
- **Type / role:** `Google Sheets` — appends the enriched job + proposal output to the sheet.
- **Configuration choices:**
  - Operation: **append**
  - Mapping: **auto-map input data**
  - Sheet name: “Upwork Jobs”
  - Document ID: `{{$env.GOOGLE_SHEETS_DOC_ID}}`
- **Inputs / outputs:** From Generate Proposal → loops back to **Loop Over Items** (to fetch next batch/item).
- **Failure modes / edge cases:**
  - If auto-mapping doesn’t match existing headers, columns may be added/misaligned depending on Sheets node behavior.
  - Concurrent runs could append duplicates if dedupe relies solely on prior rows and runs overlap.
  - Sheet permissions/quota errors.

#### Node: Loop Complete
- **Type / role:** `NoOp` — marks end of looping branch before metrics.
- **Inputs / outputs:** From SplitInBatches “done” output → to Compute Metrics.

---

### 2.5 Metrics & Telegram Summary

**Overview:** Computes how many jobs were scraped and how many passed the score threshold, then sends a Telegram summary.

**Nodes involved:**
- Compute Metrics
- Send Summary

#### Node: Compute Metrics
- **Type / role:** `Code` — derives summary stats from earlier nodes’ outputs.
- **Configuration choices:**
  - `scraped = $('Get Dataset Items').all().length` (guarded by try/catch)
  - `passed = $('Parse AI Score').all().filter(score >= 60).length` (guarded)
  - Outputs `{ timestamp, scraped, passed }`
- **Inputs / outputs:** From Loop Complete → to Send Summary.
- **Failure modes / edge cases:**
  - If earlier nodes didn’t run (e.g., no new jobs path), references may fail; try/catch prevents hard crash but may report 0.

#### Node: Send Summary
- **Type / role:** `Telegram` — sends a run summary to a chat.
- **Configuration choices:**
  - Chat ID: `{{$env.TELEGRAM_CHAT_ID}}`
  - Text template:
    - “✅ Upwork Scraper Done”
    - Scraped and Passed counts
- **Inputs / outputs:** From Compute Metrics; terminal node.
- **Failure modes / edge cases:**
  - Invalid Telegram credentials/bot token.
  - Wrong chat ID or bot not added to chat.
  - Telegram API downtime/rate limits.

---

### 2.6 Global Error Handling (Error Trigger + Telegram)

**Overview:** If the workflow errors, it triggers a separate execution path that sends an alert to Telegram.

**Nodes involved:**
- Error Trigger
- Send Error Alert

#### Node: Error Trigger
- **Type / role:** `Error Trigger` — starts when another workflow execution fails (within the same workflow context).
- **Configuration choices:** Default.
- **Inputs / outputs:** Emits error details to **Send Error Alert**.
- **Failure modes / edge cases:**
  - If Telegram credentials are also broken, you won’t get alerts.
  - Doesn’t automatically log errors to a datastore—only sends a message.

#### Node: Send Error Alert
- **Type / role:** `Telegram` — sends failure notification.
- **Configuration choices:**
  - Chat ID: `{{$env.TELEGRAM_CHAT_ID}}`
  - Text: `🚨 Error: {{ $json.error?.message || 'Unknown' }}`
- **Failure modes / edge cases:** Same Telegram limitations as summary.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / canvas annotation | — | — | ## Upwork Job Scraper with AI Scoring … (includes setup steps and pro tip) |
| Sticky Note1 | Sticky Note | Documentation / scraping block note | — | — | ## Scrape Jobs\nSchedule trigger kicks off Apify to fetch latest Upwork listings.\nBudget ranges (hourly: $25-150, fixed: $200-50k) are customizable. |
| Sticky Note2 | Sticky Note | Documentation / dedup block note | — | — | ## Deduplicate\nCross-references Google Sheet to skip already-processed jobs |
| Sticky Note3 | Sticky Note | Documentation / scoring block note | — | — | ## Perform AI Scoring\nGPT-4 evaluates fit, client quality, and budget. Filters jobs scoring 60+. |
| Sticky Note4 | Sticky Note | Documentation / proposal block note | — | — | ## Generate Proposals\nDrafts a custom proposal based on job description. Review the prompt before first use. |
| Sticky Note5 | Sticky Note | Documentation / log+notify block note | — | — | ## Log & Notify\nSaves results to Google Sheet and sends Telegram summary |
| Sticky Note6 | Sticky Note | Documentation / error handling note | — | — | ## Handle Errors\nCatches failures, logs errors, and sends alert to prevent missed opportunities |
| Schedule Trigger | Schedule Trigger | Starts workflow every 6 hours | — | Run Apify Scraper | ## Scrape Jobs… |
| Run Apify Scraper | HTTP Request | Launch Apify Actor run | Schedule Trigger | Get Dataset Items | ## Scrape Jobs… |
| Get Dataset Items | HTTP Request | Fetch Apify dataset items (jobs) | Run Apify Scraper | Read Existing Job IDs | ## Scrape Jobs… |
| Read Existing Job IDs | Google Sheets | Load existing Job IDs for dedupe | Get Dataset Items | Filter Duplicates | ## Deduplicate… |
| Filter Duplicates | Code | Remove already-seen jobs | Read Existing Job IDs (and Get Dataset Items via reference) | Has New Jobs? | ## Deduplicate… |
| Has New Jobs? | IF | Decide: proceed or stop | Filter Duplicates | Normalize Fields (true), No New Jobs (false) | ## Deduplicate… |
| No New Jobs | NoOp | Ends run when nothing new | Has New Jobs? (false) | — | ## Deduplicate… |
| Normalize Fields | Code | Standardize fields for LLM + logging | Has New Jobs? (true) | AI Scoring | ## Perform AI Scoring… |
| AI Scoring | OpenAI (LangChain) | Score job fit using GPT-4o | Normalize Fields | Parse AI Score | ## Perform AI Scoring… |
| Parse AI Score | Code | Parse JSON score output & merge | AI Scoring | Filter Score >= 60 | ## Perform AI Scoring… |
| Filter Score >= 60 | Filter | Keep only qualified jobs | Parse AI Score | Loop Over Items | ## Perform AI Scoring… |
| Loop Over Items | Split In Batches | Iterate qualified jobs | Filter Score >= 60 | Generate Proposal (loop), Loop Complete (done) | ## Generate Proposals… |
| Generate Proposal | OpenAI (LangChain) | Draft proposal with GPT-4o-mini | Loop Over Items | Log to Google Sheet | ## Generate Proposals… |
| Log to Google Sheet | Google Sheets | Append lead/proposal to sheet | Generate Proposal | Loop Over Items | ## Log & Notify… |
| Loop Complete | NoOp | Transition to metrics after loop | Loop Over Items (done) | Compute Metrics | ## Log & Notify… |
| Compute Metrics | Code | Compute scraped/passed counts | Loop Complete | Send Summary | ## Log & Notify… |
| Send Summary | Telegram | Send Telegram run summary | Compute Metrics | — | ## Log & Notify… |
| Error Trigger | Error Trigger | Start error path on failure | — | Send Error Alert | ## Handle Errors… |
| Send Error Alert | Telegram | Send Telegram error alert | Error Trigger | — | ## Handle Errors… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create required environment variables in n8n**
   - `GOOGLE_SHEETS_DOC_ID` = your spreadsheet ID
   - `APIFY_ACTOR_ID` = Apify actor id (e.g., `username~actorname` or actor identifier)
   - `TELEGRAM_CHAT_ID` = numeric chat id (user/group/channel where bot can post)

2) **Prepare Google Sheets**
   - Create a spreadsheet.
   - In the first tab (or a tab named exactly **“Upwork Jobs”**), add a header column named exactly: **`Job ID`**.
   - Add any other headers you want to log (e.g., Title, URL, Score, Proposal, etc.). Auto-mapping will attempt to match.

3) **Create credentials**
   - **Apify (HTTP Header Auth):** create an n8n “HTTP Header Auth” credential that adds `Authorization: Bearer <APIFY_TOKEN>` (or the header Apify expects for your setup).
   - **Google Sheets OAuth2:** connect a Google account with access to the sheet.
   - **OpenAI:** configure OpenAI credentials with access to **gpt-4o** and **gpt-4o-mini**.
   - **Telegram:** connect Telegram bot token credential.

4) **Build the main flow nodes (left to right)**

5) Add **Schedule Trigger**
   - Set interval: every **6 hours**.
   - Connect to **Run Apify Scraper**.

6) Add **HTTP Request** node named **Run Apify Scraper**
   - Method: **POST**
   - URL: `https://api.apify.com/v2/acts/{{$env.APIFY_ACTOR_ID}}/runs?waitForFinish=300`
   - Timeout: 300000 ms
   - Body: JSON, e.g.
     - hourly min/max: 25 / 150
     - fixed min/max: 200 / 50000
     - jobCategories: `["Web Development","AI Apps & Integration"]`
     - limit: 100
   - Auth: select your Apify header auth credential.
   - Connect to **Get Dataset Items**.

7) Add **HTTP Request** node named **Get Dataset Items**
   - Method: **GET**
   - URL: `https://api.apify.com/v2/datasets/{{$json.data.defaultDatasetId}}/items?clean=true`
   - Auth: same Apify credential.
   - Connect to **Read Existing Job IDs**.

8) Add **Google Sheets** node named **Read Existing Job IDs**
   - Document ID: `{{$env.GOOGLE_SHEETS_DOC_ID}}`
   - Sheet name: **Upwork Jobs**
   - Enable “Always Output Data” (so the workflow doesn’t break on empty sheet)
   - Connect to **Filter Duplicates**.

9) Add **Code** node named **Filter Duplicates**
   - Implement logic:
     - Load all scraped items from “Get Dataset Items”
     - Load all sheet rows from “Read Existing Job IDs”
     - Build a Set of existing `Job ID`
     - Keep only scraped jobs whose `uid` is not in the set
     - If none, output `{_noNewJobs:true}`
   - Connect to **Has New Jobs?**

10) Add **IF** node named **Has New Jobs?**
   - Condition: string `{{$json.uid}}` is **not empty**
   - True → **Normalize Fields**
   - False → **No New Jobs**

11) Add **NoOp** node named **No New Jobs** (ends this branch)

12) Add **Code** node named **Normalize Fields**
   - Map Apify output fields to:
     - `jobId, title, description, budget, url, skills, postedAt, clientVerified, clientSpent`
   - Connect to **AI Scoring**.

13) Add **OpenAI (LangChain)** node named **AI Scoring**
   - Model: **gpt-4o**
   - Configure the node prompt/messages in the UI so the model returns **valid JSON**, e.g.:
     - `{ "score": 0-100, "decision": "PASS|SKIP", "reasoning": "..." }`
   - Connect to **Parse AI Score**.

14) Add **Code** node named **Parse AI Score**
   - Parse `message.content` as JSON (strip ```json fences).
   - Merge with original normalized job fields.
   - Connect to **Filter Score >= 60**.

15) Add **Filter** node named **Filter Score >= 60**
   - Condition: number `{{$json.score}}` **gte** `60`
   - Connect to **Loop Over Items**.

16) Add **Split In Batches** node named **Loop Over Items**
   - Keep defaults or set a batch size (commonly 1–10).
   - Connect:
     - “Loop” output → **Generate Proposal**
     - “Done” output → **Loop Complete**

17) Add **OpenAI (LangChain)** node named **Generate Proposal**
   - Model: **gpt-4o-mini**
   - Configure prompt/messages to produce a proposal draft (ideally structured fields like `proposalText`).
   - Connect to **Log to Google Sheet**.

18) Add **Google Sheets** node named **Log to Google Sheet**
   - Operation: **Append**
   - Document ID: `{{$env.GOOGLE_SHEETS_DOC_ID}}`
   - Sheet name: **Upwork Jobs**
   - Columns mapping: **Auto-map input data**
   - Connect back to **Loop Over Items** (to continue batches).

19) Add **NoOp** node named **Loop Complete**
   - Connect to **Compute Metrics**.

20) Add **Code** node named **Compute Metrics**
   - Compute:
     - `scraped` = count of items from Get Dataset Items
     - `passed` = count of parsed items where score ≥ 60
   - Connect to **Send Summary**.

21) Add **Telegram** node named **Send Summary**
   - Chat ID: `{{$env.TELEGRAM_CHAT_ID}}`
   - Text: include scraped/passed counts.

22) **Add error handling**
   - Add **Error Trigger** node.
   - Connect to **Telegram** node named **Send Error Alert**
     - Chat ID: `{{$env.TELEGRAM_CHAT_ID}}`
     - Text: `Error: {{$json.error?.message || 'Unknown'}}`

23) **Activate workflow**
   - Ensure credentials + env vars exist.
   - Run once manually to confirm:
     - Apify dataset returns `uid`
     - Sheet contains “Job ID” column
     - LLM scoring returns valid JSON

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Create a new spreadsheet with a column header exactly named `Job ID` in the first tab.” | Sticky Note (Setup steps) |
| “In n8n, set `GOOGLE_SHEETS_DOC_ID`, `APIFY_ACTOR_ID`, and `TELEGRAM_CHAT_ID`.” | Sticky Note (Setup steps) |
| “The Generate Proposal node uses GPT-4o-mini to save costs. Update the system prompt with your portfolio links before the first run.” | Sticky Note (Pro tip) |
| “Budget ranges (hourly: $25-150, fixed: $200-50k) are customizable.” | Sticky Note (Scrape Jobs) |
| “GPT-4 evaluates fit… Filters jobs scoring 60+.” | Sticky Note (AI scoring) |
| “Catches failures… sends alert to prevent missed opportunities.” | Sticky Note (Error handling) |