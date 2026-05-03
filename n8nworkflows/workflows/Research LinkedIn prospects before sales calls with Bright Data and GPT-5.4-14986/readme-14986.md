Research LinkedIn prospects before sales calls with Bright Data and GPT-5.4

https://n8nworkflows.xyz/workflows/research-linkedin-prospects-before-sales-calls-with-bright-data-and-gpt-5-4-14986


# Research LinkedIn prospects before sales calls with Bright Data and GPT-5.4

Let me analyze this workflow thoroughly and create a comprehensive reference document.

The workflow is about researching LinkedIn prospects before sales calls using Bright Data and GPT-5.4. Let me break it down step by step.

**Nodes in the workflow:**

1. **Daily Prospect Research** - Schedule Trigger (runs every hour)
2. **Read Upcoming Calls Sheet** - Google Sheets node (reads from upcoming_calls tab)
3. **Scrape LinkedIn Profile** - HTTP Request node (POST to Bright Data API)
4. **Validate BD Response** - Code node (validates Bright Data response)
5. **Filter Valid Data** - IF node (filters for status === 'ok')
6. **Extract Key Talking Points** - Code node (extracts sales context from profile data)
7. **Build Call Brief** - LangChain Agent node (GPT-5.4 powered analysis)
8. **GPT-5.4 Model** - LangChain OpenAI Chat Model node
9. **Parse AI Output** - Code node (parses JSON from AI response)
10. **IF Confidence >= 0.7** - IF node (confidence gate)
11. **IF Readiness Score >= 70** - IF node (readiness score gate)
12. **Email Full Brief** - Gmail node (sends full brief email)
13. **Append Call Briefs** - Google Sheets node (appends to call_briefs)
14. **Email Short Brief** - Gmail node (sends limited brief email)
15. **Append Call Briefs Partial** - Google Sheets node (appends to call_briefs for partial)
16. **Append Low Confidence** - Google Sheets node (appends to low_confidence_briefs)

Sticky Notes:
- Sticky Note (main overview) - covers the whole workflow
- Sticky Note1 - "Data Input" - covers Read Upcoming Calls Sheet
- Sticky Note2 - "Data Collection" - covers Scrape LinkedIn Profile, Validate BD Response, Filter Valid Data
- Sticky Note3 - "AI Analysis" - covers Extract Key Talking Points, Build Call Brief, GPT-5.4 Model, Parse AI Output
- Sticky Note4 - "Quality Gates & Output" - covers IF Confidence >= 0.7, IF Readiness Score >= 70, Email Full Brief, Append Call Briefs, Email Short Brief, Append Call Briefs Partial, Append Low Confidence

Let me map the connections:

- Daily Prospect Research → Read Upcoming Calls Sheet
- Read Upcoming Calls Sheet → Scrape LinkedIn Profile
- Scrape LinkedIn Profile → Validate BD Response
- Validate BD Response → Filter Valid Data
- Filter Valid Data (true branch) → Extract Key Talking Points
- Extract Key Talking Points → Build Call Brief
- GPT-5.4 Model → Build Call Brief (ai_languageModel connection)
- Build Call Brief → Parse AI Output
- Parse AI Output → IF Confidence >= 0.7
- IF Confidence >= 0.7 (true) → IF Readiness Score >= 70
- IF Confidence >= 0.7 (false) → Append Low Confidence
- IF Readiness Score >= 70 (true) → Email Full Brief, Append Call Briefs
- IF Readiness Score >= 70 (false) → Email Short Brief, Append Call Briefs Partial

Now let me organize this into logical blocks:

**Block 1: Data Input** - Daily Prospect Research, Read Upcoming Calls Sheet
**Block 2: Data Collection** - Scrape LinkedIn Profile, Validate BD Response, Filter Valid Data
**Block 3: Data Enrichment** - Extract Key Talking Points
**Block 4: AI Analysis** - Build Call Brief, GPT-5.4 Model, Parse AI Output
**Block 5: Quality Gates & Output** - IF Confidence >= 0.7, IF Readiness Score >= 70, Email Full Brief, Append Call Briefs, Email Short Brief, Append Call Briefs Partial, Append Low Confidence

Let me now write the comprehensive document.### 1. Workflow Overview

This workflow automates pre-call intelligence research for sales teams. It runs on a schedule, reads prospect records from a Google Sheet, scrapes each prospect's LinkedIn profile via the Bright Data API, enriches the data with contextual talking points, and feeds everything into GPT-5.4 to generate a structured intelligence brief. Two quality gates — a confidence threshold and a readiness score threshold — route results into three possible outcomes: a full email brief sent to the rep, a shorter limited-data email brief, or a low-confidence log for manual review.

**Logical blocks:**

- **1.1 Data Input** — Schedule trigger + Google Sheets read of upcoming calls.
- **1.2 Data Collection** — Bright Data LinkedIn scraping, response validation, and valid-data filtering.
- **1.3 Data Enrichment** — Code-driven extraction of career signals, skills, education, and rapport hooks.
- **1.4 AI Analysis** — LangChain Agent powered by GPT-5.4 producing a structured JSON brief with self-evaluation, followed by JSON parsing.
- **1.5 Quality Gates & Output** — Confidence gate (≥ 0.7), readiness gate (≥ 70), branching to full brief email + sheet append, short brief email + sheet append, or low-confidence sheet append.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Data Input

**Overview:** A schedule trigger fires the workflow on a recurring basis. It reads every row from the `upcoming_calls` tab of a Google Sheet, providing the LinkedIn URL, meeting date, product context, and optional alert email for each prospect.

**Nodes Involved:** Daily Prospect Research, Read Upcoming Calls Sheet

**Node Details:**

| Node | Details |
|------|---------|
| **Daily Prospect Research** | Type: `n8n-nodes-base.scheduleTrigger` (v1.2). Fires every hour. Starts the entire pipeline. No credentials needed. Output: empty item to trigger downstream read. |
| **Read Upcoming Calls Sheet** | Type: `n8n-nodes-base.googleSheets` (v4.7). Operation: Read rows. Sheet tab: `upcoming_calls`. Document ID placeholder: `YOUR_SPREADSHEET_ID`. Error handling: `continueRegularOutput` (errors produce empty output rather than halting). Requires Google Sheets OAuth2 credential. Output: one item per row with fields including `url`, `meeting_date`, `our_product`, `alert_email`. |

**Edge cases:**
- If the sheet is empty or the tab does not exist, the node returns zero items and the workflow naturally terminates.
- Missing `url` column causes downstream Bright Data calls to fail at the HTTP request node.

---

#### Block 1.2 — Data Collection

**Overview:** Each prospect's LinkedIn URL is sent to the Bright Data dataset API for synchronous scraping. The raw API response is then validated by a Code node that handles three scenarios: successful array data, async snapshot responses (not ready), and error objects. An IF node filters out any item whose status is not `ok`.

**Nodes Involved:** Scrape LinkedIn Profile, Validate BD Response, Filter Valid Data

**Node Details:**

| Node | Details |
|------|---------|
| **Scrape LinkedIn Profile** | Type: `n8n-nodes-base.httpRequest` (v4.2). Method: POST. URL: `https://api.brightdata.com/datasets/v3/scrape?dataset_id=gd_l1viktl72bvl7bjuj0&format=json`. Body: JSON constructed from the `url` field of the incoming item (`{ input: [{ url: $json.url }] }`). Authentication: Generic credential type → HTTP Header Auth (expects a Bearer token with Bright Data API key). Timeout: 90 seconds. Retry: up to 3 attempts with 5-second pause between retries. Requires Bright Data HTTP Header Auth credential. |
| **Validate BD Response** | Type: `n8n-nodes-base.code` (v2). Accepts the HTTP response and classifies it: (1) If it is a non-empty array → each element receives `status: 'ok'`. (2) If empty array → returns `{ error: 'Bright Data returned empty results', status: 'no_data' }`. (3) If object contains `snapshot_id` → returns `{ error: 'Async response - snapshot not ready', snapshot_id, status: 'async_pending' }`. (4) If object contains `error` or `message` → returns `{ error: <message>, status: 'api_error' }`. (5) Otherwise, passes through with `status: 'ok'`. No credentials needed. |
| **Filter Valid Data** | Type: `n8n-nodes-base.if` (v2.2). Condition: `$json.status` equals `ok`. True branch continues to enrichment; false branch is unconnected (items with errors/no data are silently dropped). Error handling: `continueErrorOutput`. |

**Edge cases:**
- Bright Data sync endpoint times out after ~1 minute; longer scrape jobs return a `snapshot_id` instead of data. The validator catches this and marks it `async_pending`, which then gets filtered out.
- If the Bright Data API key is invalid, the response typically contains an error message, caught by the validator.
- Rows where `url` is blank or malformed will still be sent to the API, likely returning an error that the validator catches.

---

#### Block 1.3 — Data Enrichment

**Overview:** A Code node examines the scraped LinkedIn profile data and extracts sales-relevant signals: career trajectory, role tenure, tech-affinity skills, education rapport hooks, recent activity themes, and an influence tier based on connection count. These enrichments are appended to the item payload before AI analysis.

**Nodes Involved:** Extract Key Talking Points

**Node Details:**

| Node | Details |
|------|---------|
| **Extract Key Talking Points** | Type: `n8n-nodes-base.code` (v2). Processes all incoming items. For each item it: (1) Analyzes `experiences` array to detect career moves and role tenure (new-in-role under 6 months → "likely evaluating tools"; established over 36 months → "focus on optimization pitch"). (2) Extracts education entries into rapport hooks. (3) Filters skills against a keyword list (`saas`, `cloud`, `api`, `data`, `analytics`, `automation`, `ai`, `machine learning`) to identify tech affinity. (4) Collects up to 5 recent post titles/texts as activity themes. (5) Computes an influence tier from connection count (>5000 = high, >1000 = medium, else growing). Output adds `talking_points`, `rapport_hooks`, and `prospect_profile` (containing `influence_tier`, `connections_count`, `talking_point_count`, `rapport_hook_count`) alongside the original data. No credentials needed. |

**Edge cases:**
- If the profile has no `experiences`, `education`, `skills`, or `posts` arrays, the node still succeeds but produces empty enrichment arrays.
- The node tolerates both `profile.experiences` and `profile.experience` field names for compatibility with different Bright Data response schemas.

---

#### Block 1.4 — AI Analysis

**Overview:** A LangChain Agent node receives the enriched profile data and instructs GPT-5.4 to produce a structured pre-call intelligence brief in raw JSON. The agent is configured with a detailed system message that specifies every output field, scoring logic, and a mandatory self-evaluation block. The raw AI output is then parsed by a Code node that strips markdown fences and merges the parsed result with the original sheet row data.

**Nodes Involved:** Build Call Brief, GPT-5.4 Model, Parse AI Output

**Node Details:**

| Node | Details |
|------|---------|
| **Build Call Brief** | Type: `@n8n/n8n-nodes-langchain.agent` (v3). Prompt type: `define`. The user prompt dynamically injects `$json.our_product` and `$json.meeting_date` and includes `JSON.stringify($json)` of the full item. The system message instructs the agent to act as a "sales intelligence analyst" and produce: `career_trajectory` (string), `recent_activity` (string), `talking_points` (exactly 5-item string array), `potential_pain_points` (string array), `readiness_score` (0–100 integer with tier definitions), `prospect_name`, `current_role`, `company`, `connection_points`, `recommended_approach`, `risk_factors`, plus a mandatory `eval` object (`confidence` 0.0–1.0, `reasoning`, `data_quality`, `evidence_count`). Output must be raw JSON only — no markdown, no code fences. Error handling: `continueErrorOutput` — errors route to the error output. Connected to GPT-5.4 Model via `ai_languageModel`. |
| **GPT-5.4 Model** | Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1). Model: `gpt-5.4`. No additional options configured. Requires an OpenAI API credential. Receives prompts from the agent node and returns model output. |
| **Parse AI Output** | Type: `n8n-nodes-base.code` (v2). Extracts the AI response from `$input.first().json.output` or `.text`. Strips any surrounding ` ```json ` / ` ``` ` markdown fences. Attempts `JSON.parse` on the cleaned string. On parse failure, returns `{ error: 'Failed to parse AI response', parse_error, raw_preview }`. Merges the parsed object with the original row data from the "Read Upcoming Calls Sheet" node (accessed via `$("Read Upcoming Calls Sheet").first().json`). Adds a `processed_at` ISO timestamp. No credentials needed. |

**Edge cases:**
- If GPT-5.4 returns text outside JSON (despite instructions), the parser attempts cleanup but may still fail, resulting in a `parse_error` object that will fail the confidence check downstream.
- The merge with original sheet data ensures that `url`, `meeting_date`, `our_product`, and `alert_email` are preserved even if the AI doesn't include them.

---

#### Block 1.5 — Quality Gates & Output

**Overview:** Two sequential IF gates control routing. First, confidence must be ≥ 0.7; otherwise the result is logged to the `low_confidence_briefs` sheet. Second, readiness score must be ≥ 70. Prospects passing both gates receive a full plain-text email brief and are appended to `call_briefs`. Prospects passing confidence but scoring below 70 on readiness receive a shorter email brief and are also appended to `call_briefs`.

**Nodes Involved:** IF Confidence >= 0.7, IF Readiness Score >= 70, Email Full Brief, Append Call Briefs, Email Short Brief, Append Call Briefs Partial, Append Low Confidence

**Node Details:**

| Node | Details |
|------|---------|
| **IF Confidence >= 0.7** | Type: `n8n-nodes-base.if` (v2.2). Condition: `$json.eval.confidence` (as number) ≥ 0.7. True → routes to readiness gate. False → routes to Append Low Confidence. Error handling: `continueErrorOutput`. |
| **IF Readiness Score >= 70** | Type: `n8n-nodes-base.if` (v2.2). Condition: `$json.readiness_score` (as number) ≥ 70. True → routes to Email Full Brief + Append Call Briefs (two outputs). False → routes to Email Short Brief + Append Call Briefs Partial (two outputs). Error handling: `continueErrorOutput`. |
| **Email Full Brief** | Type: `n8n-nodes-base.gmail` (v2.1). Email type: plain text. Recipient: `$json.alert_email` → fallback `$vars.ALERT_EMAIL` → fallback `alerts@company.com`. Subject: `Pre-call brief: {prospect_name} at {company} (Ready: {readiness_score})`. Body includes all brief fields: career trajectory, recent activity, talking points (joined by newline), potential pain points, connection points, recommended approach, risk factors. Error handling: `continueRegularOutput`. Requires Gmail OAuth2 credential. |
| **Append Call Briefs** | Type: `n8n-nodes-base.googleSheets` (v4.7). Operation: Append. Sheet tab: `call_briefs`. Document ID placeholder: `YOUR_SPREADSHEET_ID`. Mapping mode: auto-map input data. Error handling: `continueRegularOutput`. Requires Google Sheets OAuth2 credential. |
| **Email Short Brief** | Type: `n8n-nodes-base.gmail` (v2.1). Email type: plain text. Recipient: same fallback chain as Email Full Brief. Subject: `Pre-call brief (limited): {prospect_name} at {company}`. Body includes a note about sub-70 readiness score, plus career trajectory, recommended approach, and risk factors (no talking points or pain points). Error handling: `continueRegularOutput`. Requires Gmail OAuth2 credential. |
| **Append Call Briefs Partial** | Type: `n8n-nodes-base.googleSheets` (v4.7). Operation: Append. Sheet tab: `call_briefs`. Document ID placeholder: `YOUR_SPREADSHEET_ID`. Mapping mode: auto-map input data. Error handling: `continueRegularOutput`. Requires Google Sheets OAuth2 credential. |
| **Append Low Confidence** | Type: `n8n-nodes-base.googleSheets` (v4.7). Operation: Append. Sheet tab: `low_confidence_briefs`. Document ID placeholder: `YOUR_SPREADSHEET_ID`. Mapping mode: auto-map input data. Error handling: `continueRegularOutput`. Requires Google Sheets OAuth2 credential. |

**Edge cases:**
- If `alert_email` is blank and `$vars.ALERT_EMAIL` is not set, the fallback `alerts@company.com` is used — ensure this address is intentional or overridden.
- The confidence check uses `typeValidation: strict`; if `eval.confidence` is missing or non-numeric, the condition evaluates as false and routes to low-confidence.
- Readiness score check similarly uses strict type validation; a missing or non-numeric readiness_score routes to the short brief branch.
- Gmail nodes use `continueRegularOutput` on error, meaning email send failures do not prevent sheet appends.
- Sheet appends use `continueRegularOutput` on error, so a sheet write failure does not block other parallel actions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Prospect Research | scheduleTrigger | Triggers the workflow on a recurring hourly schedule | — | Read Upcoming Calls Sheet | ### How it works — This automated LinkedIn intelligence pipeline scrapes data via Bright Data, analyzes it with GPT-5.4, and filters results by confidence and domain score before writing to Google Sheets. High-priority items trigger a Gmail alert. |
| Read Upcoming Calls Sheet | googleSheets | Reads prospect rows from the `upcoming_calls` sheet tab | Daily Prospect Research | Scrape LinkedIn Profile | ## 1. Data Input — Reads LinkedIn URLs from the 'upcoming_calls' sheet. |
| Scrape LinkedIn Profile | httpRequest | POSTs each LinkedIn URL to Bright Data dataset API for profile scraping | Read Upcoming Calls Sheet | Validate BD Response | ## 2. Data Collection — Sends each URL to the Bright Data LinkedIn API and validates the response. |
| Validate BD Response | code | Classifies Bright Data response as ok, async pending, empty, or API error | Scrape LinkedIn Profile | Filter Valid Data | ## 2. Data Collection — Sends each URL to the Bright Data LinkedIn API and validates the response. |
| Filter Valid Data | if | Passes only items with status `ok` downstream; drops error/async/empty items | Validate BD Response | Extract Key Talking Points | ## 2. Data Collection — Sends each URL to the Bright Data LinkedIn API and validates the response. |
| Extract Key Talking Points | code | Enriches profile data with career signals, skills, education hooks, and influence tier | Filter Valid Data | Build Call Brief | ## 3. AI Analysis — GPT-5.4 analyzes each item and returns structured JSON with scores and self-evaluation. |
| Build Call Brief | @n8n/n8n-nodes-langchain.agent | LangChain Agent that sends enriched profile data to GPT-5.4 with a detailed system prompt requesting a structured JSON brief | Extract Key Talking Points | Parse AI Output | ## 3. AI Analysis — GPT-5.4 analyzes each item and returns structured JSON with scores and self-evaluation. |
| GPT-5.4 Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-5.4 as the language model for the Agent node | — (connected as ai_languageModel) | Build Call Brief | ## 3. AI Analysis — GPT-5.4 analyzes each item and returns structured JSON with scores and self-evaluation. |
| Parse AI Output | code | Strips markdown fences from AI response, parses JSON, merges with original row data, adds timestamp | Build Call Brief | IF Confidence >= 0.7 | ## 3. AI Analysis — GPT-5.4 analyzes each item and returns structured JSON with scores and self-evaluation. |
| IF Confidence >= 0.7 | if | Routes items with eval.confidence ≥ 0.7 to readiness gate; others to low-confidence sheet | Parse AI Output | IF Readiness Score >= 70, Append Low Confidence | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| IF Readiness Score >= 70 | if | Routes high-readiness (≥70) prospects to full brief; lower-readiness to short brief | IF Confidence >= 0.7 | Email Full Brief, Append Call Briefs, Email Short Brief, Append Call Briefs Partial | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| Email Full Brief | gmail | Sends a full plain-text pre-call brief email to the rep | IF Readiness Score >= 70 | — | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| Append Call Briefs | googleSheets | Appends full-brief results to the `call_briefs` sheet tab | IF Readiness Score >= 70 | — | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| Email Short Brief | gmail | Sends a shorter limited-data brief email for lower-readiness prospects | IF Readiness Score >= 70 | — | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| Append Call Briefs Partial | googleSheets | Appends partial-brief results to the `call_briefs` sheet tab | IF Readiness Score >= 70 | — | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| Append Low Confidence | googleSheets | Appends low-confidence results to the `low_confidence_briefs` sheet tab | IF Confidence >= 0.7 | — | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |
| Sticky Note | stickyNote | Overview documentation note | — | — | ### How it works — This automated LinkedIn intelligence pipeline scrapes data via Bright Data, analyzes it with GPT-5.4, and filters results by confidence and domain score before writing to Google Sheets. High-priority items trigger a Gmail alert. 1. A schedule trigger reads URLs from a Google Sheet 2. Each URL is sent to the Bright Data LinkedIn API 3. Responses are validated for errors, empty results, and async snapshots 4. GPT-5.4 analyzes the data and returns structured JSON with scores 5. A confidence gate (>= 0.7) filters unreliable AI outputs 6. Results route to: call_briefs, call_briefs, low_confidence_briefs. ### Setup — 1. Create a Google Sheet with a tab named 'upcoming_calls' and a 'url' column 2. Add your Bright Data API key (HTTP Header Auth, Bearer token) 3. Add your OpenAI API key 4. Connect Google Sheets via OAuth 5. Connect Gmail via OAuth for alerts. Cost: ~$0.01-0.03 per item (Bright Data) + ~$0.005 per analysis (GPT-5.4) |
| Sticky Note1 | stickyNote | Data Input section label | — | — | ## 1. Data Input — Reads LinkedIn URLs from the 'upcoming_calls' sheet. |
| Sticky Note2 | stickyNote | Data Collection section label | — | — | ## 2. Data Collection — Sends each URL to the Bright Data LinkedIn API and validates the response. |
| Sticky Note3 | stickyNote | AI Analysis section label | — | — | ## 3. AI Analysis — GPT-5.4 analyzes each item and returns structured JSON with scores and self-evaluation. |
| Sticky Note4 | stickyNote | Quality Gates & Output section label | — | — | ## 4. Quality Gates & Output — Confidence gate (>= 0.7) filters bad output. Domain gate: 'IF Readiness Score >= 70'. Routes to: call_briefs, call_briefs, low_confidence_briefs. Gmail alerts for flagged items. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Research LinkedIn prospects before sales calls with Bright Data and GPT-5.4".

2. **Add a Schedule Trigger node.**
   - Type: `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`).
   - Configure: Run every 1 hour.
   - Name it "Daily Prospect Research".

3. **Add a Google Sheets node — Read rows.**
   - Type: `Google Sheets` (`n8n-nodes-base.googleSheets`), version 4.7.
   - Operation: Read Rows.
   - Document ID: Set to your spreadsheet ID (replace `YOUR_SPREADSHEET_ID`).
   - Sheet Name: Select or enter `upcoming_calls`.
   - Options: Leave defaults.
   - On Error: Continue Regular Output.
   - Name it "Read Upcoming Calls Sheet".
   - Connect: `Daily Prospect Research` → `Read Upcoming Calls Sheet`.
   - Credential: Create/select a Google Sheets OAuth2 credential.

4. **Add an HTTP Request node — Scrape LinkedIn.**
   - Type: `HTTP Request` (`n8n-nodes-base.httpRequest`), version 4.2.
   - Method: POST.
   - URL: `https://api.brightdata.com/datasets/v3/scrape?dataset_id=gd_l1viktl72bvl7bjuj0&format=json`
   - Authentication: Generic Credential Type → HTTP Header Auth.
   - Send Body: Yes, specify body as JSON.
   - JSON Body: `={{ JSON.stringify({ input: [{ url: $json.url }] }) }}`
   - Options → Timeout: 90000 ms.
   - Retry on Fail: Yes, max 3 tries, 5000 ms between tries.
   - Name it "Scrape LinkedIn Profile".
   - Connect: `Read Upcoming Calls Sheet` → `Scrape LinkedIn Profile`.
   - Credential: Create an HTTP Header Auth credential for Bright Data (header name `Authorization`, value `Bearer YOUR_BRIGHTDATA_TOKEN`).

5. **Add a Code node — Validate BD Response.**
   - Type: `Code` (`n8n-nodes-base.code`), version 2.
   - Language: JavaScript.
   - Paste the following logic:
     - If input is a non-empty array, return each element with `status: 'ok'`.
     - If input is an empty array, return `{ error: 'Bright Data returned empty results', status: 'no_data' }`.
     - If input object has `snapshot_id`, return `{ error: 'Async response - snapshot not ready', snapshot_id: input.snapshot_id, status: 'async_pending' }`.
     - If input object has `error` or `message`, return `{ error: input.error || input.message, status: 'api_error' }`.
     - Otherwise pass through with `status: 'ok'`.
   - Name it "Validate BD Response".
   - Connect: `Scrape LinkedIn Profile` → `Validate BD Response`.

6. **Add an IF node — Filter Valid Data.**
   - Type: `IF` (`n8n-nodes-base.if`), version 2.2.
   - Condition: `$json.status` equals `ok` (string comparison, case-sensitive, strict type validation).
   - Name it "Filter Valid Data".
   - On Error: Continue Error Output.
   - Connect: `Validate BD Response` → `Filter Valid Data` (true output).

7. **Add a Code node — Extract Key Talking Points.**
   - Type: `Code` (`n8n-nodes-base.code`), version 2.
   - Language: JavaScript.
   - Paste the full extraction logic:
     - Loop over all items. For each, access `item.json.profile || item.json`.
     - Analyze `experiences`: detect career moves and role tenure (<6 months → new-in-role hook, >36 months → established optimization pitch).
     - Extract education entries as rapport hooks.
     - Filter skills against tech keywords (`saas`, `cloud`, `api`, `data`, `analytics`, `automation`, `ai`, `machine learning`) for tech affinity talking points.
     - Collect recent post titles/texts as activity themes.
     - Compute influence tier from connections count (>5000 high, >1000 medium, else growing).
     - Output: original data plus `talking_points`, `rapport_hooks`, and `prospect_profile` object.
   - Name it "Extract Key Talking Points".
   - Connect: `Filter Valid Data` (true) → `Extract Key Talking Points`.

8. **Add a LangChain Agent node — Build Call Brief.**
   - Type: `Agent` (`@n8n/n8n-nodes-langchain.agent`), version 3.
   - Prompt Type: Define.
   - User message: `=Build a pre-call intelligence brief for this prospect. Our product: {{ $json.our_product }}. Meeting date: {{ $json.meeting_date }}\n\n{{ JSON.stringify($json) }}`
   - System message (paste the full instruction):
     - Role: "You are a sales intelligence analyst."
     - Output fields: `career_trajectory`, `recent_activity`, `talking_points` (exactly 5), `potential_pain_points`, `readiness_score` (0-100 with tier definitions), `prospect_name`, `current_role`, `company`, `connection_points`, `recommended_approach`, `risk_factors`.
     - Mandatory `eval` object: `confidence` (0.0-1.0), `reasoning`, `data_quality` (high/medium/low), `evidence_count` (integer).
     - CRITICAL: Return ONLY raw JSON. No markdown, no code fences, no explanations.
   - On Error: Continue Error Output.
   - Name it "Build Call Brief".
   - Connect: `Extract Key Talking Points` → `Build Call Brief`.

9. **Add a LangChain OpenAI Chat Model node — GPT-5.4 Model.**
   - Type: `LM Chat OpenAI` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`), version 1.
   - Model: `gpt-5.4`.
   - Options: Leave defaults.
   - Name it "GPT-5.4 Model".
   - Connect: `GPT-5.4 Model` → `Build Call Brief` (ai_languageModel connection).
   - Credential: Create/select an OpenAI API credential.

10. **Add a Code node — Parse AI Output.**
    - Type: `Code` (`n8n-nodes-base.code`), version 2.
    - Language: JavaScript.
    - Logic:
      - Extract raw text from `$input.first().json.output` or `.text`.
      - Strip markdown code fences (` ```json ` and ` ``` `).
      - Attempt `JSON.parse`. On failure, return an object with `error`, `parse_error`, and `raw_preview` (first 200 chars).
      - Merge parsed result with original row from `$("Read Upcoming Calls Sheet").first().json`.
      - Add `processed_at` as ISO timestamp.
    - Name it "Parse AI Output".
    - Connect: `Build Call Brief` → `Parse AI Output`.

11. **Add an IF node — Confidence gate.**
    - Type: `IF` (`n8n-nodes-base.if`), version 2.2.
    - Condition: `$json.eval.confidence` (number) ≥ `0.7`.
    - Combinator: AND. Type validation: strict.
    - On Error: Continue Error Output.
    - Name it "IF Confidence >= 0.7".
    - Connect: `Parse AI Output` → `IF Confidence >= 0.7`.
    - True output → `IF Readiness Score >= 70`.
    - False output → `Append Low Confidence`.

12. **Add an IF node — Readiness gate.**
    - Type: `IF` (`n8n-nodes-base.if`), version 2.2.
    - Condition: `$json.readiness_score` (number) ≥ `70`.
    - Combinator: AND. Type validation: strict.
    - On Error: Continue Error Output.
    - Name it "IF Readiness Score >= 70".
    - Connect: `IF Confidence >= 0.7` (true) → `IF Readiness Score >= 70`.

13. **Add a Gmail node — Email Full Brief.**
    - Type: `Gmail` (`n8n-nodes-base.gmail`), version 2.1.
    - Email Type: Plain text.
    - To: `={{ $json.alert_email || $vars.ALERT_EMAIL || 'alerts@company.com' }}`
    - Subject: `=Pre-call brief: {{ $json.prospect_name }} at {{ $json.company }} (Ready: {{ $json.readiness_score }})`
    - Message body: includes prospect name, role, company, meeting date, readiness score, career trajectory, recent activity, talking points (array joined by newline), potential pain points (array joined by newline), connection points, recommended approach, risk factors.
    - On Error: Continue Regular Output.
    - Name it "Email Full Brief".
    - Connect: `IF Readiness Score >= 70` (true) → `Email Full Brief`.
    - Credential: Create/select a Gmail OAuth2 credential.

14. **Add a Google Sheets node — Append Call Briefs (full).**
    - Type: `Google Sheets` (`n8n-nodes-base.googleSheets`), version 4.7.
    - Operation: Append.
    - Document ID: Your spreadsheet ID.
    - Sheet Name: `call_briefs`.
    - Mapping: Auto-map input data.
    - On Error: Continue Regular Output.
    - Name it "Append Call Briefs".
    - Connect: `IF Readiness Score >= 70` (true) → `Append Call Briefs`.
    - Credential: Google Sheets OAuth2 (same as step 3).

15. **Add a Gmail node — Email Short Brief.**
    - Type: `Gmail` (`n8n-nodes-base.gmail`), version 2.1.
    - Email Type: Plain text.
    - To: `={{ $json.alert_email || $vars.ALERT_EMAIL || 'alerts@company.com' }}`
    - Subject: `=Pre-call brief (limited): {{ $json.prospect_name }} at {{ $json.company }}`
    - Message body: includes a note about sub-70 readiness, prospect name, role, company, meeting date, readiness score, career trajectory, recommended approach, risk factors.
    - On Error: Continue Regular Output.
    - Name it "Email Short Brief".
    - Connect: `IF Readiness Score >= 70` (false) → `Email Short Brief`.
    - Credential: Same Gmail OAuth2 credential.

16. **Add a Google Sheets node — Append Call Briefs (partial).**
    - Type: `Google Sheets` (`n8n-nodes-base.googleSheets`), version 4.7.
    - Operation: Append.
    - Document ID: Your spreadsheet ID.
    - Sheet Name: `call_briefs`.
    - Mapping: Auto-map input data.
    - On Error: Continue Regular Output.
    - Name it "Append Call Briefs Partial".
    - Connect: `IF Readiness Score >= 70` (false) → `Append Call Briefs Partial`.
    - Credential: Same Google Sheets OAuth2 credential.

17. **Add a Google Sheets node — Append Low Confidence.**
    - Type: `Google Sheets` (`n8n-nodes-base.googleSheets`), version 4.7.
    - Operation: Append.
    - Document ID: Your spreadsheet ID.
    - Sheet Name: `low_confidence_briefs`.
    - Mapping: Auto-map input data.
    - On Error: Continue Regular Output.
    - Name it "Append Low Confidence".
    - Connect: `IF Confidence >= 0.7` (false) → `Append Low Confidence`.
    - Credential: Same Google Sheets OAuth2 credential.

18. **Prepare the Google Sheet.**
    - Create a spreadsheet with three tabs: `upcoming_calls`, `call_briefs`, `low_confidence_briefs`.
    - In `upcoming_calls`, add columns at minimum: `url`, `meeting_date`, `our_product`, `alert_email` (optional).
    - Add one or more rows with LinkedIn profile URLs and meeting dates.
    - Copy the spreadsheet ID and replace all `YOUR_SPREADSHEET_ID` occurrences in the Google Sheets nodes.

19. **Configure environment variables (optional).**
    - Set `ALERT_EMAIL` as a workflow variable or n8n environment variable to provide a default recipient for brief emails.

20. **Activate the workflow.**
    - Enable the workflow so the schedule trigger runs automatically.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This is a demo workflow intended for template and educational use. Adjust confidence (0.7) and readiness (70) thresholds to match your sales process. | General |
| Bright Data's synchronous endpoint has a ~1 minute timeout. Longer scraping jobs return a `snapshot_id` instead of data. For production, consider using Bright Data's async approach — copy their cURL example and convert it into an n8n HTTP Request setup. | Bright Data async guidance |
| The AI node is instructed to return raw JSON only, making the output easier to parse and route downstream. | AI output format |
| The Gmail steps send plain-text summaries, which makes the workflow easy to customize into HTML templates if desired. | Email format |
| Estimated cost: ~$0.01–0.03 per item (Bright Data) + ~$0.005 per analysis (GPT-5.4). | Cost reference |
| Good fit for: sales teams, founders doing outbound, agencies preparing for discovery calls, account executives wanting better call prep, RevOps teams building lightweight pre-call research systems. | Target users |