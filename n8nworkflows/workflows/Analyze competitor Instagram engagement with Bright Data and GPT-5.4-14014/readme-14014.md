Analyze competitor Instagram engagement with Bright Data and GPT-5.4

https://n8nworkflows.xyz/workflows/analyze-competitor-instagram-engagement-with-bright-data-and-gpt-5-4-14014


# Analyze competitor Instagram engagement with Bright Data and GPT-5.4

# 1. Workflow Overview

This workflow performs recurring competitor intelligence on Instagram accounts. On a schedule, it reads a list of competitor profile URLs from Google Sheets, scrapes profile data using Bright Data, derives baseline engagement metrics, asks GPT-5.4 to evaluate engagement quality, and routes the result into different Google Sheets depending on confidence and score. High-performing competitors also trigger a Gmail alert.

Typical use cases:
- Monitor a list of competitor Instagram accounts daily
- Identify accounts with strong engagement worth studying
- Maintain a historical log of engagement analyses
- Separate low-confidence AI outputs for manual review

## 1.1 Input Reception and Iteration

The workflow starts on a scheduled trigger, loads competitor accounts from a Google Sheet, and processes them one record at a time.

## 1.2 External Scraping and Data Validation

Each Instagram URL is sent to the Bright Data Instagram dataset API. The response is then normalized and checked for empty, error, async, or valid cases.

## 1.3 Metric Enrichment

The workflow calculates raw engagement-related metrics such as estimated engagement rate and posting frequency from the scraped profile data.

## 1.4 AI-Based Competitive Analysis

GPT-5.4 receives the scraped and calculated data plus competitor context and returns a structured JSON analysis with subscores, overall engagement score, qualitative insights, and a mandatory self-evaluation object.

## 1.5 Quality Gates and Output Routing

The workflow filters outputs by AI confidence first, then by engagement score. Low-confidence results are written to a review sheet. High-confidence, high-score competitors are written to a top performers sheet and emailed to the team. Other high-confidence results are stored in a general analysis sheet.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Iteration

### Overview
This block starts the workflow on a schedule, reads the source list of competitor accounts from Google Sheets, and iterates through them sequentially. Sequential processing helps avoid rate spikes against Bright Data, OpenAI, and Google APIs.

### Nodes Involved
- Daily Engagement Scan
- Read Competitor Accounts
- Process Items One by One

### Node Details

#### 1. Daily Engagement Scan
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger that launches the workflow periodically.
- **Configuration choices:** Configured with an hourly interval rule. Despite the sticky note saying “daily at 8 AM,” the actual JSON indicates an interval on the `hours` field, which means it is not explicitly fixed to 8 AM in the exported configuration.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Read Competitor Accounts**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Misalignment between intended schedule and actual configured schedule
  - Workflow inactive in n8n
  - Timezone misunderstandings if later changed to cron
- **Sub-workflow reference:** None.

#### 2. Read Competitor Accounts
- **Type and role:** `n8n-nodes-base.googleSheets`; reads source records from a Google Sheet tab containing competitor accounts.
- **Configuration choices:** Targets document ID `YOUR_SPREADSHEET_ID` and sheet/tab ID value `competitor_accounts`. No additional filters or options are configured.
- **Key expressions or variables used:** Downstream nodes expect at least:
  - `url`
  - `competitor_name`
  - `our_industry`
  - `team_email` (optional fallback exists later)
- **Input and output connections:** Input from **Daily Engagement Scan**; output to **Process Items One by One**.
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth credentials in n8n.
- **Edge cases or potential failure types:**
  - OAuth authorization failure
  - Spreadsheet or tab not found
  - Empty sheet
  - Missing expected columns, especially `url`
  - Access denied if spreadsheet is not shared correctly
- **Sub-workflow reference:** None.

#### 3. Process Items One by One
- **Type and role:** `n8n-nodes-base.splitInBatches`; batch controller used here as an iterator.
- **Configuration choices:** No explicit batch size shown in parameters, so it uses default behavior. It is wired in a loop so each completed branch returns here to process the next item.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Primary input from **Read Competitor Accounts**
  - Main output to **Scrape Instagram Profiles**
  - Loop-back inputs from **Append Low Confidence**, **Append Top Performers**, and **Append Engagement Analysis**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Incorrect loop wiring can cause items not to continue
  - If a downstream branch ends without returning, later items may not process
  - If an upstream sheet returns zero rows, nothing happens
- **Sub-workflow reference:** None.

---

## Block 2 — External Scraping and Data Validation

### Overview
This block sends each competitor Instagram URL to Bright Data’s scraping dataset API and interprets the result. It distinguishes valid synchronous results from empty responses, API errors, and async snapshot responses.

### Nodes Involved
- Scrape Instagram Profiles
- Validate BD Response

### Node Details

#### 4. Scrape Instagram Profiles
- **Type and role:** `n8n-nodes-base.httpRequest`; calls Bright Data dataset API to scrape Instagram profile data.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.brightdata.com/datasets/v3/scrape?dataset_id=gd_l1vikfch901nx3by4&format=json`
  - Body format: JSON
  - Timeout: 60000 ms
  - Authentication: generic credential type using HTTP Header Auth
  - Request body sends:
    - `input: [{ url: $json.url }]`
- **Key expressions or variables used:**
  - `{{ $json.url }}` from the Google Sheets row
  - Body is assembled via `JSON.stringify({ input: [{ url: $json.url }] })`
- **Input and output connections:** Input from **Process Items One by One**; output to **Validate BD Response**.
- **Version-specific requirements:** Type version `4.2`; requires an HTTP Header Auth credential, typically `Authorization: Bearer <BRIGHT_DATA_API_KEY>`.
- **Edge cases or potential failure types:**
  - Missing or malformed `url`
  - Bright Data auth failure
  - Dataset ID invalid or unavailable
  - Timeout if scraping takes too long
  - Bright Data returns an async `snapshot_id` instead of inline data
  - Rate limiting or quota exhaustion
- **Sub-workflow reference:** None.

#### 5. Validate BD Response
- **Type and role:** `n8n-nodes-base.code`; normalizes Bright Data response and adds a `status` field.
- **Configuration choices:** JavaScript logic handles several response shapes:
  - Array response with items → returns each item with `status: 'ok'`
  - Empty array → returns error object with `status: 'no_data'`
  - Object with `snapshot_id` → returns error object with `status: 'async_pending'`
  - Object with `error` or `message` → returns error object with `status: 'api_error'`
  - Otherwise passes through object with `status: 'ok'`
- **Key expressions or variables used:** Uses `$input.first().json`.
- **Input and output connections:** Input from **Scrape Instagram Profiles**; output to **Calculate Raw Engagement Metrics**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If Bright Data returns an unexpected object shape, it may be treated as valid
  - Async snapshots are not retried later in this workflow
  - Error objects continue downstream, which may cause weak or failed AI analysis later
- **Sub-workflow reference:** None.

---

## Block 3 — Metric Enrichment

### Overview
This block computes approximate engagement metrics from available profile fields before handing the data to the AI model. It gives the model a more analysis-ready input even if the source schema varies slightly.

### Nodes Involved
- Calculate Raw Engagement Metrics

### Node Details

#### 6. Calculate Raw Engagement Metrics
- **Type and role:** `n8n-nodes-base.code`; computes estimated engagement rate and posts per week.
- **Configuration choices:**
  - Reads all incoming items
  - Uses fallback field names:
    - followers = `followers_count || follower_count || 1`
    - avg likes = `avg_likes || edge_liked_by || 0`
    - avg comments = `avg_comments || 0`
  - Computes:
    - `calculated_engagement_rate = ((avgLikes + avgComments) / followers * 100).toFixed(2)`
    - `posts_per_week = media_count ? Math.min(media_count / 52, 20).toFixed(1) : 'unknown'`
  - Adds `metrics_calculated: true`
- **Key expressions or variables used:** Direct JavaScript field access on `item.json`.
- **Input and output connections:** Input from **Validate BD Response**; output to **Analyze Engagement**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the previous node produced an error object, this node still computes using defaults, which may produce misleading values
  - Follower count fallback to `1` avoids divide-by-zero but can inflate engagement rate
  - `posts_per_week` is a rough estimate based on total media count, not actual recent cadence
  - If schema fields differ from expected Bright Data fields, metrics may be inaccurate
- **Sub-workflow reference:** None.

---

## Block 4 — AI-Based Competitive Analysis

### Overview
This block sends the enriched profile data to GPT-5.4 using an AI Agent node configured with a strict JSON-only system prompt. The model is instructed to score engagement and include a self-evaluation object. A parsing node then converts the response into structured data merged with the original spreadsheet row.

### Nodes Involved
- Analyze Engagement
- GPT-5.4 Model
- Parse AI Output

### Node Details

#### 7. Analyze Engagement
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates an LLM call for structured competitor analysis.
- **Configuration choices:**
  - Prompt includes:
    - `competitor_name`
    - `our_industry`
    - full scraped JSON payload
  - System message defines scoring rubric:
    - `follower_growth_signal` 0–25
    - `engagement_rate_quality` 0–25
    - `content_consistency` 0–25
    - `audience_authenticity` 0–25
    - total `engagement_score` 0–100
  - Strict output schema required as raw JSON only
  - Mandatory `eval` object:
    - `confidence`
    - `reasoning`
    - `data_quality`
    - `evidence_count`
  - `onError` is set to `continueErrorOutput`
- **Key expressions or variables used:**
  - `{{ $json.competitor_name }}`
  - `{{ $json.our_industry }}`
  - `{{ JSON.stringify($json, null, 2) }}`
- **Input and output connections:**
  - Main input from **Calculate Raw Engagement Metrics**
  - AI language model input from **GPT-5.4 Model**
  - Main output to **Parse AI Output**
- **Version-specific requirements:** Type version `3`; requires LangChain/OpenAI-compatible setup in n8n.
- **Edge cases or potential failure types:**
  - Model may still return non-JSON text despite instructions
  - Incomplete source data can lower quality
  - Token limits if payload becomes too large
  - OpenAI auth or rate-limit errors
  - Because `continueErrorOutput` is enabled, downstream nodes may receive error-shaped outputs
- **Sub-workflow reference:** None.

#### 8. GPT-5.4 Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides the OpenAI chat model used by the agent.
- **Configuration choices:**
  - Model name: `gpt-5.4`
  - No extra options configured
- **Key expressions or variables used:** None.
- **Input and output connections:** Supplies AI model connection to **Analyze Engagement**.
- **Version-specific requirements:** Type version `1`; requires OpenAI credentials configured in n8n.
- **Edge cases or potential failure types:**
  - Model name availability depends on account access and current n8n/OpenAI integration support
  - Invalid API key or insufficient quota
  - API latency or provider-side transient failures
- **Sub-workflow reference:** None.

#### 9. Parse AI Output
- **Type and role:** `n8n-nodes-base.code`; cleans and parses model output into JSON, then merges it with the original competitor row.
- **Configuration choices:**
  - Reads model output from `output` or `text`
  - Strips ```json and ``` fences defensively
  - Attempts `JSON.parse`
  - On parse failure, returns:
    - `error: 'Failed to parse AI response'`
    - `parse_error`
    - `raw_preview`
  - Merges parsed result with the first item from **Read Competitor Accounts**
  - Adds `processed_at` timestamp
- **Key expressions or variables used:**
  - `$input.first().json.output`
  - `$input.first().json.text`
  - `$("Read Competitor Accounts").first().json`
  - `new Date().toISOString()`
- **Input and output connections:** Input from **Analyze Engagement**; output to **IF Confidence >= 0.7**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Major logic issue: it always merges with `$("Read Competitor Accounts").first().json`, not the current iterated item. In multi-row runs, this can attach the wrong competitor metadata to later analyses.
  - If AI output is malformed, downstream confidence check may fail because `eval.confidence` is missing
  - Fence stripping only handles simple markdown cases
- **Sub-workflow reference:** None.

---

## Block 5 — Quality Gates and Output Routing

### Overview
This block routes results according to AI confidence and engagement score. It ensures questionable outputs go to manual review, while strong high-confidence competitors are highlighted via a dedicated sheet and email alert.

### Nodes Involved
- IF Confidence >= 0.7
- IF Engagement >= 75
- Alert Top Performer
- Append Top Performers
- Append Engagement Analysis
- Append Low Confidence

### Node Details

#### 10. IF Confidence >= 0.7
- **Type and role:** `n8n-nodes-base.if`; checks whether the AI self-evaluation confidence meets threshold.
- **Configuration choices:**
  - Numeric comparison: `{{ $json.eval.confidence }} >= 0.7`
  - Strict type validation enabled
  - `onError` is `continueErrorOutput`
- **Key expressions or variables used:** `{{ $json.eval.confidence }}`
- **Input and output connections:**
  - Input from **Parse AI Output**
  - True output to **IF Engagement >= 75**
  - False output to **Append Low Confidence**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If `eval` or `confidence` is missing, strict numeric validation may fail
  - Because error continuation is enabled, malformed items may still proceed as error output depending on runtime behavior
- **Sub-workflow reference:** None.

#### 11. IF Engagement >= 75
- **Type and role:** `n8n-nodes-base.if`; identifies top-performing competitors based on total engagement score.
- **Configuration choices:**
  - Numeric comparison: `{{ $json.engagement_score }} >= 75`
  - Strict type validation enabled
  - `onError` is `continueErrorOutput`
- **Key expressions or variables used:** `{{ $json.engagement_score }}`
- **Input and output connections:**
  - Input from **IF Confidence >= 0.7**
  - True output to **Alert Top Performer** and **Append Top Performers**
  - False output to **Append Engagement Analysis**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Missing or non-numeric `engagement_score`
  - Parsing failures upstream create malformed items
- **Sub-workflow reference:** None.

#### 12. Alert Top Performer
- **Type and role:** `n8n-nodes-base.gmail`; sends an email alert for strong competitors.
- **Configuration choices:**
  - Recipient:
    - `{{ $('Read Competitor Accounts').first().json.team_email || 'team@company.com' }}`
  - Subject includes competitor name and score
  - Message includes competitor name, score, key strengths, and strategic takeaway
  - Email type: plain text
- **Key expressions or variables used:**
  - `$('Read Competitor Accounts').first().json.team_email || 'team@company.com'`
  - `{{ $json.competitor_name }}`
  - `{{ $json.engagement_score }}`
  - `{{ $json.key_strengths }}`
  - `{{ $json.strategic_takeaway }}`
- **Input and output connections:** Input from **IF Engagement >= 75** true branch. No downstream connection.
- **Version-specific requirements:** Type version `2.1`; requires Gmail OAuth credentials.
- **Edge cases or potential failure types:**
  - Same metadata bug as above: using `.first()` from the sheet can email the wrong recipient when multiple rows are processed
  - Gmail OAuth permission issues
  - Sending limits or blocked account policies
  - Arrays like `key_strengths` may render as comma-separated text or imperfectly formatted string
- **Sub-workflow reference:** None.

#### 13. Append Top Performers
- **Type and role:** `n8n-nodes-base.googleSheets`; appends top-performing, high-confidence results to a dedicated sheet.
- **Configuration choices:**
  - Operation: `append`
  - Target tab: `top_performers`
  - Mapping mode: auto-map input data
- **Key expressions or variables used:** None explicitly; all input fields are passed by automapping.
- **Input and output connections:**
  - Input from **IF Engagement >= 75** true branch
  - Output loops back to **Process Items One by One**
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth.
- **Edge cases or potential failure types:**
  - Missing sheet tab
  - Auto-mapping may create blank columns if headers do not match incoming keys
  - Arrays/objects may be stringified inconsistently in sheet cells
- **Sub-workflow reference:** None.

#### 14. Append Engagement Analysis
- **Type and role:** `n8n-nodes-base.googleSheets`; appends normal high-confidence analyses that do not meet top-performer threshold.
- **Configuration choices:**
  - Operation: `append`
  - Target tab: `engagement_analysis`
  - Mapping mode: auto-map input data
- **Key expressions or variables used:** None explicitly.
- **Input and output connections:**
  - Input from **IF Engagement >= 75** false branch
  - Output loops back to **Process Items One by One**
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Same Google Sheets append and schema alignment issues as above
- **Sub-workflow reference:** None.

#### 15. Append Low Confidence
- **Type and role:** `n8n-nodes-base.googleSheets`; stores low-confidence or questionable AI analyses for human review.
- **Configuration choices:**
  - Operation: `append`
  - Target tab: `low_confidence_engagement`
  - Mapping mode: auto-map input data
- **Key expressions or variables used:** None explicitly.
- **Input and output connections:**
  - Input from **IF Confidence >= 0.7** false branch
  - Output loops back to **Process Items One by One**
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Same Google Sheets append and schema alignment issues as above
  - Parse failures may leave sparse records without `eval`
- **Sub-workflow reference:** None.

---

## Block 6 — Documentation / In-Canvas Notes

### Overview
These nodes do not execute business logic. They document the workflow purpose, setup, and main stages directly in the n8n canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### 16. Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; overall workflow documentation.
- **Configuration choices:** Contains explanation of workflow purpose, setup steps, and approximate costs.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### 17. Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`; describes the input stage.
- **Configuration choices:** Notes daily trigger and one-by-one processing.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### 18. Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`; describes scraping and validation stage.
- **Configuration choices:** Notes Bright Data scraping and metric calculation.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### 19. Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`; describes AI scoring stage.
- **Configuration choices:** Notes GPT-5.4 engagement scoring dimensions.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### 20. Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`; describes quality gates and outputs.
- **Configuration choices:** Notes confidence threshold, engagement threshold, output sheets, and Gmail alerts.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Engagement Scan | Schedule Trigger | Starts the workflow on schedule |  | Read Competitor Accounts | ## 1. Data Input\nTriggers daily at 8 AM. Reads competitor Instagram profile URLs from the 'competitor_accounts' sheet and processes them one at a time. |
| Read Competitor Accounts | Google Sheets | Reads source competitor rows | Daily Engagement Scan | Process Items One by One | ## 1. Data Input\nTriggers daily at 8 AM. Reads competitor Instagram profile URLs from the 'competitor_accounts' sheet and processes them one at a time. |
| Process Items One by One | Split In Batches | Iterates through rows sequentially | Read Competitor Accounts; Append Low Confidence; Append Top Performers; Append Engagement Analysis | Scrape Instagram Profiles | ## 1. Data Input\nTriggers daily at 8 AM. Reads competitor Instagram profile URLs from the 'competitor_accounts' sheet and processes them one at a time. |
| Scrape Instagram Profiles | HTTP Request | Calls Bright Data Instagram dataset API | Process Items One by One | Validate BD Response | ## 2. Scrape & Validate\nSends each URL to the Bright Data Instagram API, validates the response, and calculates raw engagement metrics. |
| Validate BD Response | Code | Normalizes Bright Data response and flags status | Scrape Instagram Profiles | Calculate Raw Engagement Metrics | ## 2. Scrape & Validate\nSends each URL to the Bright Data Instagram API, validates the response, and calculates raw engagement metrics. |
| Calculate Raw Engagement Metrics | Code | Computes estimated engagement rate and posting cadence | Validate BD Response | Analyze Engagement | ## 2. Scrape & Validate\nSends each URL to the Bright Data Instagram API, validates the response, and calculates raw engagement metrics. |
| Analyze Engagement | AI Agent | Uses GPT to score engagement and generate strategic analysis | Calculate Raw Engagement Metrics; GPT-5.4 Model | Parse AI Output | ## 3. AI Analysis\nGPT-5.4 scores each profile on follower growth, engagement quality, content consistency, and audience authenticity (0-100). |
| GPT-5.4 Model | OpenAI Chat Model | Supplies the LLM for the AI Agent node |  | Analyze Engagement | ## 3. AI Analysis\nGPT-5.4 scores each profile on follower growth, engagement quality, content consistency, and audience authenticity (0-100). |
| Parse AI Output | Code | Parses JSON output and merges source metadata | Analyze Engagement | IF Confidence >= 0.7 | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| IF Confidence >= 0.7 | IF | Routes by AI confidence threshold | Parse AI Output | IF Engagement >= 75; Append Low Confidence | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| IF Engagement >= 75 | IF | Routes high-confidence items by engagement score | IF Confidence >= 0.7 | Alert Top Performer; Append Top Performers; Append Engagement Analysis | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| Alert Top Performer | Gmail | Emails the team when a competitor scores highly | IF Engagement >= 75 |  | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| Append Top Performers | Google Sheets | Stores top-performing competitors | IF Engagement >= 75 | Process Items One by One | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| Append Engagement Analysis | Google Sheets | Stores standard high-confidence analyses | IF Engagement >= 75 | Process Items One by One | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| Append Low Confidence | Google Sheets | Stores low-confidence analyses for review | IF Confidence >= 0.7 | Process Items One by One | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |
| Sticky Note | Sticky Note | General workflow explanation and setup notes |  |  | ### How it works\nThis workflow monitors competitor Instagram accounts on a daily schedule. It reads profile URLs from a Google Sheet, scrapes each one through the Bright Data Instagram Profiles API, and calculates raw engagement metrics like engagement rate and posting frequency.\n\nGPT-5.4 then analyzes each profile across four dimensions: follower growth signals, engagement rate quality, content consistency, and audience authenticity. Each dimension is scored 0-25 for a total engagement score of 0-100.\n\nTwo quality gates filter the results. A confidence check (>= 0.7) discards unreliable AI outputs. An engagement threshold (>= 75) separates top performers from the rest. High scorers go to the 'top_performers' sheet and trigger a Gmail alert. All other results go to 'engagement_analysis'. Low-confidence outputs land in 'low_confidence_engagement' for review.\n\n### Setup\n1. Create a Google Sheet with a tab named 'competitor_accounts' and a 'url' column containing Instagram profile URLs\n2. Add your Bright Data API key as an HTTP Header Auth credential (Bearer token)\n3. Add your OpenAI API key\n4. Connect Google Sheets via OAuth\n5. Connect Gmail via OAuth for alerts\n6. Update the recipient in the 'Alert Top Performer' node\n\nCost: ~$0.01-0.03 per profile (Bright Data) + ~$0.005 per analysis (GPT-5.4) |
| Sticky Note1 | Sticky Note | Canvas note for input block |  |  | ## 1. Data Input\nTriggers daily at 8 AM. Reads competitor Instagram profile URLs from the 'competitor_accounts' sheet and processes them one at a time. |
| Sticky Note2 | Sticky Note | Canvas note for scraping block |  |  | ## 2. Scrape & Validate\nSends each URL to the Bright Data Instagram API, validates the response, and calculates raw engagement metrics. |
| Sticky Note3 | Sticky Note | Canvas note for AI block |  |  | ## 3. AI Analysis\nGPT-5.4 scores each profile on follower growth, engagement quality, content consistency, and audience authenticity (0-100). |
| Sticky Note4 | Sticky Note | Canvas note for routing block |  |  | ## 4. Quality Gates & Output\nConfidence gate (>= 0.7) and engagement threshold (>= 75) route results to three sheets. Top performers trigger Gmail alerts. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it: **Analyze competitor Instagram engagement with Bright Data and GPT-5.4**.

2. **Prepare the Google Sheet**
   - Create a spreadsheet.
   - Add these tabs:
     1. `competitor_accounts`
     2. `top_performers`
     3. `engagement_analysis`
     4. `low_confidence_engagement`
   - In `competitor_accounts`, add at minimum:
     - `url`
   - Recommended additional columns because the workflow references them:
     - `competitor_name`
     - `our_industry`
     - `team_email`

3. **Create credentials**
   - **Google Sheets OAuth2**
     - Connect the spreadsheet account used to read and append rows.
   - **Gmail OAuth2**
     - Connect the mailbox that will send alerts.
   - **OpenAI credential**
     - Add your API key.
   - **HTTP Header Auth credential for Bright Data**
     - Add header:
       - `Authorization`
       - Value: `Bearer YOUR_BRIGHT_DATA_API_KEY`

4. **Add the trigger node**
   - Create a **Schedule Trigger** node named **Daily Engagement Scan**.
   - Configure the schedule.
   - Note: the exported workflow uses an hourly interval rule, even though comments say daily at 8 AM.
   - If you want to match the canvas note, set it explicitly to daily at 8 AM in your timezone.

5. **Add the source Google Sheets node**
   - Create a **Google Sheets** node named **Read Competitor Accounts**.
   - Connect **Daily Engagement Scan** → **Read Competitor Accounts**.
   - Set the spreadsheet document ID.
   - Select the sheet/tab `competitor_accounts`.
   - Use the Google Sheets OAuth credential.
   - Configure it to read rows from the tab.

6. **Add the iterator**
   - Create a **Split In Batches** node named **Process Items One by One**.
   - Connect **Read Competitor Accounts** → **Process Items One by One**.
   - Leave default options unless you want a custom batch size.
   - This node will later receive loop-back connections from output nodes.

7. **Add the Bright Data request**
   - Create an **HTTP Request** node named **Scrape Instagram Profiles**.
   - Connect **Process Items One by One** → **Scrape Instagram Profiles**.
   - Configure:
     - Method: `POST`
     - URL: `https://api.brightdata.com/datasets/v3/scrape?dataset_id=gd_l1vikfch901nx3by4&format=json`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Header Auth**
     - Credential: Bright Data header credential
     - Send Body: enabled
     - Body Content Type / Specify Body: JSON
     - Timeout: `60000`
   - Set the JSON body to send one profile URL from the current item:
     - `{"input":[{"url":"={{ $json.url }}"}]}`
   - If your n8n UI expects an expression for the entire body, use the equivalent object-expression approach.

8. **Add response validation**
   - Create a **Code** node named **Validate BD Response**.
   - Connect **Scrape Instagram Profiles** → **Validate BD Response**.
   - Paste logic equivalent to:
     - If input is an array:
       - return `status: ok` for each item
       - or `status: no_data` if empty
     - If response contains `snapshot_id`:
       - return `status: async_pending`
     - If response contains `error` or `message`:
       - return `status: api_error`
     - Else:
       - pass through with `status: ok`

9. **Add raw metric calculation**
   - Create another **Code** node named **Calculate Raw Engagement Metrics**.
   - Connect **Validate BD Response** → **Calculate Raw Engagement Metrics**.
   - Compute:
     - `followers = followers_count || follower_count || 1`
     - `avgLikes = avg_likes || edge_liked_by || 0`
     - `avgComments = avg_comments || 0`
     - `calculated_engagement_rate = ((avgLikes + avgComments) / followers * 100)`
     - `posts_per_week = media_count ? min(media_count / 52, 20) : 'unknown'`
     - `metrics_calculated = true`

10. **Add the OpenAI model node**
    - Create an **OpenAI Chat Model** node named **GPT-5.4 Model**.
    - Set model to `gpt-5.4`.
    - Attach your OpenAI credential.

11. **Add the AI Agent node**
    - Create an **AI Agent** node named **Analyze Engagement**.
    - Connect **Calculate Raw Engagement Metrics** → **Analyze Engagement**.
    - Connect **GPT-5.4 Model** to the AI language model input of **Analyze Engagement**.
    - Set prompt type to define text manually.
    - In the user prompt/body, include:
      - competitor name
      - industry
      - full JSON payload
    - In the system message, define:
      - the four rubric categories
      - 0–25 scoring for each
      - total engagement score 0–100
      - strict JSON-only output requirement
      - required output schema including:
        - `competitor_name`
        - `follower_count`
        - category scores
        - `engagement_score`
        - `top_content_types`
        - `posting_frequency`
        - `key_strengths`
        - `key_weaknesses`
        - `strategic_takeaway`
        - `eval` object with `confidence`, `reasoning`, `data_quality`, `evidence_count`
    - Set node error handling to continue if you want parity with the exported workflow.

12. **Add AI output parsing**
    - Create a **Code** node named **Parse AI Output**.
    - Connect **Analyze Engagement** → **Parse AI Output**.
    - Configure it to:
      - read raw model output from `output` or `text`
      - strip code fences if present
      - `JSON.parse()` the cleaned string
      - on failure, create an object with parse diagnostics
      - merge parsed output with source metadata
      - add `processed_at`
    - Important recommended improvement:
      - Do **not** use `$("Read Competitor Accounts").first().json` in production for multi-item runs.
      - Instead, preserve the current row’s metadata in the item itself before the AI call, or merge with the current incoming item directly.

13. **Add confidence gate**
    - Create an **IF** node named **IF Confidence >= 0.7**.
    - Connect **Parse AI Output** → **IF Confidence >= 0.7**.
    - Add a numeric condition:
      - left value: `{{ $json.eval.confidence }}`
      - operation: greater than or equal
      - right value: `0.7`
    - Enable strict type validation if desired to mirror the workflow.

14. **Add engagement threshold gate**
    - Create an **IF** node named **IF Engagement >= 75**.
    - Connect the true branch of **IF Confidence >= 0.7** → **IF Engagement >= 75**.
    - Add a numeric condition:
      - left value: `{{ $json.engagement_score }}`
      - operation: greater than or equal
      - right value: `75`

15. **Add low-confidence output**
    - Create a **Google Sheets** node named **Append Low Confidence**.
    - Connect the false branch of **IF Confidence >= 0.7** → **Append Low Confidence**.
    - Configure:
      - Operation: `Append`
      - Spreadsheet: your spreadsheet
      - Sheet: `low_confidence_engagement`
      - Mapping mode: auto-map input data
    - Use the same Google Sheets credential.

16. **Add normal analysis output**
    - Create a **Google Sheets** node named **Append Engagement Analysis**.
    - Connect the false branch of **IF Engagement >= 75** → **Append Engagement Analysis**.
    - Configure:
      - Operation: `Append`
      - Sheet: `engagement_analysis`
      - Mapping mode: auto-map

17. **Add top performer output**
    - Create a **Google Sheets** node named **Append Top Performers**.
    - Connect the true branch of **IF Engagement >= 75** → **Append Top Performers**.
    - Configure:
      - Operation: `Append`
      - Sheet: `top_performers`
      - Mapping mode: auto-map

18. **Add email alert**
    - Create a **Gmail** node named **Alert Top Performer**.
    - Connect the true branch of **IF Engagement >= 75** → **Alert Top Performer**.
    - Configure:
      - To:
        - preferably `{{ $json.team_email || 'team@company.com' }}`
      - Subject:
        - include competitor name and engagement score
      - Body:
        - include competitor name
        - engagement score
        - key strengths
        - strategic takeaway
      - Email type: plain text
    - Important recommended improvement:
      - Use `{{ $json.team_email || 'team@company.com' }}` rather than `$('Read Competitor Accounts').first().json.team_email` to avoid sending to the wrong recipient.

19. **Create the iteration loop-back**
    - Connect:
      - **Append Low Confidence** → **Process Items One by One**
      - **Append Engagement Analysis** → **Process Items One by One**
      - **Append Top Performers** → **Process Items One by One**
    - This returns control to the batch iterator for the next competitor row.
    - The Gmail node does not loop back, which is acceptable because **Append Top Performers** already does.

20. **Optionally add sticky notes**
    - Add notes describing:
      - workflow purpose and setup
      - input stage
      - scrape and validation stage
      - AI analysis stage
      - quality gates and outputs

21. **Test with one competitor row first**
    - Add one row to `competitor_accounts` with:
      - a valid Instagram profile URL
      - `competitor_name`
      - `our_industry`
      - `team_email`
    - Run manually and inspect:
      - Bright Data response shape
      - metric calculations
      - AI JSON formatting
      - confidence and score routing

22. **Validate sheet headers**
    - Because append nodes use auto-mapping, create destination sheet headers matching expected output fields, such as:
      - `competitor_name`
      - `follower_count`
      - `follower_growth_signal`
      - `engagement_rate_quality`
      - `content_consistency`
      - `audience_authenticity`
      - `engagement_score`
      - `top_content_types`
      - `posting_frequency`
      - `key_strengths`
      - `key_weaknesses`
      - `strategic_takeaway`
      - `processed_at`
      - `eval.confidence` if you flatten it manually, or keep object/string output if not

23. **Recommended hardening before production**
    - Skip AI analysis when `status != 'ok'`
    - Add retry or polling if Bright Data returns `snapshot_id`
    - Preserve per-item metadata rather than reading `.first()` from upstream nodes
    - Normalize arrays/objects before writing to Sheets
    - Add explicit error logging branch for API/auth failures
    - Add rate limiting if processing large competitor lists

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require Call Workflow / Execute Workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The canvas note says the workflow triggers daily at 8 AM, but the actual Schedule Trigger configuration in the JSON is an hourly interval. Review and correct this before production use. | Workflow configuration consistency |
| The workflow expects a Google Sheet tab named `competitor_accounts` with at least a `url` column. Additional useful columns are `competitor_name`, `our_industry`, and `team_email`. | Source data structure |
| Bright Data authentication is expected as HTTP Header Auth using a Bearer token. | Bright Data setup |
| Approximate operating cost noted in the workflow: `~$0.01–0.03 per profile (Bright Data) + ~$0.005 per analysis (GPT-5.4)`. | Cost estimate from sticky note |
| There is a multi-item data integrity risk because some expressions use `$('Read Competitor Accounts').first()` instead of the current item context. This can cause wrong competitor metadata or wrong email recipient when processing multiple rows. | Recommended fix before production |
| The current Bright Data handling recognizes async `snapshot_id` responses but does not poll for completion. Those cases are effectively treated as incomplete results. | Integration limitation |