Analyze Reddit competitor sentiment with Bright Data and GPT-5.4

https://n8nworkflows.xyz/workflows/analyze-reddit-competitor-sentiment-with-bright-data-and-gpt-5-4-14015


# Analyze Reddit competitor sentiment with Bright Data and GPT-5.4

# 1. Workflow Overview

This workflow performs a scheduled competitive intelligence scan on Reddit. Each week, it reads competitor-related Reddit URLs from a Google Sheet, sends those URLs to Bright Data’s Reddit scraping dataset, analyzes the scraped discussion with GPT-5.4, calculates a derived vulnerability index, and then routes the result based on confidence and opportunity score.

Its primary use cases are:

- Monitoring public Reddit sentiment about competitor brands
- Detecting recurring complaints and weakness patterns
- Prioritizing competitors with exploitable weaknesses
- Alerting strategy teams when a brand shows high competitive vulnerability
- Persisting both strong and weak-confidence analyses for later review

## 1.1 Scheduled Input Reception

The workflow starts on a weekly schedule and reads rows from a Google Sheet tab containing competitor brands and Reddit URLs.

## 1.2 Reddit Scraping via Bright Data

Each sheet row is sent to Bright Data’s dataset API to scrape Reddit post/thread content from the provided URL. The response is normalized to distinguish successful, empty, asynchronous, or error cases.

## 1.3 AI Sentiment and Competitive Analysis

The Bright Data output is passed to a LangChain agent powered by GPT-5.4. The model is instructed to return strict JSON containing sentiment distribution, complaints, praises, vulnerability areas, an opportunity score, and a self-evaluation block.

## 1.4 Post-Processing and Derived Scoring

The AI JSON is parsed, merged with the original competitor metadata from the sheet, and enriched with a calculated `vulnerability_index`, `exploitability`, and top `attack_vectors`.

## 1.5 Confidence Gate and Score-Based Routing

Results with `eval.confidence >= 0.7` proceed to a score-based switch:
- High opportunity (`>= 70`) triggers an email alert and appends to a dedicated high-opportunity sheet
- Medium and fallback outputs append to the general sentiment sheet

Results below the confidence threshold are appended to a low-confidence review sheet.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Input Reception

### Overview
This block triggers the workflow once per week and loads the source dataset of competitor brands and Reddit URLs from Google Sheets. It is the only entry point in the workflow.

### Nodes Involved
- Weekly Competitor Sentiment Scan
- Read Competitor Brands Sheet

### Node Details

#### 1. Weekly Competitor Sentiment Scan
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry trigger
- **Configuration choices:** Configured to run every week on day `1` at hour `8`, which corresponds to Monday at 8 AM in the workflow/server timezone
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to `Read Competitor Brands Sheet`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - Timezone mismatch between business expectation and n8n instance timezone
  - Trigger disabled or workflow inactive
- **Sub-workflow reference:** None

#### 2. Read Competitor Brands Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from Google Sheets
- **Configuration choices:** Reads from spreadsheet `YOUR_SPREADSHEET_ID`, tab `competitor_brands`
- **Key expressions or variables used:** None in parameters; downstream nodes expect fields such as:
  - `url`
  - `brand_name`
  - `industry`
- **Input and output connections:** Input from `Weekly Competitor Sentiment Scan`; output to `Scrape Reddit Posts`
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth credentials
- **Edge cases or potential failure types:**
  - Spreadsheet ID placeholder not replaced
  - Missing tab `competitor_brands`
  - OAuth permission issues
  - Empty sheet
  - Missing required columns like `url`, `brand_name`, `industry`
- **Sub-workflow reference:** None

---

## Block 2 — Reddit Scraping and Response Validation

### Overview
This block submits each Reddit URL to Bright Data’s scraping dataset and standardizes the returned payload into a shape suitable for AI analysis. It also flags asynchronous or failed API responses.

### Nodes Involved
- Scrape Reddit Posts
- Validate BD Response

### Node Details

#### 3. Scrape Reddit Posts
- **Type and technical role:** `n8n-nodes-base.httpRequest`; POST request to Bright Data dataset API
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.brightdata.com/datasets/v3/scrape?dataset_id=gd_md75myxy14rihbjksa&format=json`
  - Body sent as JSON
  - Timeout set to `60000` ms
  - Authentication via generic HTTP Header Auth
- **Key expressions or variables used:**
  - Request body:
    - `{{ JSON.stringify({ input: [{ url: $json.url }] }) }}`
  - Depends on source sheet field `url`
- **Input and output connections:** Input from `Read Competitor Brands Sheet`; output to `Validate BD Response`
- **Version-specific requirements:** Type version `4.2`; requires an HTTP Header Auth credential, typically Bearer token for Bright Data
- **Edge cases or potential failure types:**
  - Missing or malformed `url`
  - Bright Data auth failure
  - Dataset ID invalid or unavailable
  - Request timeout for slower scrape jobs
  - API may return synchronous data, an async `snapshot_id`, or an error object
- **Sub-workflow reference:** None

#### 4. Validate BD Response
- **Type and technical role:** `n8n-nodes-base.code`; normalizes Bright Data responses
- **Configuration choices:** Custom JavaScript checks for:
  - Array response: treated as successful synchronous results
  - Empty array: returns `status: 'no_data'`
  - Object with `snapshot_id`: returns `status: 'async_pending'`
  - Object with `error` or `message`: returns `status: 'api_error'`
  - Any other object: passed through as `status: 'ok'`
- **Key expressions or variables used:**
  - Uses `$input.first().json`
  - Adds fields such as `status`, `error`, `snapshot_id`
- **Input and output connections:** Input from `Scrape Reddit Posts`; output to `Analyze Competitor Sentiment`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If Bright Data returns an unexpected payload shape, the node may pass incomplete data forward
  - `status: 'async_pending'` is not specially routed later, so the AI node may receive a non-scraped payload
  - Empty result arrays become synthetic error rows instead of stopping execution
- **Sub-workflow reference:** None

---

## Block 3 — AI Sentiment and Competitive Opportunity Analysis

### Overview
This block sends the scraped Reddit discussion and competitor metadata to a GPT-5.4-powered LangChain agent. The prompt demands strict machine-readable JSON and asks the model to produce both analysis and confidence metadata.

### Nodes Involved
- Analyze Competitor Sentiment
- GPT-5.4 Model

### Node Details

#### 5. Analyze Competitor Sentiment
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent node orchestrating prompt + model call
- **Configuration choices:**
  - Prompt type: defined inline
  - `onError`: continue to error output behavior enabled at node level
  - Main text includes:
    - `Brand: {{ $json.brand_name }}`
    - `Industry: {{ $json.industry }}`
    - serialized current item via `{{ JSON.stringify($json) }}`
  - System message defines required JSON fields:
    - `sentiment_distribution`
    - `overall_sentiment`
    - `negative_ratio`
    - `top_complaints`
    - `top_praises`
    - `vulnerability_areas`
    - `competitive_opportunity_score`
    - `eval`
- **Key expressions or variables used:**
  - `{{ $json.brand_name }}`
  - `{{ $json.industry }}`
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:** Input from `Validate BD Response`; output to `Parse AI Output`
- **Version-specific requirements:** Type version `3`; requires compatible LangChain/OpenAI integration in n8n
- **Edge cases or potential failure types:**
  - Missing `brand_name` or `industry`
  - Model may ignore JSON-only instruction
  - If Bright Data payload is an error or async placeholder, model may still fabricate analysis
  - Rate limits or auth failures from OpenAI
  - Output may land in `output` or another text field depending on agent behavior
- **Sub-workflow reference:** None

#### 6. GPT-5.4 Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend for the agent
- **Configuration choices:**
  - Model: `gpt-5.4`
  - No extra model options set
- **Key expressions or variables used:** None
- **Input and output connections:** Connected as `ai_languageModel` to `Analyze Competitor Sentiment`
- **Version-specific requirements:** Type version `1`; requires OpenAI credentials and support for the specified model name in the connected account/environment
- **Edge cases or potential failure types:**
  - Model name may not exist in all OpenAI accounts or future environments
  - OpenAI quota/rate limit issues
  - Credential misconfiguration
- **Sub-workflow reference:** None

---

## Block 4 — AI Output Parsing and Derived Vulnerability Calculation

### Overview
This block parses the AI response into JSON, merges it with the original row data from Google Sheets, and computes an additional vulnerability score designed for internal prioritization.

### Nodes Involved
- Parse AI Output
- Calculate Vulnerability Index

### Node Details

#### 7. Parse AI Output
- **Type and technical role:** `n8n-nodes-base.code`; parses model output and merges source metadata
- **Configuration choices:**
  - Reads model text from `output` or `text`
  - Removes possible code fences
  - Attempts `JSON.parse`
  - On parse failure, outputs:
    - `error`
    - `parse_error`
    - `raw_preview`
  - Adds `processed_at` timestamp
  - Merges parsed data with the original item from `Read Competitor Brands Sheet`
- **Key expressions or variables used:**
  - `$input.first().json.output`
  - `$input.first().json.text`
  - `$("Read Competitor Brands Sheet").first().json`
  - `new Date().toISOString()`
- **Input and output connections:** Input from `Analyze Competitor Sentiment`; output to `Calculate Vulnerability Index`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Important design caveat: `$("Read Competitor Brands Sheet").first().json` always references the first item from that node, not necessarily the current paired row if multiple items are processed. This can misattribute AI results to the wrong brand in multi-item runs.
  - Parse failure if the model emits non-JSON text
  - Missing `output` and `text` fields yields empty string parse failure
- **Sub-workflow reference:** None

#### 8. Calculate Vulnerability Index
- **Type and technical role:** `n8n-nodes-base.code`; computes a composite score from AI outputs
- **Configuration choices:**
  - `ratioScore = negative_ratio * 40`
  - `areaScore = min(vulnerability_areas.length * 10, 30)`
  - `complaintScore = min(top_complaints.length * 6, 30)`
  - `vulnerability_index = round(ratioScore + areaScore + complaintScore)`
  - `exploitability`:
    - `high` if `>= 70`
    - `medium` if `>= 40`
    - `low` otherwise
  - `attack_vectors = vulnerability_areas.slice(0, 3)`
- **Key expressions or variables used:**
  - `item.negative_ratio || 0`
  - `item.vulnerability_areas || []`
  - `item.top_complaints || []`
- **Input and output connections:** Input from `Parse AI Output`; output to `IF Confidence >= 0.7`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If `negative_ratio` is a string rather than number, JavaScript coercion may still work but is not guaranteed cleanly
  - If AI parse failed, arrays default to empty and score may misleadingly become `0`
  - `vulnerability_index` is independent from `competitive_opportunity_score`, so the two may disagree
- **Sub-workflow reference:** None

---

## Block 5 — Confidence Gate and Output Routing

### Overview
This block decides whether the AI result is trustworthy enough to use and then routes accepted items by opportunity score. High-scoring items generate an email alert and are also written to a dedicated sheet.

### Nodes Involved
- IF Confidence >= 0.7
- Route by Score Level
- Email Opportunity Alert
- Append High Opportunity Brands
- Append Competitor Sentiment
- Append Low Confidence

### Node Details

#### 9. IF Confidence >= 0.7
- **Type and technical role:** `n8n-nodes-base.if`; filters items by AI confidence
- **Configuration choices:**
  - Numeric condition: `{{ $json.eval.confidence }} >= 0.7`
  - `onError`: continueErrorOutput
- **Key expressions or variables used:**
  - `{{ $json.eval.confidence }}`
- **Input and output connections:**
  - Input from `Calculate Vulnerability Index`
  - True output to `Route by Score Level`
  - False output to `Append Low Confidence`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - If `eval` or `eval.confidence` is missing, strict validation may cause expression or type issues
  - Parse-failure records will likely fall into low-confidence or error behavior depending on execution mode
- **Sub-workflow reference:** None

#### 10. Route by Score Level
- **Type and technical role:** `n8n-nodes-base.switch`; routes accepted records by opportunity score
- **Configuration choices:**
  - Output `High` when `competitive_opportunity_score >= 70`
  - Output `Medium` when `competitive_opportunity_score >= 42`
  - Fallback output enabled as `extra`
  - Outputs renamed
- **Key expressions or variables used:**
  - `{{ $json.competitive_opportunity_score }}`
- **Input and output connections:**
  - Input from `IF Confidence >= 0.7` true branch
  - `High` output to both `Email Opportunity Alert` and `Append High Opportunity Brands`
  - `Medium` output to `Append Competitor Sentiment`
  - Fallback output to `Append Competitor Sentiment`
- **Version-specific requirements:** Type version `3.2`
- **Edge cases or potential failure types:**
  - Rule ordering matters: score `>= 70` also satisfies `>= 42`, but the first matching output handles the intended high-priority path
  - There is no explicit separate low-score branch; fallback and medium both write to the same sheet
  - Missing or non-numeric score can route unexpectedly to fallback
- **Sub-workflow reference:** None

#### 11. Email Opportunity Alert
- **Type and technical role:** `n8n-nodes-base.gmail`; sends alert email for high-opportunity competitors
- **Configuration choices:**
  - Recipient: `strategy@company.com`
  - Subject includes brand name and score
  - Plain text body includes:
    - brand
    - industry
    - opportunity score
    - overall sentiment
    - negative ratio
    - joined complaint, vulnerability, and praise lists
- **Key expressions or variables used:**
  - `{{ $json.brand_name }}`
  - `{{ $json.industry }}`
  - `{{ $json.competitive_opportunity_score }}`
  - `{{ $json.overall_sentiment }}`
  - `{{ $json.negative_ratio }}`
  - `{{ $json.top_complaints.join('\n') }}`
  - `{{ $json.vulnerability_areas.join('\n') }}`
  - `{{ $json.top_praises.join('\n') }}`
- **Input and output connections:** Input from `Route by Score Level` high branch; no downstream connection
- **Version-specific requirements:** Type version `2.1`; requires Gmail OAuth
- **Edge cases or potential failure types:**
  - Recipient not updated from placeholder
  - Gmail quota/auth issues
  - If any array fields are missing, `.join()` can fail unless the data is always normalized
- **Sub-workflow reference:** None

#### 12. Append High Opportunity Brands
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends high-opportunity records
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet/tab: `high_opportunity_brands`
  - Mapping mode: auto-map input data
- **Key expressions or variables used:** Auto-maps all available fields
- **Input and output connections:** Input from `Route by Score Level` high branch; no downstream connection
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth
- **Edge cases or potential failure types:**
  - Missing target tab
  - Sheet schema mismatch or unexpected columns
  - Spreadsheet ID placeholder not replaced
- **Sub-workflow reference:** None

#### 13. Append Competitor Sentiment
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends normal-confidence records for medium/fallback opportunities
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet/tab: `competitor_sentiment`
  - Mapping mode: auto-map input data
- **Key expressions or variables used:** Auto-maps all available fields
- **Input and output connections:**
  - Input from `Route by Score Level` medium branch
  - Input from `Route by Score Level` fallback branch
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Same as other Google Sheets append nodes
  - Medium and low accepted scores are mixed in one table, so downstream reporting must distinguish them by score if needed
- **Sub-workflow reference:** None

#### 14. Append Low Confidence
- **Type and technical role:** `n8n-nodes-base.googleSheets`; stores uncertain AI outputs for review
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `YOUR_SPREADSHEET_ID`
  - Sheet/tab: `low_confidence_sentiment`
  - Mapping mode: auto-map input data
- **Key expressions or variables used:** Auto-maps all available fields
- **Input and output connections:** Input from `IF Confidence >= 0.7` false branch; no downstream connection
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Missing target tab
  - Missing confidence field may produce inconsistent review records
- **Sub-workflow reference:** None

---

## Block 6 — Documentation Sticky Notes

### Overview
These nodes are non-executable annotations embedded in the canvas. They provide setup guidance, workflow intent, and functional grouping.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 15. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation
- **Configuration choices:** Large overview note describing workflow behavior, setup steps, and estimated costs
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 16. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual section label
- **Configuration choices:** Labels the input section
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 17. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual section label
- **Configuration choices:** Labels the scraping and analysis section
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 18. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual section label
- **Configuration choices:** Labels the routing and output section
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Competitor Sentiment Scan | Schedule Trigger | Weekly workflow entry point |  | Read Competitor Brands Sheet | ## 1. Data Input\nTriggers weekly on Monday at 8 AM. Reads competitor brand URLs from the 'competitor_brands' sheet. |
| Read Competitor Brands Sheet | Google Sheets | Read competitor rows and Reddit URLs | Weekly Competitor Sentiment Scan | Scrape Reddit Posts | ## 1. Data Input\nTriggers weekly on Monday at 8 AM. Reads competitor brand URLs from the 'competitor_brands' sheet. |
| Scrape Reddit Posts | HTTP Request | Submit Reddit URL to Bright Data scraping dataset | Read Competitor Brands Sheet | Validate BD Response | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| Validate BD Response | Code | Normalize Bright Data response status | Scrape Reddit Posts | Analyze Competitor Sentiment | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| Analyze Competitor Sentiment | LangChain Agent | Generate structured sentiment and vulnerability analysis | Validate BD Response | Parse AI Output | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| GPT-5.4 Model | OpenAI Chat Model | LLM backend for the analysis agent |  | Analyze Competitor Sentiment | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| Parse AI Output | Code | Parse JSON response and merge metadata | Analyze Competitor Sentiment | Calculate Vulnerability Index | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| Calculate Vulnerability Index | Code | Compute derived vulnerability metrics | Parse AI Output | IF Confidence >= 0.7 | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| IF Confidence >= 0.7 | IF | Filter acceptable analyses by AI confidence | Calculate Vulnerability Index | Route by Score Level; Append Low Confidence | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |
| Route by Score Level | Switch | Route accepted results by opportunity score | IF Confidence >= 0.7 | Email Opportunity Alert; Append High Opportunity Brands; Append Competitor Sentiment | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |
| Email Opportunity Alert | Gmail | Send alert for high-opportunity competitors | Route by Score Level |  | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |
| Append High Opportunity Brands | Google Sheets | Store high-opportunity competitors | Route by Score Level |  | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |
| Append Competitor Sentiment | Google Sheets | Store accepted sentiment results | Route by Score Level |  | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |
| Append Low Confidence | Google Sheets | Store low-confidence results for review | IF Confidence >= 0.7 |  | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |
| Sticky Note | Sticky Note | Canvas documentation and setup notes |  |  | ### How it works\nThis workflow tracks Reddit sentiment about competitor brands on a weekly schedule. It reads brand URLs from a Google Sheet, scrapes Reddit posts through the Bright Data Reddit Posts API, and feeds the data to GPT-5.4 for sentiment analysis.\nThis AI calculates sentiment distribution (positive, neutral, negative percentages), identifies top complaints, praises, and vulnerability areas, and assigns a competitive opportunity score from 0-100. A separate code node computes a composite vulnerability index based on negative sentiment ratio, vulnerability areas, and complaint count.\nResults pass through a confidence gate (>= 0.7) to filter unreliable AI outputs. A three-way switch then routes by opportunity score: high-opportunity brands (>= 70) trigger a Gmail alert and go to 'high_opportunity_brands', while medium and low scores write to 'competitor_sentiment'. Low-confidence results land in 'low_confidence_sentiment' for review.\n### Setup\n1. Create a Google Sheet with a tab named 'competitor_brands' and a 'url' column containing Reddit post or thread URLs about each competitor\n2. Add your Bright Data API key as an HTTP Header Auth credential (Bearer token)\n3. Add your OpenAI API key\n4. Connect Google Sheets via OAuth\n5. Connect Gmail via OAuth for alerts\n6. Set the alert recipient email in the 'Email Opportunity Alert' node\nCost: ~$0.01-0.03 per post (Bright Data) + ~$0.005 per analysis (GPT-5.4) |
| Sticky Note1 | Sticky Note | Canvas section label for input |  |  | ## 1. Data Input\nTriggers weekly on Monday at 8 AM. Reads competitor brand URLs from the 'competitor_brands' sheet. |
| Sticky Note2 | Sticky Note | Canvas section label for scrape/analyze |  |  | ## 2. Scrape & Analyze\nScrapes Reddit via Bright Data. GPT-5.4 evaluates sentiment distribution and competitive vulnerabilities. Vulnerability index calculated from negative ratio, areas, and complaints. |
| Sticky Note3 | Sticky Note | Canvas section label for routing/output |  |  | ## 3. Score Routing & Output\nConfidence gate filters unreliable AI output. Three-way switch routes by opportunity score to sheets and Gmail alerts. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**
   - Name it `Weekly Competitor Sentiment Scan`
   - Set it to run every week
   - Choose Monday
   - Set hour to `8`
   - Confirm the workflow timezone matches your business expectation

3. **Prepare Google Sheets source data**
   - Create a spreadsheet
   - Add a tab named `competitor_brands`
   - Add at minimum these columns:
     - `url`
     - `brand_name`
     - `industry`
   - Populate each row with one Reddit URL related to a competitor brand

4. **Add a Google Sheets node to read competitor rows**
   - Name it `Read Competitor Brands Sheet`
   - Connect it after the schedule trigger
   - Select your Google Sheets OAuth credential
   - Set the document ID to your spreadsheet
   - Set the sheet/tab to `competitor_brands`
   - Leave options default unless you need filtering

5. **Add an HTTP Request node for Bright Data**
   - Name it `Scrape Reddit Posts`
   - Connect it after `Read Competitor Brands Sheet`
   - Method: `POST`
   - URL: `https://api.brightdata.com/datasets/v3/scrape?dataset_id=gd_md75myxy14rihbjksa&format=json`
   - Enable body sending
   - Body type/specification: JSON
   - Add JSON body expression:
     - `{{ JSON.stringify({ input: [{ url: $json.url }] }) }}`
   - Set timeout to `60000`
   - Authentication: Generic Credential Type
   - Generic Auth Type: HTTP Header Auth

6. **Create Bright Data credentials**
   - In n8n credentials, create an `HTTP Header Auth` credential
   - Configure the authorization header expected by Bright Data, typically Bearer token based
   - Attach this credential to `Scrape Reddit Posts`

7. **Add a Code node to validate Bright Data output**
   - Name it `Validate BD Response`
   - Connect it after `Scrape Reddit Posts`
   - Paste this logic conceptually:
     - If response is an array:
       - return error with `status: no_data` if empty
       - else return each array item with `status: ok`
     - If object contains `snapshot_id`:
       - return `status: async_pending`
     - If object contains `error` or `message`:
       - return `status: api_error`
     - Else pass through object with `status: ok`
   - This ensures downstream nodes receive a consistent `status`

8. **Add an AI Agent node**
   - Name it `Analyze Competitor Sentiment`
   - Connect it after `Validate BD Response`
   - Set node error behavior to continue on error
   - Use prompt type “define”
   - In the main text prompt, include:
     - brand name from `{{ $json.brand_name }}`
     - industry from `{{ $json.industry }}`
     - full current item with `{{ JSON.stringify($json) }}`
   - In the system instructions, require a raw JSON object only with:
     - `sentiment_distribution`
     - `overall_sentiment`
     - `negative_ratio`
     - `top_complaints`
     - `top_praises`
     - `vulnerability_areas`
     - `competitive_opportunity_score`
     - `eval` containing `confidence`, `reasoning`, `data_quality`, `evidence_count`

9. **Add an OpenAI Chat Model node**
   - Name it `GPT-5.4 Model`
   - Choose the OpenAI credential
   - Set model to `gpt-5.4`
   - Connect this node to the AI Agent through the model connector, not the main execution line

10. **Create OpenAI credentials**
    - Add your OpenAI API key in n8n credentials
    - Ensure the account supports the selected model name
    - If `gpt-5.4` is unavailable, choose a compatible replacement and update documentation/prompt expectations

11. **Add a Code node to parse the AI response**
    - Name it `Parse AI Output`
    - Connect it after `Analyze Competitor Sentiment`
    - Implement logic to:
      - read `output` or `text`
      - strip markdown code fences if present
      - `JSON.parse()` the cleaned string
      - on failure, emit `error`, `parse_error`, and a short `raw_preview`
      - merge parsed data with original source fields
      - add `processed_at` timestamp
    - Important: do **not** use `$("Read Competitor Brands Sheet").first().json` in a production-safe rebuild if multiple items are processed. Instead preserve item lineage directly from current input, or merge upstream fields before the agent call.

12. **Recommended safer rebuild adjustment**
    - Before sending data to the AI, ensure `brand_name`, `industry`, and `url` are already present in the item passed through all nodes
    - Then in `Parse AI Output`, merge parsed data with the current incoming item rather than fetching the first sheet row from another node
    - This avoids cross-item data corruption

13. **Add a Code node for derived scoring**
    - Name it `Calculate Vulnerability Index`
    - Connect it after `Parse AI Output`
    - Compute:
      - `ratioScore = negative_ratio * 40`
      - `areaScore = min(vulnerability_areas.length * 10, 30)`
      - `complaintScore = min(top_complaints.length * 6, 30)`
      - `vulnerability_index = round(ratioScore + areaScore + complaintScore)`
      - `exploitability = high / medium / low`
      - `attack_vectors = first 3 vulnerability areas`

14. **Add an IF node for confidence filtering**
    - Name it `IF Confidence >= 0.7`
    - Connect it after `Calculate Vulnerability Index`
    - Set numeric condition:
      - left value: `{{ $json.eval.confidence }}`
      - operation: greater than or equal
      - right value: `0.7`
    - Enable continue-on-error if desired

15. **Create a low-confidence review sheet**
    - In the spreadsheet, add a tab named `low_confidence_sentiment`

16. **Add a Google Sheets append node for low-confidence outputs**
    - Name it `Append Low Confidence`
    - Connect it to the false output of the IF node
    - Operation: `append`
    - Spreadsheet: your document
    - Sheet: `low_confidence_sentiment`
    - Mapping mode: auto-map input data

17. **Add a Switch node for score routing**
    - Name it `Route by Score Level`
    - Connect it to the true output of the IF node
    - Add rule 1:
      - Output name: `High`
      - Condition: `{{ $json.competitive_opportunity_score }} >= 70`
    - Add rule 2:
      - Output name: `Medium`
      - Condition: `{{ $json.competitive_opportunity_score }} >= 42`
    - Enable fallback output and name it `extra`
    - Keep rule order so high scores are caught first

18. **Create result sheets**
    - Add a tab named `high_opportunity_brands`
    - Add a tab named `competitor_sentiment`

19. **Add a Gmail node for alerts**
    - Name it `Email Opportunity Alert`
    - Connect it to the `High` output of `Route by Score Level`
    - Authenticate with Gmail OAuth
    - Set recipient to your real strategy address
    - Use plain text email
    - Subject example:
      - `Competitive vulnerability: {{ $json.brand_name }} (Opportunity: {{ $json.competitive_opportunity_score }})`
    - Body should include:
      - brand
      - industry
      - opportunity score
      - overall sentiment
      - negative ratio
      - joined complaints
      - joined vulnerability areas
      - joined praises

20. **Create Gmail credentials**
    - Configure Gmail OAuth in n8n
    - Ensure send permissions are granted
    - Test with a personal or internal mailbox first

21. **Add a Google Sheets append node for high-opportunity brands**
    - Name it `Append High Opportunity Brands`
    - Connect it to the `High` output of `Route by Score Level`
    - Operation: `append`
    - Sheet: `high_opportunity_brands`
    - Mapping mode: auto-map

22. **Add a Google Sheets append node for accepted sentiment records**
    - Name it `Append Competitor Sentiment`
    - Connect both:
      - `Medium` output of `Route by Score Level`
      - `extra` fallback output of `Route by Score Level`
    - Operation: `append`
    - Sheet: `competitor_sentiment`
    - Mapping mode: auto-map

23. **Test with sample rows**
    - Include at least one Reddit URL that Bright Data can scrape successfully
    - Verify AI output parses correctly
    - Verify `eval.confidence` exists
    - Verify score routing behaves as intended

24. **Validate destination schema**
    - Because append nodes use auto-map, new JSON keys will create broader row payloads
    - Ensure your sheets are ready to store fields like:
      - `sentiment_distribution`
      - `overall_sentiment`
      - `negative_ratio`
      - `top_complaints`
      - `top_praises`
      - `vulnerability_areas`
      - `competitive_opportunity_score`
      - `eval`
      - `vulnerability_index`
      - `exploitability`
      - `attack_vectors`
      - `processed_at`

25. **Recommended production hardening**
    - Add an explicit branch for `status != ok` after `Validate BD Response`
    - Add retries or polling for Bright Data `snapshot_id` async responses
    - Add schema validation after AI parsing
    - Default missing arrays before Gmail `.join()`
    - Store raw Bright Data and AI payloads for audits
    - Consider splitting medium and low accepted scores into separate output sheets if reporting requires clearer segmentation

26. **Optional canvas documentation**
    - Add sticky notes summarizing:
      - input block
      - scrape/analyze block
      - routing/output block
      - setup prerequisites and cost estimate

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow tracks Reddit sentiment about competitor brands weekly, using Bright Data for scraping and GPT-5.4 for structured analysis. | Canvas documentation |
| Results are filtered by confidence `>= 0.7`; high-opportunity brands `>= 70` trigger Gmail alerts and are written to `high_opportunity_brands`. | Canvas documentation |
| Medium and fallback accepted results are written to `competitor_sentiment`; low-confidence results go to `low_confidence_sentiment`. | Canvas documentation |
| Required setup includes Google Sheets OAuth, Gmail OAuth, OpenAI credentials, and Bright Data HTTP Header Auth. | Canvas documentation |
| Source sheet must include a tab named `competitor_brands` and at least a `url` column; the workflow logic also expects `brand_name` and `industry`. | Implementation note |
| Estimated cost from the note: Bright Data about `$0.01–0.03` per post and GPT-5.4 about `$0.005` per analysis. | Canvas documentation |
| Important implementation caveat: the current `Parse AI Output` node references `$("Read Competitor Brands Sheet").first().json`, which can mismatch rows when multiple items are processed. | Reliability note |