Generate weekly Seed & Series A scouting reports with PredictLeads and OpenAI

https://n8nworkflows.xyz/workflows/generate-weekly-seed---series-a-scouting-reports-with-predictleads-and-openai-14102


# Generate weekly Seed & Series A scouting reports with PredictLeads and OpenAI

# 1. Workflow Overview

This workflow monitors a watchlist of companies stored in Google Sheets, checks each company for newly detected financing events via PredictLeads, filters those events down to recent Seed and Series A rounds, generates a weekly scouting report with OpenAI, and distributes the result by email and Slack.

Primary use cases:
- Venture capital scouting for early-stage rounds
- Competitive intelligence on tracked startups
- Automated weekly funding monitoring for investor teams

The workflow is organized into four logical blocks.

## 1.1 Trigger & Watchlist Intake

A weekly schedule starts the workflow. The workflow then reads a Google Sheets document containing the company watchlist, expected to include company domains used for downstream API lookups.

## 1.2 Financing Event Enrichment & Filtering

Each watchlist entry is processed one by one through a loop. For each company domain, the workflow calls the PredictLeads financing events endpoint and filters the results to retain only Seed and Series A rounds dated within the last 7 days.

## 1.3 Result Aggregation & AI Report Generation

All filtered events from the loop are consolidated into one structured payload. That payload is sent to OpenAI Chat Completions, which returns a formatted weekly scouting report in natural language plus a markdown-style summary table.

## 1.4 Delivery & Notification

The generated report is sent by Gmail as the main deliverable. A Slack webhook message is also posted to notify stakeholders that the weekly report is ready and summarize the outcome.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Watchlist Intake

### Overview
This block launches the workflow on a weekly cadence and loads the watchlist source data from Google Sheets. It establishes the set of company domains that will be iterated over in the enrichment stage.

### Nodes Involved
- ⏰ Weekly Schedule
- 📋 Read Watchlist

### Node Details

#### ⏰ Weekly Schedule
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Configured with a weekly interval rule. The sticky note states it is intended to run every Monday, but the actual JSON only specifies a weekly interval and does not explicitly define weekday/time.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `📋 Read Watchlist`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Schedule may not run on Monday unless weekday/time are explicitly set in the node UI.
  - Timezone behavior depends on workflow/server settings.
- **Sub-workflow reference:** None.

#### 📋 Read Watchlist
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a Google Sheet containing the company watchlist.
- **Configuration choices:**
  - Uses a Google Sheets OAuth2 credential.
  - Targets a specific spreadsheet ID placeholder: `YOUR_GOOGLE_SHEET_ID_07`.
  - Reads sheet `gid=0`, cached as `Watchlist`.
  - No extra read options are configured.
- **Key expressions or variables used:** None directly in parameters. Downstream nodes expect a field named `domain` in each returned row.
- **Input and output connections:** Input from `⏰ Weekly Schedule`; output to `🔄 Loop Companies`.
- **Version-specific requirements:** Type version `4.5`.
- **Edge cases or potential failure types:**
  - OAuth credential not configured or expired.
  - Spreadsheet ID invalid or inaccessible.
  - Wrong sheet selected.
  - Rows missing a `domain` column will cause downstream PredictLeads calls to fail or hit malformed URLs.
  - Empty sheet results in no useful downstream processing.
- **Sub-workflow reference:** None.

---

## 2.2 Financing Event Enrichment & Filtering

### Overview
This block iterates through each company in the watchlist, fetches funding events from PredictLeads, and filters the results to recent Seed and Series A rounds only. It is the core enrichment logic of the workflow.

### Nodes Involved
- 🔄 Loop Companies
- 🔍 Fetch Financing Events
- ⚙️ Filter Seed & Series A

### Node Details

#### 🔄 Loop Companies
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates over watchlist rows item by item.
- **Configuration choices:** Default options only; batch size is not explicitly set in the JSON.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from `📋 Read Watchlist`.
  - One output path goes to `🔍 Fetch Financing Events` for per-company processing.
  - Another output path goes to `⚙️ Aggregate Results` to emit the post-loop combined stream.
  - Receives loopback input from `⚙️ Filter Seed & Series A`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If batch behavior is misunderstood, rebuilders may connect it incorrectly.
  - Empty upstream input may skip enrichment entirely.
  - Large watchlists may increase runtime and external API usage.
- **Sub-workflow reference:** None.

#### 🔍 Fetch Financing Events
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls PredictLeads financing events API for each domain.
- **Configuration choices:**
  - URL is dynamically built from the incoming item:
    `https://predictleads.com/api/v3/companies/{{ $json.domain }}/financing_events`
  - Sends headers:
    - `X-Api-Key`
    - `X-Api-Token`
    - `Content-Type: application/json`
  - Uses hardcoded placeholders rather than n8n credentials.
- **Key expressions or variables used:**
  - `{{ $json.domain }}` in the URL.
- **Input and output connections:** Input from `🔄 Loop Companies`; output to `⚙️ Filter Seed & Series A`.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid `domain` field produces an invalid endpoint.
  - PredictLeads authentication errors due to invalid API key/token.
  - 404 or empty response for unknown companies.
  - Rate limiting if the watchlist is large.
  - Network timeout or API service interruption.
  - Response schema may differ from what the code node expects.
- **Sub-workflow reference:** None.

#### ⚙️ Filter Seed & Series A
- **Type and technical role:** `n8n-nodes-base.code`; parses PredictLeads responses and retains only relevant events.
- **Configuration choices:**
  - JavaScript code reads all input items with `$input.all()`.
  - Extracts events from either `item.json.data` or `item.json.financing_events`.
  - Extracts company domain from `item.json.domain` or `item.json.company_domain`.
  - Keeps events where:
    - category contains `seed`, `series_a`, or `series a`
    - event date is within the last 7 days
  - Maps output fields:
    - `domain`
    - `company_name`
    - `round_type`
    - `amount`
    - `currency`
    - `date`
    - `investors`
  - If no qualifying events exist, returns one marker item with `_no_results: true`.
- **Key expressions or variables used:**
  - Internal JS variables:
    - `sevenDaysAgo`
    - `event.attributes?.category`
    - `event.attributes?.date`
    - `event.attributes?.investors`
  - Fallback logic across potential response shapes.
- **Input and output connections:**
  - Input from `🔍 Fetch Financing Events`.
  - Output loops back to `🔄 Loop Companies`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the HTTP node does not preserve the input domain field, `domain` may become `unknown`.
  - Date parsing may behave unexpectedly for invalid or non-ISO dates.
  - Category naming variations beyond `seed` and `series a` may be missed.
  - Investors may not be an array; `.join(', ')` would fail if a non-array object is returned.
  - `_no_results` markers are emitted once per company with no match, increasing item count downstream.
- **Sub-workflow reference:** None.

---

## 2.3 Result Aggregation & AI Report Generation

### Overview
This block consolidates all qualifying financing events into a single reporting payload and submits that payload to OpenAI for narrative report generation. It transforms item-level enrichment results into a stakeholder-friendly weekly report.

### Nodes Involved
- ⚙️ Aggregate Results
- 🤖 Generate Scouting Report

### Node Details

#### ⚙️ Aggregate Results
- **Type and technical role:** `n8n-nodes-base.code`; merges filtered event items into one structured summary object.
- **Configuration choices:**
  - Reads all incoming items with `$input.all()`.
  - Excludes items containing `_no_results`.
  - Produces one output item containing:
    - `summary`
    - `events`
    - `count`
  - If no events exist, returns:
    - summary text stating no new Seed or Series A rounds were found
    - empty `events` array
    - `count: 0`
- **Key expressions or variables used:**
  - Internal JS variables:
    - `allEvents`
    - `item.json._no_results`
- **Input and output connections:**
  - Input from `🔄 Loop Companies` after loop completion.
  - Output to `🤖 Generate Scouting Report`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If loop wiring is changed incorrectly, this node may aggregate prematurely.
  - High event volume could create a large payload for OpenAI.
  - No deduplication is performed; duplicate events from upstream will remain.
- **Sub-workflow reference:** None.

#### 🤖 Generate Scouting Report
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a Chat Completions request to OpenAI.
- **Configuration choices:**
  - POST request to `https://api.openai.com/v1/chat/completions`
  - JSON body includes:
    - `model: gpt-4o-mini`
    - system message instructing the model to act as a VC analyst
    - user message containing workflow summary and serialized events
    - `temperature: 0.4`
    - `max_tokens: 2000`
  - Uses header-based auth with:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $json.summary }}`
  - `{{ JSON.stringify($json.events) }}`
- **Input and output connections:**
  - Input from `⚙️ Aggregate Results`.
  - Output to both `📧 Send Report Email` and `💬 Slack Summary`.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid OpenAI API key.
  - Model availability or account access restrictions.
  - Large `events` arrays may exceed token limits.
  - Output structure is assumed downstream to include `choices[0].message.content`.
  - API schema changes could break parsing.
- **Sub-workflow reference:** None.

---

## 2.4 Delivery & Notification

### Overview
This block distributes the generated report via Gmail and posts a Slack notification. It separates full-content delivery from lightweight team alerting.

### Nodes Involved
- 📧 Send Report Email
- 💬 Slack Summary

### Node Details

#### 📧 Send Report Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the full AI-generated report by email.
- **Configuration choices:**
  - Uses Gmail OAuth2 credentials.
  - Sends to `user@example.com`.
  - Subject includes current date via expression:
    `Weekly Seed/Series A Scouting Report - YYYY-MM-DD`
  - Email body is HTML and inserts:
    - heading
    - OpenAI response content from `$json.choices[0].message.content`
    - footer noting automatic generation by n8n + PredictLeads
- **Key expressions or variables used:**
  - `{{ $json.choices[0].message.content }}`
  - `{{ new Date().toISOString().split('T')[0] }}`
- **Input and output connections:** Input from `🤖 Generate Scouting Report`; no downstream nodes.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Gmail OAuth not connected or expired.
  - Recipient address placeholder not changed.
  - HTML may render markdown table imperfectly because the model output is inserted into a `<p>` tag rather than converted to HTML.
  - If OpenAI returns an error payload instead of `choices`, expression evaluation fails.
- **Sub-workflow reference:** None.

#### 💬 Slack Summary
- **Type and technical role:** `n8n-nodes-base.httpRequest`; posts a message to a Slack incoming webhook.
- **Configuration choices:**
  - POST request to a Slack webhook placeholder URL.
  - Sends JSON body with a simple text notification.
  - Message includes a static header and the aggregate summary fetched by node reference.
  - Header `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{ $('⚙️ Aggregate Results').item.json.summary }}`
- **Input and output connections:** Input from `🤖 Generate Scouting Report`; no downstream nodes.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid or revoked Slack webhook.
  - Workspace/channel restrictions.
  - If `⚙️ Aggregate Results` did not execute as expected, expression resolution may fail.
  - Slack webhook errors may not be obvious unless response handling is inspected.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| About This Workflow | Sticky Note | Workspace documentation note |  |  | ABOUT THIS WORKFLOW<br>Monitors your company watchlist for new Seed and Series A funding rounds and delivers a formatted weekly scouting report via email and Slack.<br>Setup: Google Sheet with company domains, PredictLeads API, OpenAI API key, Gmail OAuth2, Slack webhook.<br>Use case: A VC analyst tracks 200 companies. Every Monday, this workflow checks for new funding rounds and sends a clean report with company, round type, amount, and investors.<br>PredictLeads API: https://predictleads.com<br>Questions: https://www.linkedin.com/in/yaronbeen |
| 📌 TRIGGER & INPUT | Sticky Note | Block annotation |  |  | ## 1️⃣ Trigger & Watchlist Source<br>**Nodes:** ⏰ Weekly Schedule → 📋 Read Watchlist<br>**Description:** The workflow begins with a weekly scheduled trigger that runs automatically every Monday. It reads a list of company domains from a Google Sheets watchlist, which represents the companies you want to monitor for funding activity. This sheet acts as the source dataset for the scouting workflow. Each domain from the list will be checked to determine whether the company has recently raised funding. |
| 🔍 ENRICHMENT | Sticky Note | Block annotation |  |  | ## 2️⃣ Financing Event Enrichment<br>**Nodes:** 🔄 Loop Companies → 🔍 Fetch Financing Events → ⚙️ Filter Seed & Series A<br>**Description:** Each company from the watchlist is processed individually in a loop. The workflow calls the PredictLeads Financing Events API to retrieve recent funding events for that company. After retrieving the data, a code node filters the results to keep only Seed and Series A funding rounds that occurred within the last 7 days. This ensures the workflow focuses only on early-stage funding activity, which is typically most relevant for scouting and market monitoring. |
| ⚙️ PROCESSING | Sticky Note | Block annotation |  |  | ## 3️⃣ Data Processing & AI Report Generation<br>**Nodes:** ⚙️ Aggregate Results → 🤖 Generate Scouting Report<br>**Description:** All filtered funding events from the workflow run are aggregated into a single structured dataset. This dataset is then sent to OpenAI, which generates a formatted weekly venture scouting report. The report includes a summary of discovered funding rounds along with a structured markdown table showing: Company, Round Type, Funding Amount, Date, Key Investors. The AI also produces a short analysis highlighting key patterns or trends in the funding activity. |
| 🤖 AI REPORT | Sticky Note | Block annotation |  |  | ## 4️⃣ Report Delivery & Notifications<br>**Nodes:** 📧 Send Report Email → 💬 Slack Summary<br>**Description:** Once the AI-generated scouting report is ready, it is delivered through two channels. The full report is sent via Gmail, allowing the team to review the complete analysis and funding details. At the same time, a Slack notification is posted to alert the team that the weekly scouting report is available. This ensures the insights reach stakeholders quickly while keeping the workflow fully automated. |
| ⏰ Weekly Schedule | Schedule Trigger | Weekly workflow trigger |  | 📋 Read Watchlist | ## 1️⃣ Trigger & Watchlist Source<br>**Nodes:** ⏰ Weekly Schedule → 📋 Read Watchlist<br>**Description:** The workflow begins with a weekly scheduled trigger that runs automatically every Monday. It reads a list of company domains from a Google Sheets watchlist, which represents the companies you want to monitor for funding activity. This sheet acts as the source dataset for the scouting workflow. Each domain from the list will be checked to determine whether the company has recently raised funding. |
| 📋 Read Watchlist | Google Sheets | Reads company domains from watchlist sheet | ⏰ Weekly Schedule | 🔄 Loop Companies | ## 1️⃣ Trigger & Watchlist Source<br>**Nodes:** ⏰ Weekly Schedule → 📋 Read Watchlist<br>**Description:** The workflow begins with a weekly scheduled trigger that runs automatically every Monday. It reads a list of company domains from a Google Sheets watchlist, which represents the companies you want to monitor for funding activity. This sheet acts as the source dataset for the scouting workflow. Each domain from the list will be checked to determine whether the company has recently raised funding. |
| 🔄 Loop Companies | Split In Batches | Iterates through watchlist companies | 📋 Read Watchlist, ⚙️ Filter Seed & Series A | 🔍 Fetch Financing Events, ⚙️ Aggregate Results | ## 2️⃣ Financing Event Enrichment<br>**Nodes:** 🔄 Loop Companies → 🔍 Fetch Financing Events → ⚙️ Filter Seed & Series A<br>**Description:** Each company from the watchlist is processed individually in a loop. The workflow calls the PredictLeads Financing Events API to retrieve recent funding events for that company. After retrieving the data, a code node filters the results to keep only Seed and Series A funding rounds that occurred within the last 7 days. This ensures the workflow focuses only on early-stage funding activity, which is typically most relevant for scouting and market monitoring. |
| 🔍 Fetch Financing Events | HTTP Request | Calls PredictLeads financing events API per domain | 🔄 Loop Companies | ⚙️ Filter Seed & Series A | ## 2️⃣ Financing Event Enrichment<br>**Nodes:** 🔄 Loop Companies → 🔍 Fetch Financing Events → ⚙️ Filter Seed & Series A<br>**Description:** Each company from the watchlist is processed individually in a loop. The workflow calls the PredictLeads Financing Events API to retrieve recent funding events for that company. After retrieving the data, a code node filters the results to keep only Seed and Series A funding rounds that occurred within the last 7 days. This ensures the workflow focuses only on early-stage funding activity, which is typically most relevant for scouting and market monitoring. |
| ⚙️ Filter Seed & Series A | Code | Filters recent Seed and Series A events | 🔍 Fetch Financing Events | 🔄 Loop Companies | ## 2️⃣ Financing Event Enrichment<br>**Nodes:** 🔄 Loop Companies → 🔍 Fetch Financing Events → ⚙️ Filter Seed & Series A<br>**Description:** Each company from the watchlist is processed individually in a loop. The workflow calls the PredictLeads Financing Events API to retrieve recent funding events for that company. After retrieving the data, a code node filters the results to keep only Seed and Series A funding rounds that occurred within the last 7 days. This ensures the workflow focuses only on early-stage funding activity, which is typically most relevant for scouting and market monitoring. |
| ⚙️ Aggregate Results | Code | Aggregates all qualifying events into one summary payload | 🔄 Loop Companies | 🤖 Generate Scouting Report | ## 3️⃣ Data Processing & AI Report Generation<br>**Nodes:** ⚙️ Aggregate Results → 🤖 Generate Scouting Report<br>**Description:** All filtered funding events from the workflow run are aggregated into a single structured dataset. This dataset is then sent to OpenAI, which generates a formatted weekly venture scouting report. The report includes a summary of discovered funding rounds along with a structured markdown table showing: Company, Round Type, Funding Amount, Date, Key Investors. The AI also produces a short analysis highlighting key patterns or trends in the funding activity. |
| 🤖 Generate Scouting Report | HTTP Request | Sends aggregated events to OpenAI for report generation | ⚙️ Aggregate Results | 📧 Send Report Email, 💬 Slack Summary | ## 3️⃣ Data Processing & AI Report Generation<br>**Nodes:** ⚙️ Aggregate Results → 🤖 Generate Scouting Report<br>**Description:** All filtered funding events from the workflow run are aggregated into a single structured dataset. This dataset is then sent to OpenAI, which generates a formatted weekly venture scouting report. The report includes a summary of discovered funding rounds along with a structured markdown table showing: Company, Round Type, Funding Amount, Date, Key Investors. The AI also produces a short analysis highlighting key patterns or trends in the funding activity. |
| 📧 Send Report Email | Gmail | Sends full scouting report via email | 🤖 Generate Scouting Report |  | ## 4️⃣ Report Delivery & Notifications<br>**Nodes:** 📧 Send Report Email → 💬 Slack Summary<br>**Description:** Once the AI-generated scouting report is ready, it is delivered through two channels. The full report is sent via Gmail, allowing the team to review the complete analysis and funding details. At the same time, a Slack notification is posted to alert the team that the weekly scouting report is available. This ensures the insights reach stakeholders quickly while keeping the workflow fully automated. |
| 💬 Slack Summary | HTTP Request | Posts a Slack webhook notification | 🤖 Generate Scouting Report |  | ## 4️⃣ Report Delivery & Notifications<br>**Nodes:** 📧 Send Report Email → 💬 Slack Summary<br>**Description:** Once the AI-generated scouting report is ready, it is delivered through two channels. The full report is sent via Gmail, allowing the team to review the complete analysis and funding details. At the same time, a Slack notification is posted to alert the team that the weekly scouting report is available. This ensures the insights reach stakeholders quickly while keeping the workflow fully automated. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Seed & Series A Funding Enrichment & Weekly Scouting Report with PredictLeads`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `⏰ Weekly Schedule`
   - Configure it to run weekly.
   - If you want to match the description precisely, set it to every Monday at your preferred time.
   - Confirm the workflow timezone in n8n settings.

3. **Prepare the Google Sheet watchlist**
   - Create a Google Sheet with at least one sheet tab.
   - Add a header row including a `domain` column.
   - Populate it with company domains such as:
     - `example.com`
     - `startup.io`
   - Share the sheet with the Google account used by n8n if necessary.

4. **Add a Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `📋 Read Watchlist`
   - Connect `⏰ Weekly Schedule` → `📋 Read Watchlist`
   - Configure Google Sheets OAuth2 credentials.
   - Select the target spreadsheet.
   - Select the watchlist sheet, equivalent to `gid=0` in the source workflow.
   - Operation should read rows from the selected sheet.
   - No special options are required.

5. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name: `🔄 Loop Companies`
   - Connect `📋 Read Watchlist` → `🔄 Loop Companies`
   - Leave default options unless you want to specify batch size.
   - Standard one-item iteration is the intended pattern here.

6. **Add the PredictLeads HTTP Request node**
   - Node type: `HTTP Request`
   - Name: `🔍 Fetch Financing Events`
   - Connect `🔄 Loop Companies` → `🔍 Fetch Financing Events`
   - Configure:
     - Method: `GET`
     - URL:
       `https://predictleads.com/api/v3/companies/{{$json.domain}}/financing_events`
   - Enable headers.
   - Add headers:
     - `X-Api-Key` = your PredictLeads API key
     - `X-Api-Token` = your PredictLeads API token
     - `Content-Type` = `application/json`
   - Prefer using n8n credentials or environment variables instead of hardcoding secrets.

7. **Add the filtering Code node**
   - Node type: `Code`
   - Name: `⚙️ Filter Seed & Series A`
   - Connect `🔍 Fetch Financing Events` → `⚙️ Filter Seed & Series A`
   - Paste logic equivalent to:
     - Read all incoming API items
     - Extract `data` or `financing_events`
     - Keep only events whose category contains Seed or Series A
     - Keep only events whose date is within the last 7 days
     - Return normalized fields:
       - `domain`
       - `company_name`
       - `round_type`
       - `amount`
       - `currency`
       - `date`
       - `investors`
     - If no matches exist, return one item containing `_no_results: true`
   - Use the same code structure as the source workflow.

8. **Close the loop**
   - Connect `⚙️ Filter Seed & Series A` back to `🔄 Loop Companies`
   - This loopback is required so the Split In Batches node continues processing the next company.

9. **Add the aggregation Code node**
   - Node type: `Code`
   - Name: `⚙️ Aggregate Results`
   - Connect the completion output of `🔄 Loop Companies` → `⚙️ Aggregate Results`
   - Configure code to:
     - read all incoming items
     - discard `_no_results` items
     - create a single output item with:
       - `summary`
       - `events`
       - `count`
     - if no events remain, emit:
       - summary saying no new Seed or Series A rounds were found
       - `events: []`
       - `count: 0`

10. **Add the OpenAI HTTP Request node**
    - Node type: `HTTP Request`
    - Name: `🤖 Generate Scouting Report`
    - Connect `⚙️ Aggregate Results` → `🤖 Generate Scouting Report`
    - Configure:
      - Method: `POST`
      - URL: `https://api.openai.com/v1/chat/completions`
      - Body type: JSON
      - Headers:
        - `Authorization` = `Bearer YOUR_OPENAI_API_KEY`
        - `Content-Type` = `application/json`
    - JSON body should include:
      - model: `gpt-4o-mini`
      - a system message instructing the model to act as a venture capital analyst
      - a user message passing:
        - `{{$json.summary}}`
        - `{{JSON.stringify($json.events)}}`
      - `temperature: 0.4`
      - `max_tokens: 2000`
    - Ensure the response format is standard OpenAI Chat Completions JSON.

11. **Add the Gmail node**
    - Node type: `Gmail`
    - Name: `📧 Send Report Email`
    - Connect `🤖 Generate Scouting Report` → `📧 Send Report Email`
    - Configure Gmail OAuth2 credentials.
    - Set recipient email address.
    - Subject:
      `Weekly Seed/Series A Scouting Report - {{ new Date().toISOString().split('T')[0] }}`
    - Message body as HTML:
      - title heading
      - `{{ $json.choices[0].message.content }}`
      - footer text noting it was generated automatically
    - Note: if you want better rendering, consider converting markdown to HTML before sending.

12. **Add the Slack notification node**
    - Node type: `HTTP Request`
    - Name: `💬 Slack Summary`
    - Connect `🤖 Generate Scouting Report` → `💬 Slack Summary`
    - Configure:
      - Method: `POST`
      - URL: your Slack incoming webhook URL
      - Body type: JSON
      - Header:
        - `Content-Type: application/json`
    - JSON body text should include:
      - a notification header
      - the summary from `⚙️ Aggregate Results`
    - Use this expression pattern:
      - `{{ $('⚙️ Aggregate Results').item.json.summary }}`

13. **Optionally add sticky notes**
    - Add descriptive sticky notes for:
      - overall workflow purpose
      - trigger/input block
      - enrichment block
      - processing block
      - report delivery block

14. **Configure credentials**
    - Required external access:
      - Google Sheets OAuth2
      - Gmail OAuth2
      - PredictLeads API key and token
      - OpenAI API key
      - Slack incoming webhook
    - Recommended practice:
      - store secrets in n8n credentials or environment variables
      - avoid literal placeholders in production

15. **Test the watchlist input**
    - Execute `📋 Read Watchlist`
    - Verify each row includes `domain`
    - Confirm domains are valid and consistently formatted

16. **Test the PredictLeads call**
    - Execute `🔍 Fetch Financing Events` with a known domain
    - Verify the response contains financing event data
    - Adjust parsing logic if the API response shape differs

17. **Test the filter logic**
    - Execute `⚙️ Filter Seed & Series A`
    - Confirm that:
      - recent Seed and Series A rounds pass through
      - old or irrelevant rounds are excluded
      - no-match cases return `_no_results: true`

18. **Test the aggregate step**
    - Execute the loop with sample data
    - Verify `⚙️ Aggregate Results` returns one item only
    - Confirm summary text and count are correct

19. **Test AI generation**
    - Execute `🤖 Generate Scouting Report`
    - Confirm the response includes:
      - `choices[0].message.content`
    - Check prompt quality and formatting of the returned report

20. **Test delivery nodes**
    - Send a test Gmail report
    - Post a test Slack notification
    - Verify recipients, formatting, and webhook permissions

21. **Activate the workflow**
    - Once all credentials and tests are validated, activate the workflow.
    - Confirm the weekly schedule is correct and aligned with your timezone.

## Sub-workflow setup
- This workflow does **not** invoke any sub-workflows.
- There are no `Execute Workflow` nodes.
- There is a single entry point: `⏰ Weekly Schedule`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| PredictLeads website | https://predictleads.com |
| Questions / creator contact | https://www.linkedin.com/in/yaronbeen |
| Required services mentioned in the workflow notes: Google Sheet with company domains, PredictLeads API, OpenAI API key, Gmail OAuth2, Slack webhook | Workflow setup prerequisites |
| Example use case mentioned: a VC analyst tracks 200 companies and receives a Monday scouting report with company, round type, amount, and investors | Workflow purpose / operating context |

## Additional implementation notes
- The workflow description says it runs every Monday, but the current Schedule Trigger JSON only guarantees a weekly interval. Explicit weekday/time configuration should be added if Monday execution is required.
- The email body wraps AI output inside a single HTML paragraph tag. If OpenAI returns markdown tables, rendering in email clients may be poor.
- The OpenAI and PredictLeads authentication details are hardcoded as placeholders in HTTP headers; in production, use secure credential storage.
- The filter logic assumes investor data is an array and response fields may exist under either `attributes` or top-level keys.
- If preserving the original domain is important, ensure the PredictLeads response still carries it or merge original input fields before filtering.