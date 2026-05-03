Generate personalized trip recommendations with Claude AI and Google Sheets

https://n8nworkflows.xyz/workflows/generate-personalized-trip-recommendations-with-claude-ai-and-google-sheets-14264


# Generate personalized trip recommendations with Claude AI and Google Sheets

# 1. Workflow Overview

This workflow implements a personalized travel recommendation engine that combines structured traveler data from Google Sheets with Claude AI to generate destination suggestions. It supports two main execution paths:

- **On-demand recommendation generation** via a webhook
- **Scheduled trend and batch analysis** via a daily cron trigger

Its purpose is to recommend the next trip for a user based on travel history, preferences, constraints, and inferred behavioral patterns, then optionally enrich and deliver the result.

## 1.1 Request Intake & Validation

This block receives a POST request containing trip preferences and constraints, validates the payload, normalizes data, and builds a canonical `tripRequest` object for downstream processing.

## 1.2 Travel History Retrieval & Pattern Analysis

This block reads travel history and profile data from Google Sheets, filters records for the current user, and derives behavioral signals such as preferred climate, average budget, destination type, recent trips, and regional habits.

## 1.3 AI Recommendation Generation

This block sends the analyzed traveler context to Claude Sonnet 4 through an AI Agent node and requests a strict JSON recommendation payload containing 3–5 destination suggestions and travel strategy metadata.

## 1.4 Enrichment, Persistence & Response Delivery

This block parses Claude’s output, creates a normalized recommendation record, optionally enriches it with weather data, stores it in Google Sheets, sends an email, and returns a JSON API response to the original webhook caller.

## 1.5 Scheduled Batch Trend Analysis

This separate branch runs daily, reads all user profiles, determines which users need refreshed recommendations, aggregates travel preference trends, sends a report by email, and logs analytics to Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Request Intake & Validation

### Overview

This block receives the travel recommendation request and converts arbitrary incoming payload data into a validated, normalized structure. It is the entry point for the real-time recommendation path and prevents downstream failures caused by missing required fields or malformed email addresses.

### Nodes Involved

- Receive Trip Suggestion Request
- Validate and Normalize Request

### Node Details

#### Receive Trip Suggestion Request

- **Type and technical role:** `n8n-nodes-base.webhook`  
  HTTP entry point for on-demand recommendation requests.
- **Configuration choices:**
  - Method: `POST`
  - Path: `trip-suggestion`
  - Response mode: `responseNode`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input node
  - Outputs to: `Validate and Normalize Request`
- **Version-specific requirements:**
  - Uses Webhook node type version `2`
  - Because `responseMode` is `responseNode`, a downstream `Respond to Webhook` node is required
- **Edge cases or potential failure types:**
  - Caller may send malformed JSON or unexpected schema
  - If downstream flow never reaches the response node, the HTTP caller may time out
- **Sub-workflow reference:** None

#### Validate and Normalize Request

- **Type and technical role:** `n8n-nodes-base.code`  
  Validates required fields, checks email syntax, normalizes preferences and constraints, and creates a canonical `tripRequest`.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Extracts payload from either `body` or root JSON
  - Requires:
    - `userId`
    - `userName`
    - `userEmail`
    - `preferences`
  - Defaults applied:
    - `budget`: `moderate`
    - `travelStyle`: `balanced`
    - `interests`: `[]`
    - `climate`: `any`
    - `duration`: `5-7 days`
    - `maxBudget`: `5000`
    - `departureCity`: `Not specified`
    - `companions`: `flexible`
    - `accessibility`: `none`
    - `prioritize`: `balanced`
    - `requestSource`: `web_app`
  - Generates:
    - `requestId`
    - `requestedAt`
    - `profileCompleteness`
    - `status: VALIDATED`
- **Key expressions or variables used:**
  - `$input.item.json.body || $input.item.json`
  - Email regex validation
  - `Date.now()` and `Math.random()` for request ID
- **Input and output connections:**
  - Input from: `Receive Trip Suggestion Request`
  - Outputs to:
    - `Fetch User Travel History`
    - `Fetch User Profile Data`
- **Version-specific requirements:**
  - Code node type version `2`
- **Edge cases or potential failure types:**
  - Missing required fields throws an error
  - Invalid email throws an error
  - `trim()` assumes string values for `userId`, `userName`, `userEmail`; non-string payloads may fail
  - `constraints.maxBudget` is defaulted only if falsy; a valid `0` budget would be replaced by `5000`
- **Sub-workflow reference:** None

---

## 2.2 Block: Travel History Retrieval & Pattern Analysis

### Overview

This block fetches trip and profile records from Google Sheets and derives user-level travel insights. These insights provide structured grounding for the AI prompt and reduce generic recommendations.

### Nodes Involved

- Fetch User Travel History
- Fetch User Profile Data
- Analyze Travel Patterns

### Node Details

#### Fetch User Travel History

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the `trip_history` sheet.
- **Configuration choices:**
  - Sheet name: `trip_history`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID_HERE`
  - Authentication: `serviceAccount`
  - `continueOnFail: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Validate and Normalize Request`
  - Output to: `Analyze Travel Patterns`
- **Version-specific requirements:**
  - Google Sheets node version `4.5`
  - Requires a configured Google API credential compatible with service accounts
- **Edge cases or potential failure types:**
  - Invalid spreadsheet ID
  - Missing sheet tab
  - Service account lacks access to the spreadsheet
  - If it fails, downstream analysis may receive no usable travel history because `continueOnFail` is enabled
- **Sub-workflow reference:** None

#### Fetch User Profile Data

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the `user_profiles` sheet.
- **Configuration choices:**
  - Sheet name: `user_profiles`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID_HERE`
  - Authentication: `serviceAccount`
  - `continueOnFail: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Validate and Normalize Request`
  - Output to: `Analyze Travel Patterns`
- **Version-specific requirements:**
  - Google Sheets node version `4.5`
- **Edge cases or potential failure types:**
  - Same Google Sheets access issues as above
  - The returned data is loaded but not actually used in the current analysis code, so this node currently contributes no analytical effect
- **Sub-workflow reference:** None

#### Analyze Travel Patterns

- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates travel history into user statistics and preference patterns for prompting the AI model.
- **Configuration choices:**
  - Reads:
    - `tripRequest` from `Validate and Normalize Request`
    - all rows from `Fetch User Travel History`
    - all rows from `Fetch User Profile Data`
  - Filters history rows where `t.userId === tripRequest.userId`
  - Computes:
    - total trips
    - recent trips in the last 12 months
    - average budget
    - average rating
    - favorite destination type
    - preferred climate
    - top region
    - travel frequency label
  - Builds:
    - `patterns`
    - `recentDestinations`
    - `highestRatedTrips`
- **Key expressions or variables used:**
  - `$('Validate and Normalize Request').item.json.tripRequest`
  - `$('Fetch User Travel History').all()`
  - `$('Fetch User Profile Data').all()`
- **Input and output connections:**
  - Inputs from:
    - `Fetch User Travel History`
    - `Fetch User Profile Data`
  - Output to: `Claude AI Trip Recommendation Engine`
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - If date values in `tripDate` are invalid, recent-trip filtering may behave unpredictably
  - Numeric fields like `totalCost` and `rating` may arrive as strings from Google Sheets; arithmetic may still work inconsistently unless explicitly converted
  - If history read failed but `continueOnFail` allowed execution, analysis may treat the user as a new traveler
  - `profileData` is loaded but not used, which may indicate an incomplete implementation
- **Sub-workflow reference:** None

---

## 2.3 Block: AI Recommendation Generation

### Overview

This block sends the structured travel analysis to Claude AI and asks for personalized recommendations in strict JSON format. It is the core intelligence layer of the workflow.

### Nodes Involved

- Claude AI Trip Recommendation Engine
- Claude Sonnet 4 Model

### Node Details

#### Claude AI Trip Recommendation Engine

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  An AI Agent node that builds the prompt and delegates language generation to the connected Anthropic chat model.
- **Configuration choices:**
  - Prompt type: `define`
  - Large inline prompt containing:
    - user profile
    - trip history summary
    - exclusions
    - required recommendation logic
    - strict JSON response schema
  - System message instructs model to return valid JSON only
- **Key expressions or variables used:**
  - `{{ $json.tripRequest.userId }}`
  - `{{ $json.userStats.avgBudget }}`
  - `{{ $json.tripRequest.preferences.interests.join(', ') }}`
  - `{{ $json.recentDestinations.join(', ') || 'No recent trips' }}`
  - `{{ $json.tripRequest.excludeRegions.join(', ') || 'None' }}`
- **Input and output connections:**
  - Main input from: `Analyze Travel Patterns`
  - AI language model input from: `Claude Sonnet 4 Model`
  - Main output to: `Parse AI Recommendations`
- **Version-specific requirements:**
  - Agent node version `1.6`
  - Requires the LangChain-compatible n8n AI nodes package
- **Edge cases or potential failure types:**
  - Model may still return malformed JSON despite instructions
  - Very sparse or inconsistent input data may reduce personalization quality
  - Prompt expressions like `.join(', ')` assume arrays are defined
  - Model token limits could become relevant if travel history is expanded later
- **Sub-workflow reference:** None

#### Claude Sonnet 4 Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Anthropic chat model provider used by the Agent node.
- **Configuration choices:**
  - Model: `claude-sonnet-4-20250514`
  - Temperature: `0.3`
  - Uses Anthropic API credentials
- **Key expressions or variables used:**
  - Model field is expression-based but resolves to a static string
- **Input and output connections:**
  - Output via `ai_languageModel` connection to: `Claude AI Trip Recommendation Engine`
- **Version-specific requirements:**
  - Anthropic chat model node version `1`
  - Requires valid Anthropic API credentials
- **Edge cases or potential failure types:**
  - Invalid or expired API key
  - Model availability or naming mismatch by workspace/version
  - Provider-side rate limiting or quota exhaustion
- **Sub-workflow reference:** None

---

## 2.4 Block: Enrichment, Persistence & Response Delivery

### Overview

This block converts the AI output into a normalized application record, optionally calls a weather API, stores the recommendation, sends an email, and prepares the final JSON response. It is operationally important but contains several branching behaviors that affect reliability and timing.

### Nodes Involved

- Parse AI Recommendations
- Wait for Data Enrichment
- Enrich with Weather Data (Optional)
- Store Recommendation Record
- Send Personalized Email
- Build API Response
- Wait Before Response
- Send Trip Recommendations Response

### Node Details

#### Parse AI Recommendations

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses Claude’s response text as JSON and creates the structured `recommendationRecord`.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Supports multiple possible response fields:
    - `response`
    - `output`
    - `text`
    - `content[0].text`
  - Removes markdown code fences before `JSON.parse`
  - Builds:
    - `recommendationId`
    - user metadata
    - top destination
    - confidence and strategy fields
    - request context
    - metadata such as model and generation time
- **Key expressions or variables used:**
  - `$input.item.json`
  - `$('Analyze Travel Patterns').item.json`
- **Input and output connections:**
  - Input from: `Claude AI Trip Recommendation Engine`
  - Output to: `Wait for Data Enrichment`
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - Invalid JSON from the model throws a parsing error
  - If recommendations array is missing, top destination defaults to `None`
  - Assumes `recommendations.recommendations` is an array-like structure
- **Sub-workflow reference:** None

#### Wait for Data Enrichment

- **Type and technical role:** `n8n-nodes-base.wait`  
  Introduces a pause before branching into enrichment, storage, and email operations.
- **Configuration choices:**
  - Amount: `2`
  - No unit is explicitly visible in the JSON snippet; behavior depends on node defaults/version
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Parse AI Recommendations`
  - Outputs to:
    - `Enrich with Weather Data (Optional)`
    - `Store Recommendation Record`
    - `Send Personalized Email`
- **Version-specific requirements:**
  - Wait node version `1.1`
- **Edge cases or potential failure types:**
  - Wait behavior may not serve true synchronization; it only delays and then fans out
  - Because downstream branches independently connect to `Build API Response`, the response path may execute multiple times or from incomplete data depending on execution order
- **Sub-workflow reference:** None

#### Enrich with Weather Data (Optional)

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls OpenWeatherMap for the top recommended destination.
- **Configuration choices:**
  - URL built from `recommendationRecord.topDestination`
  - Timeout: `5000 ms`
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - `https://api.openweathermap.org/data/2.5/weather?q={{ $json.recommendationRecord.topDestination }}&appid=YOUR_API_KEY&units=metric`
- **Input and output connections:**
  - Input from: `Wait for Data Enrichment`
  - Output to: `Build API Response`
- **Version-specific requirements:**
  - HTTP Request node version `4.2`
- **Edge cases or potential failure types:**
  - Placeholder API key must be replaced
  - `topDestination` uses `"City, Country"` formatting, which may not always be accepted by the weather API
  - No data merge occurs after enrichment, so returned weather data is not incorporated into the final response payload
  - API timeout or rate limit
- **Sub-workflow reference:** None

#### Store Recommendation Record

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the recommendation record to the `recommendations` sheet.
- **Configuration choices:**
  - Operation: `append`
  - Auto-map input data
  - Sheet name: `recommendations`
  - Uses OAuth2 Google Sheets credentials
  - `continueOnFail: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Wait for Data Enrichment`
  - Output to: `Build API Response`
- **Version-specific requirements:**
  - Google Sheets node version `4.5`
- **Edge cases or potential failure types:**
  - Spreadsheet columns may not match nested object fields cleanly
  - Google Sheets append may serialize arrays/objects in unexpected ways
  - Access/permission issues
- **Sub-workflow reference:** None

#### Send Personalized Email

- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends recommendation notification to the traveler.
- **Configuration choices:**
  - Subject uses top destination
  - To: `recommendationRecord.userEmail`
  - From: `user@example.com`
  - SMTP credentials required
  - `continueOnFail: true`
  - No email body is configured in the provided workflow
- **Key expressions or variables used:**
  - `{{ $json.recommendationRecord.topDestination }}`
  - `{{ $json.recommendationRecord.userEmail }}`
- **Input and output connections:**
  - Input from: `Wait for Data Enrichment`
  - Output to: `Build API Response`
- **Version-specific requirements:**
  - Email Send node version `2.1`
- **Edge cases or potential failure types:**
  - Missing body content may result in a blank email depending on node defaults
  - SMTP auth or sender-address rejection
  - Since this branch emits its own output shape, it may not contain the fields expected by `Build API Response`
- **Sub-workflow reference:** None

#### Build API Response

- **Type and technical role:** `n8n-nodes-base.code`  
  Creates the final simplified JSON response for the webhook caller.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Pulls canonical data directly from `Parse AI Recommendations`
  - Returns:
    - `success`
    - `recommendationId`
    - `userId`
    - `generatedAt`
    - `topDestination`
    - recommendation summary list
    - strategy and budget tips
    - human-readable message
- **Key expressions or variables used:**
  - `$('Parse AI Recommendations').item.json`
- **Input and output connections:**
  - Inputs from:
    - `Enrich with Weather Data (Optional)`
    - `Store Recommendation Record`
    - `Send Personalized Email`
  - Output to: `Wait Before Response`
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - Because this node has three incoming branches but uses data from `Parse AI Recommendations`, it does not merge branch outputs
  - It may execute once per incoming branch, which can cause duplicate downstream executions or webhook response conflicts
  - Assumes `full.recommendations`, `highlights`, and `insiderTips` exist and are arrays
- **Sub-workflow reference:** None

#### Wait Before Response

- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds a short pause before responding to the webhook.
- **Configuration choices:**
  - Amount: `1`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Build API Response`
  - Output to: `Send Trip Recommendations Response`
- **Version-specific requirements:**
  - Wait node version `1.1`
- **Edge cases or potential failure types:**
  - As with the earlier Wait node, this is not a guaranteed branch join mechanism
  - Multiple upstream executions may still produce multiple attempts to respond
- **Sub-workflow reference:** None

#### Send Trip Recommendations Response

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the final JSON HTTP response back to the original webhook caller.
- **Configuration choices:**
  - Respond with: `json`
  - Body: pretty-printed `JSON.stringify($json, null, 2)`
  - Header: `Content-Type: application/json`
- **Key expressions or variables used:**
  - `={{ JSON.stringify($json, null, 2) }}`
- **Input and output connections:**
  - Input from: `Wait Before Response`
  - No output
- **Version-specific requirements:**
  - Respond to Webhook node version `1`
- **Edge cases or potential failure types:**
  - If multiple upstream branches trigger multiple executions, only one HTTP response can be sent successfully
  - Responding with JSON while also stringifying manually may produce a JSON string body rather than a structured object depending on n8n behavior
- **Sub-workflow reference:** None

---

## 2.5 Block: Scheduled Batch Recommendations & Trend Analysis

### Overview

This branch runs on a schedule and aggregates user-level trend data from the profile sheet. It identifies users due for refreshed recommendations and produces a trend report for operational monitoring.

### Nodes Involved

- Daily Batch Recommendation Trigger
- Fetch All Active Users
- Analyze Emerging Travel Trends
- Send Weekly Trend Report
- Log Analytics to Sheet

### Node Details

#### Daily Batch Recommendation Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry point for analytics and batch refresh evaluation.
- **Configuration choices:**
  - Cron expression: `0 8 * * *`
  - Runs daily at 08:00
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input node
  - Output to: `Fetch All Active Users`
- **Version-specific requirements:**
  - Schedule Trigger version `1.2`
- **Edge cases or potential failure types:**
  - Time zone depends on instance/workflow settings
  - If the instance is down at scheduled time, execution may be missed depending on deployment mode
- **Sub-workflow reference:** None

#### Fetch All Active Users

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads all user records from the `user_profiles` sheet.
- **Configuration choices:**
  - Sheet name: `user_profiles`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID_HERE`
  - Uses Google Sheets OAuth2 credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Daily Batch Recommendation Trigger`
  - Output to: `Analyze Emerging Travel Trends`
- **Version-specific requirements:**
  - Google Sheets node version `4.5`
- **Edge cases or potential failure types:**
  - Missing spreadsheet or sheet access
  - No filtering for “active” users is actually configured despite the node name
- **Sub-workflow reference:** None

#### Analyze Emerging Travel Trends

- **Type and technical role:** `n8n-nodes-base.code`  
  Computes user refresh eligibility and aggregates trend metrics from profile data.
- **Configuration choices:**
  - Reads all rows from input
  - Flags users needing update if:
    - no `lastRecommendationDate`, or
    - at least 7 days since last recommendation
  - Aggregates:
    - destination searches from `recentSearches`
    - style popularity from `travelStyle`
    - budget distribution from `typicalBudget`
  - Returns a report with:
    - `reportDate`
    - `totalUsers`
    - `usersNeedingUpdate`
    - `trends`
    - `usersToUpdate`
- **Key expressions or variables used:**
  - `$input.all().map(i => i.json)`
- **Input and output connections:**
  - Input from: `Fetch All Active Users`
  - Output to: `Send Weekly Trend Report`
- **Version-specific requirements:**
  - Code node version `2`
- **Edge cases or potential failure types:**
  - Assumes `recentSearches` is iterable; if stored as a comma-separated string in Sheets, `.forEach()` will fail
  - Date parsing may fail for non-ISO sheet values
  - Node name says “emerging trends,” but no actual recommendation refresh is triggered for users
- **Sub-workflow reference:** None

#### Send Weekly Trend Report

- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Emails the trend report.
- **Configuration choices:**
  - Subject: `Weekly Travel Trends Report - <date>`
  - To and From: `user@example.com`
  - SMTP credentials required
  - `continueOnFail: true`
  - No email body is configured
- **Key expressions or variables used:**
  - `{{ new Date().toDateString() }}`
- **Input and output connections:**
  - Input from: `Analyze Emerging Travel Trends`
  - Output to: `Log Analytics to Sheet`
- **Version-specific requirements:**
  - Email Send node version `2.1`
- **Edge cases or potential failure types:**
  - Empty email body
  - SMTP auth or deliverability issues
  - Since output shape may differ after email send, downstream append may log email-send metadata instead of the report unless node output is preserved as expected
- **Sub-workflow reference:** None

#### Log Analytics to Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends analytics/trend data to the `analytics` sheet.
- **Configuration choices:**
  - Operation: `append`
  - Auto-map input data
  - Sheet name: `analytics`
  - Uses Google Sheets OAuth2 credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from: `Send Weekly Trend Report`
  - No output
- **Version-specific requirements:**
  - Google Sheets node version `4.5`
- **Edge cases or potential failure types:**
  - Nested trend objects may not map cleanly to sheet columns
  - If previous node output is altered, intended analytics data may not be what is actually written
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Documentation | Sticky Note | Workspace documentation and setup guidance |  |  | ## Smart Next Trip Suggestion Engine with Claude AI<br>This workflow provides personalized travel destination recommendations by analyzing past trip history, user preferences, travel behavior patterns, and current trends. It uses Claude AI to generate intelligent, context-aware suggestions tailored to each traveler.<br>How it works: Receive request, validate, fetch history, analyze patterns, generate AI recommendations, enrich, store, respond, run daily updates, and trend analysis.<br>Setup: configure Anthropic, Google Sheets, SMTP/Gmail, optional Weather API and Flight API; create `user_profiles`, `trip_history`, `recommendations`, `analytics`; activate webhook and scheduled branches. |
| Section 1 Header | Sticky Note | Visual section label |  |  | ## 1. Request Intake & Validation |
| Section 2 Header | Sticky Note | Visual section label |  |  | ## 2. History Analysis & AI Recommendation |
| Section 3 Header | Sticky Note | Visual section label |  |  | ## 3. Enrichment, Storage & Delivery |
| Section 4 Header | Sticky Note | Visual section label |  |  | ## 4. Daily Batch Recommendations & Trend Analysis |
| Receive Trip Suggestion Request | Webhook | Receives POST trip recommendation requests |  | Validate and Normalize Request | ## 1. Request Intake & Validation |
| Validate and Normalize Request | Code | Validates request and builds normalized `tripRequest` | Receive Trip Suggestion Request | Fetch User Travel History; Fetch User Profile Data | ## 1. Request Intake & Validation |
| Fetch User Travel History | Google Sheets | Reads trip history rows | Validate and Normalize Request | Analyze Travel Patterns | ## 2. History Analysis & AI Recommendation |
| Fetch User Profile Data | Google Sheets | Reads user profile rows | Validate and Normalize Request | Analyze Travel Patterns | ## 2. History Analysis & AI Recommendation |
| Analyze Travel Patterns | Code | Derives travel behavior metrics and user stats | Fetch User Travel History; Fetch User Profile Data | Claude AI Trip Recommendation Engine | ## 2. History Analysis & AI Recommendation |
| Claude AI Trip Recommendation Engine | LangChain Agent | Generates structured travel recommendations prompt flow | Analyze Travel Patterns; Claude Sonnet 4 Model | Parse AI Recommendations | ## 2. History Analysis & AI Recommendation |
| Claude Sonnet 4 Model | Anthropic Chat Model | Provides Claude Sonnet 4 language model to the agent |  | Claude AI Trip Recommendation Engine | ## 2. History Analysis & AI Recommendation |
| Parse AI Recommendations | Code | Parses Claude JSON and builds `recommendationRecord` | Claude AI Trip Recommendation Engine | Wait for Data Enrichment | ## 3. Enrichment, Storage & Delivery |
| Wait for Data Enrichment | Wait | Delays before enrichment/storage/delivery fan-out | Parse AI Recommendations | Enrich with Weather Data (Optional); Store Recommendation Record; Send Personalized Email | ## 3. Enrichment, Storage & Delivery |
| Enrich with Weather Data (Optional) | HTTP Request | Fetches weather for top destination | Wait for Data Enrichment | Build API Response | ## 3. Enrichment, Storage & Delivery |
| Store Recommendation Record | Google Sheets | Appends recommendation record to sheet | Wait for Data Enrichment | Build API Response | ## 3. Enrichment, Storage & Delivery |
| Send Personalized Email | Email Send | Emails recommendation notification to traveler | Wait for Data Enrichment | Build API Response | ## 3. Enrichment, Storage & Delivery |
| Build API Response | Code | Creates final API response payload | Enrich with Weather Data (Optional); Store Recommendation Record; Send Personalized Email | Wait Before Response | ## 3. Enrichment, Storage & Delivery |
| Wait Before Response | Wait | Delays before webhook response | Build API Response | Send Trip Recommendations Response | ## 3. Enrichment, Storage & Delivery |
| Send Trip Recommendations Response | Respond to Webhook | Returns final JSON response to API caller | Wait Before Response |  | ## 3. Enrichment, Storage & Delivery |
| Daily Batch Recommendation Trigger | Schedule Trigger | Starts daily trend-analysis branch |  | Fetch All Active Users | ## 4. Daily Batch Recommendations & Trend Analysis |
| Fetch All Active Users | Google Sheets | Reads user profile rows for batch analysis | Daily Batch Recommendation Trigger | Analyze Emerging Travel Trends | ## 4. Daily Batch Recommendations & Trend Analysis |
| Analyze Emerging Travel Trends | Code | Computes update candidates and aggregate trends | Fetch All Active Users | Send Weekly Trend Report | ## 4. Daily Batch Recommendations & Trend Analysis |
| Send Weekly Trend Report | Email Send | Sends trend-report email | Analyze Emerging Travel Trends | Log Analytics to Sheet | ## 4. Daily Batch Recommendations & Trend Analysis |
| Log Analytics to Sheet | Google Sheets | Appends analytics report to sheet | Send Weekly Trend Report |  | ## 4. Daily Batch Recommendations & Trend Analysis |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Smart Next Trip Suggestion Engine with Claude AI and Behavioral Analysis`.

2. **Add a Webhook node** named `Receive Trip Suggestion Request`.
   - Type: Webhook
   - HTTP Method: `POST`
   - Path: `trip-suggestion`
   - Response Mode: `Using Respond to Webhook Node` / `responseNode`

3. **Add a Code node** named `Validate and Normalize Request`.
   - Connect it after the webhook.
   - Set mode to `Run Once for Each Item`.
   - Paste logic that:
     - reads payload from `body` or root
     - validates `userId`, `userName`, `userEmail`, `preferences`
     - validates email format
     - normalizes `preferences` and `constraints`
     - computes `profileCompleteness`
     - generates `requestId`
     - outputs `{ tripRequest }`

4. **Add a Google Sheets node** named `Fetch User Travel History`.
   - Connect from `Validate and Normalize Request`.
   - Use operation to read rows from sheet `trip_history`.
   - Set document ID to your spreadsheet ID.
   - Authentication: service account.
   - Enable `Continue On Fail`.

5. **Configure Google service account credentials** for the history node.
   - In Google Cloud, create or reuse a service account.
   - Enable Google Sheets API.
   - Share the spreadsheet with the service account email.
   - Add the credential in n8n and attach it to this node.

6. **Add a second Google Sheets node** named `Fetch User Profile Data`.
   - Connect from `Validate and Normalize Request`.
   - Read rows from sheet `user_profiles`.
   - Use the same spreadsheet ID.
   - Authentication: service account.
   - Enable `Continue On Fail`.

7. **Add a Code node** named `Analyze Travel Patterns`.
   - Connect both sheet nodes into it.
   - Implement logic that:
     - loads `tripRequest` from the validation node
     - loads all travel history rows
     - optionally loads profile rows
     - filters trips by matching `userId`
     - calculates:
       - total trips
       - recent trips in the last 12 months
       - average budget
       - average rating
       - favorite destination type
       - preferred climate
       - top region
       - travel frequency
     - returns a single object with:
       - `tripRequest`
       - `userStats`
       - `patterns`
       - `recentDestinations`
       - `highestRatedTrips`

8. **Add an AI Agent node** named `Claude AI Trip Recommendation Engine`.
   - Connect it after `Analyze Travel Patterns`.
   - Set prompt type to `Define`.
   - Paste a prompt that includes:
     - user profile fields
     - current request data
     - travel history summary
     - exclusion list
     - instructions to recommend 3–5 destinations
     - strict JSON output schema
   - Add a system message instructing the model to return valid JSON only.

9. **Add an Anthropic Chat Model node** named `Claude Sonnet 4 Model`.
   - Connect it to the agent using the AI language model connector.
   - Model: `claude-sonnet-4-20250514`
   - Temperature: `0.3`

10. **Configure Anthropic credentials**.
    - Create an Anthropic API key.
    - Add it in n8n as Anthropic credentials.
    - Attach it to `Claude Sonnet 4 Model`.

11. **Add a Code node** named `Parse AI Recommendations`.
    - Connect it after the AI Agent.
    - Set mode to `Run Once for Each Item`.
    - Implement parsing logic that:
      - reads output text from possible response fields
      - strips code fences
      - parses JSON
      - loads original analysis context from `Analyze Travel Patterns`
      - builds:
        - `recommendationRecord`
        - `fullRecommendations`

12. **Add a Wait node** named `Wait for Data Enrichment`.
    - Connect it after `Parse AI Recommendations`.
    - Set amount to `2`.
    - Keep in mind this is a delay, not a true merge/join mechanism.

13. **Add an HTTP Request node** named `Enrich with Weather Data (Optional)`.
    - Connect it from `Wait for Data Enrichment`.
    - Method: GET
    - URL:
      `https://api.openweathermap.org/data/2.5/weather?q={{ $json.recommendationRecord.topDestination }}&appid=YOUR_API_KEY&units=metric`
    - Timeout: `5000`
    - Enable `Continue On Fail`.

14. **Replace the weather API placeholder**.
    - Create an OpenWeatherMap API key.
    - Prefer storing the key in credentials or environment variables rather than directly in the URL.
    - Note that querying by `"City, Country"` may need adjustment for some destinations.

15. **Add a Google Sheets node** named `Store Recommendation Record`.
    - Connect it from `Wait for Data Enrichment`.
    - Operation: `Append`
    - Sheet: `recommendations`
    - Spreadsheet ID: same Google Sheet or another destination sheet
    - Mapping mode: `Auto-map input data`
    - Enable `Continue On Fail`

16. **Configure OAuth2 Google Sheets credentials** for append operations.
    - This workflow uses OAuth2 for append nodes, while read nodes use a service account.
    - You may standardize both if preferred, but if reproducing exactly, create:
      - one Google API service account credential for reads
      - one Google Sheets OAuth2 credential for appends

17. **Add an Email Send node** named `Send Personalized Email`.
    - Connect it from `Wait for Data Enrichment`.
    - To: `{{ $json.recommendationRecord.userEmail }}`
    - From: your sender address
    - Subject: `Your Personalized Travel Recommendations - {{ $json.recommendationRecord.topDestination }} & More!`
    - Enable `Continue On Fail`
    - Add a body manually, because the provided workflow does not define one and many implementations require it.

18. **Configure SMTP credentials**.
    - Add SMTP credentials in n8n.
    - Use a valid sender address allowed by your mail server.
    - If using Gmail or Outlook, ensure application-password or OAuth requirements are met.

19. **Add a Code node** named `Build API Response`.
    - Connect all three nodes into it:
      - `Enrich with Weather Data (Optional)`
      - `Store Recommendation Record`
      - `Send Personalized Email`
    - Implement logic that reads canonical data from `Parse AI Recommendations` and returns:
      - `success`
      - `recommendationId`
      - `userId`
      - `generatedAt`
      - `topDestination`
      - `totalRecommendations`
      - `confidenceScore`
      - reduced recommendation objects
      - `strategy`
      - `budgetTips`
      - human-friendly message

20. **Add a Wait node** named `Wait Before Response`.
    - Connect it after `Build API Response`.
    - Set amount to `1`.

21. **Add a Respond to Webhook node** named `Send Trip Recommendations Response`.
    - Connect it after `Wait Before Response`.
    - Respond with JSON.
    - Set response body to `{{ JSON.stringify($json, null, 2) }}`
    - Add header:
      - `Content-Type: application/json`

22. **Create the scheduled branch entry point**.
    - Add a Schedule Trigger node named `Daily Batch Recommendation Trigger`.
    - Set cron expression to `0 8 * * *`.

23. **Add a Google Sheets node** named `Fetch All Active Users`.
    - Connect it after the schedule trigger.
    - Read rows from `user_profiles`.
    - Use OAuth2 Google Sheets credentials, matching the original workflow.

24. **Add a Code node** named `Analyze Emerging Travel Trends`.
    - Connect it after `Fetch All Active Users`.
    - Implement logic that:
      - loads all users
      - marks users as needing update if no recommendation exists or if the last recommendation is at least 7 days old
      - aggregates `recentSearches`
      - aggregates `travelStyle`
      - bins `typicalBudget` into budget/moderate/luxury
      - returns one analytics report object

25. **Add an Email Send node** named `Send Weekly Trend Report`.
    - Connect it after `Analyze Emerging Travel Trends`.
    - Subject: `Weekly Travel Trends Report - {{ new Date().toDateString() }}`
    - To: your reporting address
    - From: your sender address
    - Add an email body manually for practical use.
    - Enable `Continue On Fail`.

26. **Add a Google Sheets node** named `Log Analytics to Sheet`.
    - Connect it after `Send Weekly Trend Report`.
    - Operation: `Append`
    - Sheet name: `analytics`
    - Mapping mode: auto-map input data

27. **Create the spreadsheet structure** in Google Sheets.
    - Tab `user_profiles`
    - Tab `trip_history`
    - Tab `recommendations`
    - Tab `analytics`

28. **Populate expected columns** in the source sheets.
    - `trip_history` should ideally include:
      - `userId`
      - `tripDate`
      - `destination`
      - `destinationType`
      - `climate`
      - `totalCost`
      - `rating`
      - `region`
    - `user_profiles` should ideally include:
      - `userId`
      - `email`
      - `lastRecommendationDate`
      - `recentSearches`
      - `travelStyle`
      - `typicalBudget`

29. **Add visual documentation sticky notes** if you want to replicate the original layout.
    - Main workspace note explaining purpose, setup, sample request, and features
    - Section headers for:
      - Request Intake & Validation
      - History Analysis & AI Recommendation
      - Enrichment, Storage & Delivery
      - Daily Batch Recommendations & Trend Analysis

30. **Test the webhook branch** with a sample POST payload similar to:
    - `userId`
    - `userName`
    - `userEmail`
    - `preferences`
    - `constraints`
    - optional exclusions and priority

31. **Test the scheduled branch manually** by executing from `Daily Batch Recommendation Trigger` or the first downstream node.

32. **Review branch behavior carefully before production use**.
    - In the provided design, three branches all flow into `Build API Response`.
    - This can lead to multiple executions and webhook response conflicts.
    - A more robust reproduction would add explicit merge logic before responding.

33. **Activate the workflow** only after:
    - credentials are valid
    - spreadsheet tabs exist
    - sender email is authorized
    - webhook branch returns exactly one response per request

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow title in metadata is `Smart Next Trip Suggestion Engine with Claude AI and Behavioral Analysis`, while the provided title is `Generate personalized trip recommendations with Claude AI and Google Sheets`. | Naming discrepancy between provided title and workflow name |
| The main documentation note includes a sample request payload with nested `preferences` and `constraints`. | Use it as the contract for webhook testing |
| Optional integrations mentioned in the documentation include Weather API and Flight API, but only weather enrichment is partially implemented in the workflow. | Weather API is present; flight enrichment is not implemented |
| The documentation references activation of both webhook and scheduled flows. | Operational deployment guidance |
| Google Sheets is used both as a source system and as a storage/logging destination. | `user_profiles`, `trip_history`, `recommendations`, `analytics` |
| The workflow uses mixed Google authentication methods: service account for some reads and OAuth2 for appends. | Consider standardizing during hardening |
| The current design does not include a sub-workflow node and does not call any child workflows. | No sub-workflow setup required |
| The disclaimer states that the source content comes from a lawful n8n automation workflow and contains no illegal, offensive, or protected content. | Context note provided with the request |