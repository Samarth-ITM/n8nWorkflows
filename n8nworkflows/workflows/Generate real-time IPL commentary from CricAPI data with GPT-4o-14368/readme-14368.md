Generate real-time IPL commentary from CricAPI data with GPT-4o

https://n8nworkflows.xyz/workflows/generate-real-time-ipl-commentary-from-cricapi-data-with-gpt-4o-14368


# Generate real-time IPL commentary from CricAPI data with GPT-4o

# 1. Workflow Overview

This workflow generates short, real-time cricket match narratives from live CricAPI data and GPT-4o-mini. It is designed primarily for IPL fan apps, widgets, or dashboards that need a compact analyst-style update every few minutes, but it also includes fallback logic so it can still operate on other live T20 matches when no IPL game is active.

The workflow has two entry points: a scheduled trigger for automatic polling and a webhook for manual/on-demand execution. Both paths converge into the same processing chain: fetch live matches, identify a relevant match, compute match indicators, generate a narrative with OpenAI, log the result to Google Sheets, and return a JSON response to the webhook caller.

## 1.1 Input Reception

Two triggers start the workflow:
- a cron-based Schedule Trigger that runs every 6 minutes during defined hours
- a Webhook node for manual testing or app-driven invocation

Both feed into the same HTTP request to CricAPI.

## 1.2 Match Retrieval and Indicator Computation

The workflow pulls current matches from CricAPI, then uses a Code node to:
- find a live IPL T20 match if possible
- otherwise fall back to a live T20 match
- otherwise fall back to any scored T20 for demo/testing

It derives cricket-specific indicators such as innings, score, overs, current run rate, required run rate, runs needed, wickets in hand, phase, pressure level, and projected total.

## 1.3 Conditional Gating

An If node checks whether the Code node marked the run as skippable. If no suitable match exists, processing stops here.

## 1.4 AI Narrative Generation

The computed match context is converted into a prompt and sent to the OpenAI node using `gpt-4o-mini`. The model is instructed to produce exactly two sentences, use concrete numbers, avoid percentages, and sound like a human cricket analyst.

## 1.5 Output Parsing, Logging, and API Response

The AI output is parsed and merged back with the computed match data. The result is appended to a Google Sheet for tracking, then returned as JSON through a Respond to Webhook node.

---

# 2. Block-by-Block Analysis

## 2.1 Triggering Block

### Overview
This block provides two ways to start the workflow: automatically on a schedule and manually via webhook. Both entry points are functionally equivalent after the trigger stage.

### Nodes Involved
- Schedule Trigger
- Manual test trigger

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; starts the workflow automatically on a cron schedule.
- **Configuration choices:** Configured with cron expression `*/6 14-23 * * *`, which runs every 6 minutes between 14:00 and 23:59 server time every day.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Fetch live match list`.
- **Version-specific requirements:** Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**
  - Timezone ambiguity: execution depends on the n8n instance timezone.
  - If the server timezone does not align with IPL match timing, runs may occur at unintended hours.
- **Sub-workflow reference:** None.

#### Manual test trigger
- **Type and technical role:** `n8n-nodes-base.webhook`; exposes an HTTP endpoint to trigger the workflow manually.
- **Configuration choices:**
  - Path: `ipl-narrative`
  - Response mode: `responseNode`, meaning the workflow must eventually reach a `Respond to Webhook` node.
- **Key expressions or variables used:** None in the trigger itself.
- **Input and output connections:** No input; outputs to `Fetch live match list`.
- **Version-specific requirements:** Uses `typeVersion 2.1`.
- **Edge cases or potential failure types:**
  - If execution stops before `Respond to Webhook`, the webhook caller may hang or get an incomplete response.
  - Path collisions can occur if another webhook uses the same route.
  - Production/test webhook URL differences may matter during validation.
- **Sub-workflow reference:** None.

---

## 2.2 Match Retrieval and Computation Block

### Overview
This block queries CricAPI for current matches and transforms the raw match list into a single structured cricket context payload suitable for AI prompting. It contains the workflow’s main business logic and fallback behavior.

### Nodes Involved
- Fetch live match list
- Compute Match Indicators

### Node Details

#### Fetch live match list
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls CricAPI to retrieve current matches.
- **Configuration choices:**
  - Method defaults to GET.
  - URL: `https://api.cricapi.com/v1/currentMatches`
  - Query parameters:
    - `apikey=your_api_key`
    - `offset=0`
- **Key expressions or variables used:** None currently; the API key is hardcoded placeholder text.
- **Input and output connections:** Inputs from `Schedule Trigger` and `Manual test trigger`; outputs to `Compute Match Indicators`.
- **Version-specific requirements:** Uses `typeVersion 4.3`.
- **Edge cases or potential failure types:**
  - Invalid or missing CricAPI key.
  - Rate limit responses from CricAPI.
  - Upstream API downtime or latency.
  - Response shape changes could break downstream Code node assumptions.
- **Sub-workflow reference:** None.

#### Compute Match Indicators
- **Type and technical role:** `n8n-nodes-base.code`; parses CricAPI response, selects a match, computes indicators, and creates the LLM prompt.
- **Configuration choices:** Custom JavaScript logic with several layers:
  1. Validate `response.status === "success"`
  2. Read `response.data`
  3. Select live IPL T20 first
  4. Fallback to any live T20 with score
  5. Fallback to any T20 with score for demo use
  6. If nothing matches, return `{ skip: true, reason: 'No T20 matches available' }`
  7. Detect innings based on score array length
  8. Compute different metrics for 1st vs 2nd innings
  9. Build a prompt string for OpenAI
- **Key expressions or variables used:**
  - `response.status`
  - `response.data`
  - `matches.find(...)`
  - `demoMatch.score`
  - `demoMatch.teams?.[0]`, `demoMatch.teams?.[1]`
  - Computed fields:
    - `innings`
    - `score`
    - `overs`
    - `oversRemaining`
    - `crr`
    - `projectedTotal`
    - `target`
    - `runsNeeded`
    - `rrr`
    - `rrrGap`
    - `wicketsInHand`
    - `phase`
    - `pressure`
    - `prompt`
    - `matchName`
    - `isLiveIPL`
    - `timestamp`
- **Input and output connections:** Input from `Fetch live match list`; output to `Is Live Match ?`.
- **Version-specific requirements:** Uses `typeVersion 2`.
- **Edge cases or potential failure types:**
  - Throws an error if CricAPI returns a non-success status.
  - Assumes the CricAPI payload includes `data`, `name`, `matchType`, `matchStarted`, `matchEnded`, `score`, and `teams`.
  - Team ordering assumptions may be incorrect if API team arrays do not correspond exactly to innings order.
  - A match with partial or malformed score objects may lead to inaccurate metrics.
  - `parseFloat` on overs may yield unexpected values if overs strings use nonstandard formatting.
  - `rrr` becomes `99.00` when no overs remain, which is an artificial sentinel rather than a true cricket calculation.
- **Sub-workflow reference:** None.

---

## 2.3 Conditional Gating Block

### Overview
This block prevents downstream AI generation when the prior Code node explicitly marked the execution as skippable. It acts as a lightweight guard.

### Nodes Involved
- Is Live Match ?

### Node Details

#### Is Live Match ?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether a valid match payload should proceed.
- **Configuration choices:**
  - Condition: `{{$json.skip}}` not equals `"true"`
  - Loose type validation enabled
- **Key expressions or variables used:** `{{$json.skip}}`
- **Input and output connections:** Input from `Compute Match Indicators`; true output goes to `Narrative Generator`. No false branch is connected.
- **Version-specific requirements:** Uses `typeVersion 2.3`.
- **Edge cases or potential failure types:**
  - Because the comparison is against string `"true"`, boolean/string coercion behavior matters.
  - With loose validation, `undefined` typically passes the condition, which is intended here.
  - No false-path handling means skipped executions simply end silently.
  - For webhook executions, ending silently before the response node can lead to no response or timeout.
- **Sub-workflow reference:** None.

---

## 2.4 AI Narrative Generation Block

### Overview
This block sends the generated match prompt to OpenAI and requests a concise, structured cricket narrative. The system message constrains the style and length.

### Nodes Involved
- Narrative Generator

### Node Details

#### Narrative Generator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; calls OpenAI chat-style generation.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - System instruction:
    - expert IPL cricket analyst
    - short punchy win-probability narratives
    - be specific about numbers
    - never use percentages
    - exactly 2 sentences
  - User content: `={{ $json.prompt }}`
- **Key expressions or variables used:** `{{$json.prompt}}`
- **Input and output connections:** Input from `Is Live Match ?`; output to `Parse Narrative`.
- **Version-specific requirements:** Uses `typeVersion 2.1`; depends on the LangChain OpenAI node package and a valid OpenAI credential.
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials.
  - Model availability changes or account restrictions.
  - LLM may occasionally violate the 2-sentence or no-percent instruction.
  - Prompt quality fully depends on upstream computed values.
  - Token/network errors can interrupt execution.
- **Sub-workflow reference:** None.

---

## 2.5 Parsing, Logging, and Response Block

### Overview
This block extracts the generated text from the OpenAI node output, restores the upstream match metadata, logs the result to Google Sheets, and returns a webhook-friendly JSON response.

### Nodes Involved
- Parse Narrative
- Log to IPL Win Probability Log
- Respond to Webhook

### Node Details

#### Parse Narrative
- **Type and technical role:** `n8n-nodes-base.code`; normalizes OpenAI output into a simple `narrative` field and merges it with match data.
- **Configuration choices:**
  - Reads AI response from `response.output?.[0]?.content?.[0]?.text`
  - Falls back to `'Narrative unavailable.'`
  - Pulls the original computed payload using `$('Compute Match Indicators').first().json`
- **Key expressions or variables used:**
  - `response.output?.[0]?.content?.[0]?.text`
  - `$('Compute Match Indicators').first().json`
- **Input and output connections:** Input from `Narrative Generator`; output to `Log to IPL Win Probability Log`.
- **Version-specific requirements:** Uses `typeVersion 2`.
- **Edge cases or potential failure types:**
  - If the OpenAI node output structure changes, narrative extraction will fail or fall back.
  - Direct node-reference access ties this node tightly to the upstream node name.
  - If multiple items are ever processed, `.first()` may cause incorrect pairing.
- **Sub-workflow reference:** None.

#### Log to IPL Win Probability Log
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends each generated narrative as a row in a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `IPL Win Probability Log`
  - Sheet: `Sheet1` (`gid=0`)
  - Explicit column mapping:
    - `timestamp` ← `{{$json.timestamp}}`
    - `match_name` ← `{{$json.matchName}}`
    - `innings` ← `{{$json.innings}}`
    - `over` ← `{{$json.overs}}`
    - `score` ← `{{$json.score}}`
    - `target` ← `{{$json.target}}`
    - `rrr` ← `{{$json.rrr}}`
    - `crr` ← `{{$json.crr}}`
    - `phase` ← `{{$json.phase}}`
    - `pressure` ← `{{$json.pressure}}`
    - `narrative` ← `{{$json.narrative}}`
- **Key expressions or variables used:** The expressions above.
- **Input and output connections:** Input from `Parse Narrative`; output to `Respond to Webhook`.
- **Version-specific requirements:** Uses `typeVersion 4.7`; requires Google Sheets OAuth2 credentials and access to the target file.
- **Edge cases or potential failure types:**
  - OAuth token expiration or revoked sheet access.
  - Column/schema mismatch if the sheet headers differ.
  - Append failures if the document or sheet is deleted/renamed.
  - Data type expectations are loose because conversion is disabled.
- **Sub-workflow reference:** None.

#### Respond to Webhook
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; returns JSON to the webhook caller.
- **Configuration choices:**
  - Respond with: JSON
  - Body template includes:
    - `match`
    - `innings`
    - `score`
    - `over`
    - `phase`
    - `pressure`
    - `narrative`
    - `timestamp`
- **Key expressions or variables used:**
  - `{{ $json.match_name }}`
  - `{{ $json.innings }}`
  - `{{ $json.score }}`
  - `{{ $json.over }}`
  - `{{ $json.phase }}`
  - `{{ $json.pressure }}`
  - `{{ $json.narrative }}`
  - `{{ $json.timestamp }}`
- **Input and output connections:** Input from `Log to IPL Win Probability Log`; no output.
- **Version-specific requirements:** Uses `typeVersion 1.5`.
- **Edge cases or potential failure types:**
  - Field-name mismatch: upstream data uses `matchName` and `overs`, but this node references `match_name` and `over`. It works only if those fields remain present after Google Sheets output, which is not guaranteed.
  - If the workflow is started by the schedule trigger, this node has no webhook caller, but n8n will still execute it harmlessly in most cases.
  - If the workflow stops earlier, webhook calls may not receive a response.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point every 6 minutes during defined hours |  | Fetch live match list | Two triggers run this workflow. The Schedule Trigger fires automatically every 6 minutes between 2PM and 11PM covering all IPL match windows. The Webhook Trigger lets you fire it manually at any time for testing or on-demand narrative generation without waiting for the schedule. Both triggers feed into the same API call so the logic is identical. |
| Manual test trigger | n8n-nodes-base.webhook | Manual/on-demand workflow entry via HTTP webhook |  | Fetch live match list | Two triggers run this workflow. The Schedule Trigger fires automatically every 6 minutes between 2PM and 11PM covering all IPL match windows. The Webhook Trigger lets you fire it manually at any time for testing or on-demand narrative generation without waiting for the schedule. Both triggers feed into the same API call so the logic is identical. |
| Fetch live match list | n8n-nodes-base.httpRequest | Retrieves current matches from CricAPI | Schedule Trigger, Manual test trigger | Compute Match Indicators | One API call to CricAPI fetches all current matches. The Code node filters for a live IPL match first — if IPL is not in season it falls back to any live T20 match for testing. It then detects which innings is active and computes all indicators — CRR, RRR, runs needed, overs remaining, wickets in hand, match phase, and pressure level. Both 1st and 2nd innings are handled with different prompt logic per scenario. |
| Compute Match Indicators | n8n-nodes-base.code | Selects relevant match, computes indicators, builds prompt | Fetch live match list | Is Live Match ? | One API call to CricAPI fetches all current matches. The Code node filters for a live IPL match first — if IPL is not in season it falls back to any live T20 match for testing. It then detects which innings is active and computes all indicators — CRR, RRR, runs needed, overs remaining, wickets in hand, match phase, and pressure level. Both 1st and 2nd innings are handled with different prompt logic per scenario. |
| Is Live Match ? | n8n-nodes-base.if | Guards AI generation when no match is available | Compute Match Indicators | Narrative Generator | One API call to CricAPI fetches all current matches. The Code node filters for a live IPL match first — if IPL is not in season it falls back to any live T20 match for testing. It then detects which innings is active and computes all indicators — CRR, RRR, runs needed, overs remaining, wickets in hand, match phase, and pressure level. Both 1st and 2nd innings are handled with different prompt logic per scenario. |
| Narrative Generator | @n8n/n8n-nodes-langchain.openAi | Generates 2-sentence cricket narrative with GPT-4o-mini | Is Live Match ? | Parse Narrative | The computed indicators are assembled into a structured prompt and sent to GPT-4o via HTTP Request. The system prompt instructs the model to write exactly 2 sentences, use specific numbers, avoid percentages, and sound like a human cricket analyst. The response is parsed and passed to both the Google Sheets log and the Webhook response node so any app polling the webhook gets the narrative instantly. |
| Parse Narrative | n8n-nodes-base.code | Extracts narrative text and merges with computed data | Narrative Generator | Log to IPL Win Probability Log | The computed indicators are assembled into a structured prompt and sent to GPT-4o via HTTP Request. The system prompt instructs the model to write exactly 2 sentences, use specific numbers, avoid percentages, and sound like a human cricket analyst. The response is parsed and passed to both the Google Sheets log and the Webhook response node so any app polling the webhook gets the narrative instantly. |
| Log to IPL Win Probability Log | n8n-nodes-base.googleSheets | Appends generated narratives to Google Sheets | Parse Narrative | Respond to Webhook | Every narrative is logged with timestamp, match name, innings, over, score, target, RRR, CRR, phase, pressure, and the full narrative text. This creates a complete ball-by-ball record of every narrative generated across the match.  |
| Log to IPL Win Probability Log | n8n-nodes-base.googleSheets | Appends generated narratives to Google Sheets | Parse Narrative | Respond to Webhook | Webhook Response — returns a clean JSON payload that any fan app, widget, or dashboard can poll to display the latest narrative in real time. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns final JSON response to webhook callers | Log to IPL Win Probability Log |  | Every narrative is logged with timestamp, match name, innings, over, score, target, RRR, CRR, phase, pressure, and the full narrative text. This creates a complete ball-by-ball record of every narrative generated across the match.  |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns final JSON response to webhook callers | Log to IPL Win Probability Log |  | Webhook Response — returns a clean JSON payload that any fan app, widget, or dashboard can poll to display the latest narrative in real time. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Real-Time IPL Commentary Generator With Live API Data and GPT-4o Narratives`.

2. **Add the Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure a cron-based interval:
     - Expression: `*/6 14-23 * * *`
   - This runs every 6 minutes between 2 PM and 11 PM server time.

3. **Add the Webhook node**
   - Node type: `Webhook`
   - Name it `Manual test trigger`
   - Set:
     - Path: `ipl-narrative`
     - Response Mode: `Using Respond to Webhook Node` / `responseNode`
   - This provides an HTTP endpoint for manual testing or app polling.

4. **Add the CricAPI request node**
   - Node type: `HTTP Request`
   - Name it `Fetch live match list`
   - Configure:
     - Method: `GET`
     - URL: `https://api.cricapi.com/v1/currentMatches`
     - Send Query Parameters: enabled
     - Query parameters:
       - `apikey` = your CricAPI key
       - `offset` = `0`
   - Recommended improvement:
     - Store the API key in an n8n variable or credential rather than hardcoding it.

5. **Connect both triggers to the CricAPI request**
   - `Schedule Trigger` → `Fetch live match list`
   - `Manual test trigger` → `Fetch live match list`

6. **Add a Code node for cricket calculations**
   - Node type: `Code`
   - Name it `Compute Match Indicators`
   - Paste logic that:
     - validates CricAPI success status
     - finds a live IPL T20 match
     - falls back to live T20
     - falls back to any T20 with score
     - returns `{ skip: true }` if nothing is suitable
     - computes innings-sensitive fields:
       - batting and bowling team
       - score
       - overs
       - overs remaining
       - wickets in hand
       - current run rate
       - projected total in first innings
       - target, runs needed, required run rate, run-rate gap in second innings
       - phase (`Powerplay`, `Middle overs`, `Death overs`)
       - pressure (`Low`, `Medium`, `High`)
       - prompt for OpenAI
       - match name
       - timestamp
   - Connect:
     - `Fetch live match list` → `Compute Match Indicators`

7. **Add an If node**
   - Node type: `If`
   - Name it `Is Live Match ?`
   - Set a condition checking that skip is not true:
     - Left value: `={{ $json.skip }}`
     - Operator: `notEquals`
     - Right value: `true`
   - Enable loose type validation if available.
   - Connect:
     - `Compute Match Indicators` → `Is Live Match ?`

8. **Add the OpenAI node**
   - Node type: `OpenAI` from LangChain package
   - Name it `Narrative Generator`
   - Configure credentials:
     - Add an OpenAI API credential in n8n and select it here.
   - Set model:
     - `gpt-4o-mini`
   - Add messages:
     - **System message:**  
       `You are an expert IPL cricket analyst. You write short punchy win-probability narratives for fan apps. Always be specific about numbers. Never use percentages. Always write exactly 2 sentences.`
     - **User message/content:**  
       `={{ $json.prompt }}`
   - Connect the **true** output of `Is Live Match ?` to `Narrative Generator`.

9. **Add a Code node to parse the AI response**
   - Node type: `Code`
   - Name it `Parse Narrative`
   - Use logic that:
     - extracts the generated narrative from the OpenAI node output path
     - falls back to `Narrative unavailable.`
     - merges it with the original computed match indicators from `Compute Match Indicators`
   - Example logic behavior:
     - Read: `response.output?.[0]?.content?.[0]?.text`
     - Read original data from `$('Compute Match Indicators').first().json`
     - Return merged JSON with `narrative`
   - Connect:
     - `Narrative Generator` → `Parse Narrative`

10. **Prepare Google Sheets logging**
    - Create a Google Sheet, for example `IPL Win Probability Log`
    - Add headers in the first row:
      - `timestamp`
      - `match_name`
      - `innings`
      - `over`
      - `score`
      - `target`
      - `rrr`
      - `crr`
      - `phase`
      - `pressure`
      - `narrative`

11. **Add the Google Sheets node**
    - Node type: `Google Sheets`
    - Name it `Log to IPL Win Probability Log`
    - Authenticate with Google Sheets OAuth2 credentials.
    - Configure:
      - Operation: `Append`
      - Select the spreadsheet you created
      - Select the target sheet/tab
      - Use manual field mapping
    - Map fields:
      - `timestamp` ← `={{ $json.timestamp }}`
      - `match_name` ← `={{ $json.matchName }}`
      - `innings` ← `={{ $json.innings }}`
      - `over` ← `={{ $json.overs }}`
      - `score` ← `={{ $json.score }}`
      - `target` ← `={{ $json.target }}`
      - `rrr` ← `={{ $json.rrr }}`
      - `crr` ← `={{ $json.crr }}`
      - `phase` ← `={{ $json.phase }}`
      - `pressure` ← `={{ $json.pressure }}`
      - `narrative` ← `={{ $json.narrative }}`
    - Connect:
      - `Parse Narrative` → `Log to IPL Win Probability Log`

12. **Add the Respond to Webhook node**
    - Node type: `Respond to Webhook`
    - Name it `Respond to Webhook`
    - Configure:
      - Respond with: `JSON`
      - Body with fields for the consuming app
   - Recommended body:
     ```json
     {
       "match": "{{ $json.matchName }}",
       "innings": "{{ $json.innings }}",
       "score": "{{ $json.score }}",
       "over": "{{ $json.overs }}",
       "phase": "{{ $json.phase }}",
       "pressure": "{{ $json.pressure }}",
       "narrative": "{{ $json.narrative }}",
       "timestamp": "{{ $json.timestamp }}"
     }
     ```
   - Important: this recommended body fixes the field-name mismatch present in the source workflow.
   - Connect:
     - `Log to IPL Win Probability Log` → `Respond to Webhook`

13. **Test the webhook path**
   - Open the webhook test URL in n8n.
   - Trigger it with a browser or HTTP client.
   - Confirm:
     - CricAPI returns match data
     - indicators are computed
     - OpenAI generates exactly two sentences
     - Google Sheets receives a new row
     - webhook returns JSON

14. **Test scheduled execution**
   - Execute the workflow manually first.
   - Then activate the workflow.
   - Wait for the cron interval or temporarily shorten the schedule for validation.

15. **Handle no-match scenarios explicitly**
   - The original workflow stops if no match exists, with no false branch on the If node.
   - Recommended improvement:
     - Add a false-path node that returns a friendly webhook response such as:
       - `status: "skipped"`
       - `reason: "No T20 matches available"`
   - This avoids webhook timeouts during inactive periods.

16. **Secure credentials properly**
   - CricAPI key:
     - ideally stored as an environment variable or n8n variable, then referenced in the HTTP Request node
   - OpenAI:
     - configure via n8n OpenAI credentials
   - Google Sheets:
     - configure via OAuth2 and ensure the target spreadsheet is accessible to the authenticated account

17. **Optional hardening**
   - Add retry/error handling around CricAPI and OpenAI calls.
   - Add a validation step after AI generation to enforce two sentences.
   - Add deduplication if you do not want duplicate rows every 6 minutes with unchanged scores.
   - Add timezone documentation for the schedule.

### Sub-workflow setup
This workflow does not invoke any sub-workflows and does not contain an Execute Workflow node. No separate child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow Overview: This workflow turns live IPL match data into human-sounding win-probability narratives that update every 6 minutes throughout a match. It fetches live scores from CricAPI, computes key cricket indicators like required run rate, current run rate, wickets in hand, match phase, and pressure level, then sends all of that context to GPT-4o which generates a 2-sentence analyst-style narrative ready to embed in any fan app, widget, or dashboard. | General workflow description |
| Setup Steps: 1. Sign up at cricapi.com and get your API key 2. Go to Settings → Variables and add CRICAPI_KEY 3. Add your OpenAI API key to the HTTP Request node 4. Create the Google Sheet with columns listed below 5. Connect your Google Sheets OAuth2 credentials 6. Activate the workflow — it runs automatically every 6 minutes during match hours (2PM–11PM) 7. Test anytime via the webhook URL | Operational setup note |
| Two triggers run this workflow. The Schedule Trigger fires automatically every 6 minutes between 2PM and 11PM covering all IPL match windows. The Webhook Trigger lets you fire it manually at any time for testing or on-demand narrative generation without waiting for the schedule. Both triggers feed into the same API call so the logic is identical. | Trigger behavior |
| One API call to CricAPI fetches all current matches. The Code node filters for a live IPL match first — if IPL is not in season it falls back to any live T20 match for testing. It then detects which innings is active and computes all indicators — CRR, RRR, runs needed, overs remaining, wickets in hand, match phase, and pressure level. Both 1st and 2nd innings are handled with different prompt logic per scenario. | Match selection and metric derivation |
| The computed indicators are assembled into a structured prompt and sent to GPT-4o via HTTP Request. The system prompt instructs the model to write exactly 2 sentences, use specific numbers, avoid percentages, and sound like a human cricket analyst. The response is parsed and passed to both the Google Sheets log and the Webhook response node so any app polling the webhook gets the narrative instantly. | AI generation and output usage |
| Every narrative is logged with timestamp, match name, innings, over, score, target, RRR, CRR, phase, pressure, and the full narrative text. This creates a complete ball-by-ball record of every narrative generated across the match. | Logging behavior |
| Webhook Response — returns a clean JSON payload that any fan app, widget, or dashboard can poll to display the latest narrative in real time. | API response behavior |

## Important implementation notes
- The sticky note says to add `CRICAPI_KEY` as a variable, but the actual HTTP Request node uses a hardcoded placeholder `your_api_key`. To make the workflow truly portable, replace that with a variable or credential-backed expression.
- The sticky note mentions adding the OpenAI API key to an HTTP Request node, but the actual workflow uses the dedicated OpenAI node, not an HTTP Request to OpenAI.
- The sticky notes mention Google Sheets logging and a “ball-by-ball record,” but the workflow runs every 6 minutes, not every ball.
- The `Respond to Webhook` node currently references `match_name` and `over`, while upstream generated data uses `matchName` and `overs`. This is a schema inconsistency and should be corrected.
- If triggered by webhook during a period with no valid T20 match, the current workflow may not send any response because the false branch of `Is Live Match ?` is not wired.