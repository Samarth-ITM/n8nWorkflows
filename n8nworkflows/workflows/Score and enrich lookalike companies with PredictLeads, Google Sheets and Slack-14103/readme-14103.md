Score and enrich lookalike companies with PredictLeads, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/score-and-enrich-lookalike-companies-with-predictleads--google-sheets-and-slack-14103


# Score and enrich lookalike companies with PredictLeads, Google Sheets and Slack

# 1. Workflow Overview

This workflow identifies and prioritizes lookalike companies based on a list of best customer domains stored in Google Sheets. It uses PredictLeads to discover similar companies, enriches those companies with external buying and growth signals, computes a composite lead score, and then triggers downstream actions such as Slack alerts, Google Sheets storage, and AI-generated outreach emails via Gmail.

Its main use cases are:

- outbound prospecting based on ideal customer profile similarity
- signal-based lead prioritization
- sales alerting for high-growth or high-score targets
- semi-automated outreach generation

The workflow is organized into the following logical blocks.

## 1.1 Trigger & Input Reception

A manual trigger starts the workflow, then Google Sheets loads a list of customer domains from an input sheet.

## 1.2 Client Iteration & Lookalike Discovery

The workflow loops through each source customer domain, calls PredictLeads to retrieve company data, and extracts related lookalike companies from the API response.

## 1.3 Lookalike Iteration & Signal Enrichment

Each extracted lookalike company is processed individually. PredictLeads is queried for company news events, job openings, and technologies used.

## 1.4 Growth Signal Detection & Slack Alerting

Recent and historical news events are analyzed to detect growth indicators and negative signals. If a company is classified as high-growth, a Slack notification is sent.

## 1.5 Composite Lead Scoring

The workflow combines recency of news, hiring volume, target technology overlap, similarity, and region to produce a 0–100 score.

## 1.6 Qualified Lead Handling: Alerts, Storage, and Outreach

A filter is intended to pass only high-scoring leads. Qualified leads then trigger a Slack alert, are appended to an output Google Sheet, and receive an AI-generated outreach email draft that is sent through Gmail.

Important implementation note: the score filter node is currently **disabled**, which means downstream high-score actions will not execute unless it is re-enabled or replaced.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Input Reception

### Overview

This block starts the workflow manually and reads the list of best client domains from Google Sheets. It provides the source domains that drive all later lookalike discovery and enrichment.

### Nodes Involved

- When clicking 'Execute workflow'
- Read Best Client Domains

### Node Details

#### When clicking 'Execute workflow'

- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for ad hoc execution inside the n8n editor.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none
  - Output: `Read Best Client Domains`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime beyond standard manual execution context.
- **Sub-workflow reference:** None.

#### Read Best Client Domains

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet containing the best customer domains.
- **Configuration choices:**  
  - Uses a specific spreadsheet document
  - Reads from `Sheet1` (`gid=0`)
  - No extra options configured
  - The workflow description expects a `domain` column
- **Key expressions or variables used:** Downstream nodes rely on `$json.domain`.
- **Input and output connections:**  
  - Input: `When clicking 'Execute workflow'`
  - Output: `Loop Clients`
- **Version-specific requirements:** Type version `4.5`; requires a valid Google Sheets credential.
- **Edge cases or potential failure types:**  
  - Missing or invalid Google credentials
  - Wrong spreadsheet access permissions
  - Sheet missing expected `domain` column
  - Empty rows or blank domain values
- **Sub-workflow reference:** None.

---

## 2.2 Client Iteration & Lookalike Discovery

### Overview

This block iterates through every best client domain, retrieves the company from PredictLeads, and extracts related lookalike companies from the API payload. It converts nested API relationships into normalized item records for later enrichment.

### Nodes Involved

- Loop Clients
- Retrieve Company
- Extract Lookalikes

### Node Details

#### Loop Clients

- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over input client rows one batch at a time.
- **Configuration choices:** Default options; effectively processes one item per cycle unless changed.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - Input: `Read Best Client Domains`
  - Output 1: `Retrieve Company`
  - Loop-back input from `Loop Lookalikes` to continue outer iteration
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - If sheet output is empty, no downstream discovery occurs
  - Loop behavior can be confusing if batch settings are changed
- **Sub-workflow reference:** None.

#### Retrieve Company

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Calls PredictLeads to retrieve company data for the current source domain.
- **Configuration choices:**  
  - Operation: `retrieveCompany`
  - Domain input: `={{ $json.domain }}`
- **Key expressions or variables used:**  
  - `$json.domain`
- **Input and output connections:**  
  - Input: `Loop Clients`
  - Output: `Extract Lookalikes`
- **Version-specific requirements:** Type version `1`; requires PredictLeads credential setup.
- **Edge cases or potential failure types:**  
  - Missing/invalid PredictLeads API credential
  - Domain not found in PredictLeads
  - Rate limiting or API unavailability
  - API response may omit `included` or `relationships.lookalike_companies`
- **Sub-workflow reference:** None.

#### Extract Lookalikes

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the PredictLeads company response and emits one normalized item per lookalike company.
- **Configuration choices:** JavaScript code:
  - reads `item.json.data?.[0]`
  - reads `item.json.included`
  - finds `relationships.lookalike_companies.data`
  - matches related IDs inside `included`
  - outputs:
    - `domain`
    - `company_name`
    - `source_domain`
    - `similarity_score`
- **Key expressions or variables used:**  
  Internal JS variables:
  - `mainData`
  - `included`
  - `lookalikeIds`
  - `match`
- **Important logic detail:**  
  `similarity_score` is hardcoded to `0.9`; it is not read from PredictLeads response. This affects the later composite score.
- **Input and output connections:**  
  - Input: `Retrieve Company`
  - Output: `Loop Lookalikes`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing `data` array or malformed response
  - `included` array may not contain matching IDs
  - Empty or invalid domains are skipped silently
  - Hardcoded similarity may over-score all lookalikes
- **Sub-workflow reference:** None.

---

## 2.3 Lookalike Iteration & Signal Enrichment

### Overview

This block iterates through each lookalike company and enriches it with PredictLeads signals: news events, job openings, and technology detections. These outputs feed both growth detection and scoring.

### Nodes Involved

- Loop Lookalikes
- Retrieve Company News Events
- Retrieve Company Job Openings
- Retrieve Technologies

### Node Details

#### Loop Lookalikes

- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through extracted lookalike companies one at a time.
- **Configuration choices:** Default options.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - Input: `Extract Lookalikes`
  - Output 1: `Retrieve Company News Events`
  - Loop completion output returns to `Loop Clients`
  - Receives loop-back from `Write Scored Output`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Loop can stall if the loop-back path never executes
  - Because loop continuation depends on `Write Scored Output`, a disabled or bypassed downstream path can prevent proper completion
- **Sub-workflow reference:** None.

#### Retrieve Company News Events

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Retrieves news events for the current lookalike company.
- **Configuration choices:**  
  - Resource: `newsEvents`
  - Operation: `retrieveCompanyNewsEvents`
  - Domain: `={{ $json.domain }}`
- **Key expressions or variables used:**  
  - `$json.domain`
- **Input and output connections:**  
  - Input: `Loop Lookalikes`
  - Output: `Retrieve Company Job Openings`
  - Output also branches to `Detect Growth Signals`
- **Version-specific requirements:** Type version `1`; requires PredictLeads credentials.
- **Edge cases or potential failure types:**  
  - No news data available
  - API rate limits
  - Domain mismatch or invalid domain
- **Sub-workflow reference:** None.

#### Retrieve Company Job Openings

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Retrieves job openings for the current lookalike.
- **Configuration choices:**  
  - Resource: `jobOpenings`
  - Operation: `retrieveCompanyJobOpenings`
  - Domain expression: `={{ $('Extract Lookalikes').item.json.domain }}`
- **Key expressions or variables used:**  
  - `$('Extract Lookalikes').item.json.domain`
- **Input and output connections:**  
  - Input: `Retrieve Company News Events`
  - Output: `Retrieve Technologies`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Cross-node item linking can fail if execution context changes
  - If `Extract Lookalikes` item resolution does not align with the current batch item, wrong domains may be queried
- **Sub-workflow reference:** None.

#### Retrieve Technologies

- **Type and technical role:** `@predictleads/n8n-nodes-predictleads.predictLeads`  
  Retrieves detected technologies for the current lookalike company.
- **Configuration choices:**  
  - Resource: `technologyDetections`
  - Operation: `retrieveTechnologiesUsedByCompany`
  - Domain expression: `={{ $('Extract Lookalikes').item.json.domain }}`
- **Key expressions or variables used:**  
  - `$('Extract Lookalikes').item.json.domain`
- **Input and output connections:**  
  - Input: `Retrieve Company Job Openings`
  - Output: `Calculate Composite Score`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Same item-linking risk as job openings node
  - Response may contain technology detections without resolvable included technology names
- **Sub-workflow reference:** None.

---

## 2.4 Growth Signal Detection & Slack Alerting

### Overview

This block analyzes company news events to derive growth and risk signals. Companies with sufficient positive signal strength are flagged as high-growth and sent to Slack.

### Nodes Involved

- Detect Growth Signals
- High Growth Detected?
- Send Growth Alert

### Node Details

#### Detect Growth Signals

- **Type and technical role:** `n8n-nodes-base.code`  
  Classifies news event categories into positive and negative growth signals, computes a growth score, and labels high-growth companies.
- **Configuration choices:** JavaScript logic includes:
  - recent window: last 30 days
  - positive categories:
    - `launches`
    - `partners_with`
    - `integrates_with`
    - `acquires`
    - `hires`
    - `invests_into`
    - `signs_new_client`
  - negative categories:
    - `has_issues_with`
    - `files_suit_against`
  - score formula:
    - `recentSignals.length * 2`
    - `+ growthSignals.length`
    - `- negativeEvents.length * 2`
  - high-growth threshold: `growth_score >= 5`
- **Key expressions or variables used:**  
  Internal JS variables:
  - `positiveSignals`
  - `negativeSignals`
  - `growth_signal_count`
  - `recent_signal_count`
  - `negative_signal_count`
  - `growth_score`
  - `is_high_growth`
- **Input and output connections:**  
  - Input: `Retrieve Company News Events`
  - Output: `High Growth Detected?`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - News data may be empty
  - Event categories outside the predefined list are ignored
  - Date parsing issues if `found_at` is malformed
  - Output contains only growth analysis, not the original lookalike record, so context is limited unless rejoined elsewhere
- **Sub-workflow reference:** None.

#### High Growth Detected?

- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the computed `is_high_growth` field is true.
- **Configuration choices:**  
  - Boolean equals `true`
  - Expression: `={{ $json.is_high_growth }}`
- **Key expressions or variables used:**  
  - `$json.is_high_growth`
- **Input and output connections:**  
  - Input: `Detect Growth Signals`
  - True output: `Send Growth Alert`
  - False output: none
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Strict type validation means non-boolean values could fail the condition unexpectedly
- **Sub-workflow reference:** None.

#### Send Growth Alert

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message for companies classified as high-growth.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Sends to a selected channel
  - Text is configured as expression `=...` in the export, meaning actual message content is not visible in the provided JSON
- **Key expressions or variables used:** Slack text expression exists but is not included in readable form.
- **Input and output connections:**  
  - Input: `High Growth Detected?`
  - Output: none
- **Version-specific requirements:** Type version `2.4`; requires Slack OAuth2 credential.
- **Edge cases or potential failure types:**  
  - Missing or expired Slack OAuth token
  - Channel access restrictions
  - Empty or malformed text expression
- **Sub-workflow reference:** None.

---

## 2.5 Composite Lead Scoring

### Overview

This block converts enrichment data into a composite score from 0 to 100. It emphasizes recent news, active hiring, target technology usage, profile similarity, and target geography.

### Nodes Involved

- Calculate Composite Score
- Filter High Scores

### Node Details

#### Calculate Composite Score

- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates data from several upstream nodes and assigns a score.
- **Configuration choices:** JavaScript scoring model:
  - `+30` if at least one news event in the last 30 days
  - `+30` if at least 5 job openings
  - `+20` if technologies match any of:
    - HubSpot
    - Salesforce
    - Marketo
  - `+10` if `similarity_score > 0.7`
  - `+10` if country is one of:
    - United States
    - United Kingdom
    - Canada
- **Key expressions or variables used:**  
  References:
  - `$('Retrieve Company News Events').first().json`
  - `$('Retrieve Company Job Openings').first().json`
  - `$('Retrieve Technologies').first().json`
  - `$('Loop Lookalikes').first().json`
- **Important logic detail:**  
  This node uses `.first()` for all upstream references. In batched or iterative execution, this may incorrectly read the first item from the node instead of the current item, depending on execution context. A safer pattern would usually use `.item`-scoped references.
- **Other logic details:**  
  - Technology names are resolved via `included` objects of type `technology`
  - Region is inferred from the first news event containing `location_data`
- **Input and output connections:**  
  - Input: `Retrieve Technologies`
  - Output: `Filter High Scores`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Missing `included` data can prevent tech name resolution
  - Country inference from news may be absent or inaccurate
  - `.first()` usage may produce mismatched records
  - Scoring can overestimate because similarity is hardcoded upstream to `0.9`
- **Sub-workflow reference:** None.

#### Filter High Scores

- **Type and technical role:** `n8n-nodes-base.if`  
  Intended to keep only leads with a composite score above 70.
- **Configuration choices:**  
  - Condition: `={{ $json.composite_score }} > 70`
  - Strict numeric comparison
- **Key expressions or variables used:**  
  - `$json.composite_score`
- **Input and output connections:**  
  - Input: `Calculate Composite Score`
  - True output: `Send High Score Alert`, `Write Scored Output`, `Build Outreach Prompt`
- **Version-specific requirements:** Type version `2.3`.
- **Critical note:** This node is **disabled** in the workflow export.
- **Edge cases or potential failure types:**  
  - Since disabled, downstream lead handling path will not run
  - If re-enabled, string/number coercion issues could arise if score is not numeric
- **Sub-workflow reference:** None.

---

## 2.6 Qualified Lead Handling: Alerts, Storage, and Outreach

### Overview

This block is meant to execute only for high-scoring leads. It sends a Slack alert, appends scored lead data to Google Sheets, builds an outreach prompt, generates email text via OpenAI over HTTP, and sends the email via Gmail.

### Nodes Involved

- Send High Score Alert
- Write Scored Output
- Build Outreach Prompt
- Generate Outreach Email
- Send Outreach Email

### Node Details

#### Send High Score Alert

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack notification for a high-scoring lead.
- **Configuration choices:**  
  - OAuth2 authentication
  - Fixed selected channel
  - Text is present as `=...` in export, so message content is not visible
- **Key expressions or variables used:** Hidden by export truncation.
- **Input and output connections:**  
  - Input: `Filter High Scores`
  - Output: none
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Slack auth expiration
  - Channel permissions
  - Hidden text expression may reference fields that are absent
- **Sub-workflow reference:** None.

#### Write Scored Output

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends qualified lead records to the `Scored Lookalikes` sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Mapping mode: defined manually
  - Writes:
    - `domain`
    - `company_name`
    - `source_domain`
    - `similarity_score`
    - `news_count`
    - `job_count`
    - `tech_names`
    - `tech_match`
    - `composite_score`
    - `scored_at`
  - Target sheet: `Scored Lookalikes`
- **Key expressions or variables used:** Field-level expressions such as `={{ $json.domain }}`.
- **Input and output connections:**  
  - Input: `Filter High Scores`
  - Output: `Loop Lookalikes` to continue iteration
- **Version-specific requirements:** Type version `4.5`; requires Google Sheets credential with write access.
- **Edge cases or potential failure types:**  
  - Missing output tab
  - Sheet column mismatch
  - Append permission issues
  - Loop continuation depends on this node path
- **Sub-workflow reference:** None.

#### Build Outreach Prompt

- **Type and technical role:** `n8n-nodes-base.code`  
  Creates a structured prompt for OpenAI and derives a recipient email from the company domain.
- **Configuration choices:** JavaScript logic:
  - takes `company_name`, `domain`, `composite_score`
  - builds `recipient_email` as `contact@domain`
  - creates prompt requesting:
    - subject
    - concise body
    - personalized intro
    - growth signal mention
    - CTA
    - max 120 words
- **Key expressions or variables used:**  
  Fields added:
  - `ai_prompt`
  - `recipient_email`
- **Input and output connections:**  
  - Input: `Filter High Scores`
  - Output: `Generate Outreach Email`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Domain may be blank, producing empty recipient email
  - `contact@domain` may not be valid or monitored in the real world
- **Sub-workflow reference:** None.

#### Generate Outreach Email

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls OpenAI Chat Completions API directly to generate outreach text.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.openai.com/v1/chat/completions`
  - JSON body includes:
    - model: `gpt-4o-mini`
    - system message defining role
    - user message from `ai_prompt`
    - `temperature: 0.7`
    - `max_tokens: 500`
  - Headers:
    - `Authorization: Bearer <OPENAI_API_KEY from env>`
    - `Content-Type: application/json`
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.ai_prompt) }}`
  - `={{ $env.OPENAI_API_KEY ? 'Bearer ' + $env.OPENAI_API_KEY : '' }}`
- **Input and output connections:**  
  - Input: `Build Outreach Prompt`
  - Output: `Send Outreach Email`
- **Version-specific requirements:** Type version `4.2`. Requires `OPENAI_API_KEY` in n8n environment variables.
- **Edge cases or potential failure types:**  
  - Missing env variable yields empty Authorization header
  - OpenAI API auth or quota failure
  - Model availability mismatch
  - HTTP non-200 responses
  - Prompt content may exceed limits if expanded later
- **Sub-workflow reference:** None.

#### Send Outreach Email

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated outreach email via Gmail.
- **Configuration choices:**  
  - `sendTo`: `={{ $('Build Outreach Prompt').item.json.recipient_email }}`
  - subject extracted from generated text using regex:
    - `Subject:\s*(.+)`
    - fallback: `Quick idea for your team`
  - body extracted by removing `Subject:` and `Body:` labels
  - email type: `text`
- **Key expressions or variables used:**  
  - `$('Build Outreach Prompt').item.json.recipient_email`
  - `$json.choices[0].message.content`
- **Input and output connections:**  
  - Input: `Generate Outreach Email`
  - Output: none
- **Version-specific requirements:** Type version `2.2`; requires Gmail credential.
- **Edge cases or potential failure types:**  
  - Invalid generated email address
  - Gmail OAuth issues
  - OpenAI response format may not contain expected `choices[0].message.content`
  - Regex parsing may fail if the model does not follow the requested format
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow start |  | Read Best Client Domains | ## Trigger & Input\nManually starts the workflow and loads your best customer domains from Google Sheets. |
| Read Best Client Domains | Google Sheets | Read source client domains from input sheet | When clicking 'Execute workflow' | Loop Clients | ## Trigger & Input\nManually starts the workflow and loads your best customer domains from Google Sheets. |
| Loop Clients | Split In Batches | Iterate through source client domains | Read Best Client Domains; Loop Lookalikes | Retrieve Company | ## Lookalike Discovery\nLoops through each client domain, queries PredictLeads for similar companies, and extracts structured lookalike data. |
| Retrieve Company | PredictLeads | Fetch company data and related lookalikes | Loop Clients | Extract Lookalikes | ## Lookalike Discovery\nLoops through each client domain, queries PredictLeads for similar companies, and extracts structured lookalike data. |
| Extract Lookalikes | Code | Parse PredictLeads relationships into lookalike items | Retrieve Company | Loop Lookalikes | ## Lookalike Discovery\nLoops through each client domain, queries PredictLeads for similar companies, and extracts structured lookalike data. |
| Loop Lookalikes | Split In Batches | Iterate through discovered lookalike companies | Extract Lookalikes; Write Scored Output | Retrieve Company News Events; Loop Clients | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| Retrieve Company News Events | PredictLeads | Get news events for lookalike company | Loop Lookalikes | Retrieve Company Job Openings; Detect Growth Signals | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| Retrieve Company Job Openings | PredictLeads | Get job openings for lookalike company | Retrieve Company News Events | Retrieve Technologies | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| Retrieve Technologies | PredictLeads | Get technology detections for lookalike company | Retrieve Company Job Openings | Calculate Composite Score | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| Detect Growth Signals | Code | Analyze news for positive/negative growth indicators | Retrieve Company News Events | High Growth Detected? | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| High Growth Detected? | If | Branch on high-growth classification | Detect Growth Signals | Send Growth Alert | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| Send Growth Alert | Slack | Send Slack notification for high-growth company | High Growth Detected? |  | ## Signal Enrichment & Growth Detection\nEnriches each lookalike with news, jobs, and tech data. Detects growth patterns and sends Slack alerts for high-growth companies. |
| Calculate Composite Score | Code | Compute 0–100 lead score from all signals | Retrieve Technologies | Filter High Scores | ## Lead Scoring\nCombines news, hiring, tech, similarity, and region signals into a composite score (0-100) and filters high-scoring leads. |
| Filter High Scores | If | Intended score threshold filter above 70 | Calculate Composite Score | Send High Score Alert; Write Scored Output; Build Outreach Prompt | ## Lead Scoring\nCombines news, hiring, tech, similarity, and region signals into a composite score (0-100) and filters high-scoring leads. |
| Send High Score Alert | Slack | Send Slack notification for top-scoring lead | Filter High Scores |  | ## Alerts, Outreach & Storage\nSends Slack alerts for top leads, saves scored results to Google Sheets, and generates AI-powered outreach emails via Gmail. |
| Write Scored Output | Google Sheets | Append qualified lead data to output sheet | Filter High Scores | Loop Lookalikes | ## Alerts, Outreach & Storage\nSends Slack alerts for top leads, saves scored results to Google Sheets, and generates AI-powered outreach emails via Gmail. |
| Build Outreach Prompt | Code | Build OpenAI prompt and recipient email | Filter High Scores | Generate Outreach Email | ## Alerts, Outreach & Storage\nSends Slack alerts for top leads, saves scored results to Google Sheets, and generates AI-powered outreach emails via Gmail. |
| Generate Outreach Email | HTTP Request | Call OpenAI API for email generation | Build Outreach Prompt | Send Outreach Email | ## Alerts, Outreach & Storage\nSends Slack alerts for top leads, saves scored results to Google Sheets, and generates AI-powered outreach emails via Gmail. |
| Send Outreach Email | Gmail | Send generated email via Gmail | Generate Outreach Email |  | ## Alerts, Outreach & Storage\nSends Slack alerts for top leads, saves scored results to Google Sheets, and generates AI-powered outreach emails via Gmail. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Score and enrich lookalike companies with PredictLeads, Google Sheets and Slack`.

2. **Add a Manual Trigger node**.  
   - Node type: `Manual Trigger`
   - Keep default configuration.
   - This is the workflow entry point.

3. **Add a Google Sheets node** named `Read Best Client Domains`.  
   - Node type: `Google Sheets`
   - Credential: connect a Google Sheets account with access to your spreadsheet
   - Operation: read rows
   - Spreadsheet: choose your source file
   - Sheet: choose the tab containing best client domains
   - Ensure the sheet contains a `domain` column
   - Connect: `Manual Trigger -> Read Best Client Domains`

4. **Add a Split In Batches node** named `Loop Clients`.  
   - Use default settings
   - Connect: `Read Best Client Domains -> Loop Clients`

5. **Add a PredictLeads node** named `Retrieve Company`.  
   - Node type: `PredictLeads`
   - Credential: connect your PredictLeads API credential
   - Operation: `retrieveCompany`
   - Domain: `{{$json.domain}}`
   - Connect: `Loop Clients -> Retrieve Company`

6. **Add a Code node** named `Extract Lookalikes`.  
   - Paste logic that:
     - reads the first main company record from `data[0]`
     - reads the `included` array
     - finds `relationships.lookalike_companies.data`
     - matches relationship IDs to `included`
     - returns one item per valid lookalike domain
   - Output fields:
     - `domain`
     - `company_name`
     - `source_domain`
     - `similarity_score`
   - In the provided workflow, `similarity_score` is hardcoded to `0.9`
   - Connect: `Retrieve Company -> Extract Lookalikes`

7. **Add a Split In Batches node** named `Loop Lookalikes`.  
   - Use default settings
   - Connect: `Extract Lookalikes -> Loop Lookalikes`

8. **Add a PredictLeads node** named `Retrieve Company News Events`.  
   - Resource: `newsEvents`
   - Operation: `retrieveCompanyNewsEvents`
   - Domain: `{{$json.domain}}`
   - Connect: `Loop Lookalikes -> Retrieve Company News Events`

9. **Add a PredictLeads node** named `Retrieve Company Job Openings`.  
   - Resource: `jobOpenings`
   - Operation: `retrieveCompanyJobOpenings`
   - Domain: use the current lookalike domain  
     Prefer `{{$json.domain}}` if you keep item context aligned.  
     The exported workflow uses `{{$('Extract Lookalikes').item.json.domain}}`.
   - Connect: `Retrieve Company News Events -> Retrieve Company Job Openings`

10. **Add a PredictLeads node** named `Retrieve Technologies`.  
    - Resource: `technologyDetections`
    - Operation: `retrieveTechnologiesUsedByCompany`
    - Domain: same current lookalike domain logic as above
    - Connect: `Retrieve Company Job Openings -> Retrieve Technologies`

11. **Add a Code node** named `Detect Growth Signals`.  
    - Connect it from `Retrieve Company News Events`
    - Implement logic to:
      - inspect all news events
      - classify positive categories:
        - launches
        - partners_with
        - integrates_with
        - acquires
        - hires
        - invests_into
        - signs_new_client
      - classify negative categories:
        - has_issues_with
        - files_suit_against
      - mark events in the last 30 days as recent
      - compute:
        - `growth_signal_count`
        - `recent_signal_count`
        - `negative_signal_count`
        - `growth_signal_categories`
        - `growth_signal_summary`
        - `growth_score`
        - `is_high_growth`
        - `analyzed_at`
      - use threshold `growth_score >= 5` for `is_high_growth`

12. **Add an IF node** named `High Growth Detected?`.  
    - Condition type: boolean
    - Expression: `{{$json.is_high_growth}}`
    - Operator: equals
    - Value: `true`
    - Connect: `Detect Growth Signals -> High Growth Detected?`

13. **Add a Slack node** named `Send Growth Alert`.  
    - Credential: connect Slack OAuth2
    - Action: send message to channel
    - Choose the target channel
    - Compose a message using fields from growth analysis, for example:
      - company/domain
      - growth score
      - top categories
      - summary
    - Connect the **true** output of `High Growth Detected? -> Send Growth Alert`

14. **Add a Code node** named `Calculate Composite Score`.  
    - Connect: `Retrieve Technologies -> Calculate Composite Score`
    - Implement scoring logic:
      - read news, jobs, technologies, and lookalike profile
      - add `30` if at least one news event in last 30 days
      - add `30` if job openings count is at least `5`
      - add `20` if company uses one of:
        - hubspot
        - salesforce
        - marketo
      - add `10` if `similarity_score > 0.7`
      - infer country from news location data
      - add `10` if country in:
        - United States
        - United Kingdom
        - Canada
    - Return:
      - `domain`
      - `company_name`
      - `source_domain`
      - `similarity_score`
      - `news_count`
      - `job_count`
      - `tech_names`
      - `tech_match`
      - `composite_score`
      - `scored_at`
    - Prefer current-item references rather than `.first()` if rebuilding robustly.

15. **Add an IF node** named `Filter High Scores`.  
    - Condition type: number
    - Expression: `{{$json.composite_score}}`
    - Operator: greater than
    - Value: `70`
    - Connect: `Calculate Composite Score -> Filter High Scores`
    - Important: in the provided workflow this node is disabled. If you want the workflow to actually continue for qualified leads, leave it enabled.

16. **Add a Slack node** named `Send High Score Alert`.  
    - Connect Slack OAuth2 credential
    - Select the alert channel
    - Build a message including:
      - company name
      - domain
      - source domain
      - score
      - jobs/news/tech summary
    - Connect from the **true** output of `Filter High Scores`

17. **Add a Google Sheets node** named `Write Scored Output`.  
    - Credential: same or another Google Sheets credential with write access
    - Operation: `Append`
    - Spreadsheet: same output workbook or another workbook
    - Sheet tab: `Scored Lookalikes`
    - Create this tab in advance
    - Map these columns:
      - `domain`
      - `company_name`
      - `source_domain`
      - `similarity_score`
      - `news_count`
      - `job_count`
      - `tech_names`
      - `tech_match`
      - `composite_score`
      - `scored_at`
    - Connect from the **true** output of `Filter High Scores`

18. **Add a Code node** named `Build Outreach Prompt`.  
    - Connect from the **true** output of `Filter High Scores`
    - Implement logic to:
      - read `company_name`, `domain`, `composite_score`
      - create `recipient_email` as `contact@${domain}`
      - construct an AI prompt requesting a short B2B cold email
      - ask for output in:
        - `Subject: ...`
        - `Body: ...`
      - keep the requested email under 120 words

19. **Add an HTTP Request node** named `Generate Outreach Email`.  
    - Method: `POST`
    - URL: `https://api.openai.com/v1/chat/completions`
    - Send body as JSON
    - Headers:
      - `Authorization`: `Bearer {{$env.OPENAI_API_KEY}}`
      - `Content-Type`: `application/json`
    - JSON body should include:
      - model `gpt-4o-mini`
      - one system message
      - one user message using the prompt from previous node
      - `temperature: 0.7`
      - `max_tokens: 500`
    - Connect: `Build Outreach Prompt -> Generate Outreach Email`

20. **Configure the OpenAI API key** in your n8n environment.  
    - Set `OPENAI_API_KEY` in your n8n runtime environment
    - Example conceptually:
      - Docker env
      - `.env` file
      - hosting platform secret manager
    - Ensure the value is the raw API key, not already prefixed with `Bearer`

21. **Add a Gmail node** named `Send Outreach Email`.  
    - Credential: connect Gmail OAuth2
    - Action: send email
    - To: `{{$('Build Outreach Prompt').item.json.recipient_email}}`
    - Subject: extract from the generated text with regex, fallback to a default
    - Message body: strip `Subject:` and `Body:` labels from the model output
    - Email type: plain text
    - Connect: `Generate Outreach Email -> Send Outreach Email`

22. **Add loop continuation wiring for lookalikes.**  
    - Connect `Write Scored Output -> Loop Lookalikes`
    - This lets the inner batch loop continue after processing each qualified item
    - Note: if only qualified items continue the loop, non-qualified items may interrupt expected loop behavior. A more robust rebuild would add an explicit merge/continue path from both true and false branches.

23. **Add outer loop continuation wiring.**  
    - Connect `Loop Lookalikes` completion output back to `Loop Clients`
    - This allows the workflow to move from one best-client domain to the next after all lookalikes are processed

24. **Create and verify the Google Sheets structure.**
    - Input sheet must include:
      - `domain`
    - Output sheet tab `Scored Lookalikes` should contain columns matching your append mapping

25. **Test with a small dataset first.**
    - Start with 1–2 customer domains
    - Confirm:
      - PredictLeads returns company and lookalike data
      - technologies/news/jobs are available
      - score values are generated
      - Slack alerts arrive
      - Google Sheets appends rows
      - Gmail sends correctly formatted messages

26. **Recommended hardening adjustments if rebuilding for production.**
    - Re-enable `Filter High Scores`
    - Replace `.first()` references in score calculation with current-item-safe references
    - Preserve original context when branching into growth detection
    - Validate recipient email before Gmail send
    - Add error handling branches or retry logic for external APIs
    - Add rate limiting if PredictLeads quotas are strict

### Required Credentials Summary

- **Google Sheets credential**
  - Read access to input sheet
  - Write access to output sheet
- **PredictLeads credential**
  - API access for company, news, jobs, and technologies
- **Slack OAuth2 credential**
  - Permission to post messages in the selected channel
- **Gmail credential**
  - Permission to send outbound email
- **Environment variable**
  - `OPENAI_API_KEY`

### Sub-workflow Setup

This workflow does **not** use any Sub-workflow or Execute Workflow node.  
There are **no sub-workflows to create**.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PredictLeads credential is required for company retrieval, lookalike discovery, news events, job openings, and technology detection. | https://predictleads.com |
| Input Google Sheet must contain a `domain` column listing best customer domains. | Workflow setup note |
| Create a second Google Sheets tab named `Scored Lookalikes` for output. | Workflow setup note |
| Slack is used for both growth alerts and top lead alerts. | Workflow setup note |
| Gmail is used to send AI-generated outreach emails. | Workflow setup note |
| The workflow expects an environment variable named `OPENAI_API_KEY`. | Workflow setup note |
| Default score threshold in the filter logic is 70. | Workflow customization note |
| Default target technologies in score calculation are HubSpot, Salesforce, and Marketo. | Workflow customization note |
| Growth signal categories can be changed in the `Detect Growth Signals` code node. | Workflow customization note |
| Regional scoring bonus can be customized by editing the target regions list. | Workflow customization note |
| Full embedded workflow note: Lookalike Company Enrichment & Lead Scoring. It explains the end-to-end logic: manual trigger, Google Sheets input, PredictLeads lookalikes, enrichment with news/jobs/tech, growth detection, composite scoring, Slack alerts, Google Sheets output, and Gmail outreach. | Included as a top-level sticky note in the workflow canvas |