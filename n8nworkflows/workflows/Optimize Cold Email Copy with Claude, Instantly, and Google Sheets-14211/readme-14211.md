Optimize Cold Email Copy with Claude, Instantly, and Google Sheets

https://n8nworkflows.xyz/workflows/optimize-cold-email-copy-with-claude--instantly--and-google-sheets-14211


# Optimize Cold Email Copy with Claude, Instantly, and Google Sheets

# 1. Workflow Overview

This workflow implements a self-improving cold email optimization loop. Its purpose is to continuously evaluate A/B cold email performance using live campaign metrics from Instantly, historical experiment data from Google Sheets, and rule constraints from a program sheet, then use Claude to decide a winner and generate the next challenger.

Typical use cases include:

- Continuous optimization of outbound cold email copy
- Automated champion/challenger testing
- Guardrailed AI-assisted copy iteration
- Maintaining an experiment memory across multiple rounds
- Pausing automatically when reply performance drops below a safety threshold

## 1.1 Scheduled Trigger

The workflow starts automatically every 6 hours.

## 1.2 Data Collection

It retrieves:

- campaign analytics from Instantly
- active champion/challenger copy from the `Active` Google Sheets tab
- past experiments from the `Experiments` tab
- program constraints/rules from the `Program` tab

## 1.3 Data Consolidation and Decisioning

A Code node parses the various data sources into a normalized structure, derives evaluation flags, and determines one of four actions:

- `wait`
- `generate_first_challenger`
- `evaluate_and_generate`
- `safety_pause`

## 1.4 Routing and Safety Control

If there is insufficient data, the workflow exits. If both variants are under the reply-rate safety floor, it sends a Telegram alert and does not continue experimentation. Otherwise, it proceeds to Claude.

## 1.5 AI Evaluation and Challenger Generation

Claude receives:

- champion/challenger copy and metrics
- recent experiment history
- current action
- program rules

It returns structured JSON containing the winner, reasoning, hypothesis, change type, and the next challenger.

## 1.6 Logging and Deployment

The workflow logs the experiment to Google Sheets, updates the active champion/challenger sheet, and deploys the new A/B copy back into Instantly.

---

# 2. Block-by-Block Analysis

## 2.1 Workflow Context and Embedded Documentation

**Overview:**  
This block consists of sticky notes that document the workflow’s purpose, architecture, dependencies, and operator expectations. These notes are not executable, but they are important for maintainability and for understanding the intended sheet schema and environment variables.

**Nodes Involved:**

- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node
- **Configuration choices:** Large overview note describing the optimization loop, Sheets structure, required accounts, and environment variables
- **Key expressions or variables used:** None in execution; content references:
  - `OPTIMIZER_SHEET_ID`
  - `REMOVEFAST_CAMPAIGN_ID`
  - `INSTANTLY_API_KEY`
  - `ANTHROPIC_API_KEY`
  - `TELEGRAM_BOT_TOKEN`
  - `TELEGRAM_CHAT_ID`
  - `REPLY_RATE_FLOOR`
- **Input and output connections:** None
- **Version-specific requirements:** Sticky Note type version 1
- **Edge cases or potential failure types:** None at runtime; documentation may become outdated if the workflow changes
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** Sticky note for the data collection area
- **Configuration choices:** Explains that live stats, active variants, history, and program rules are fetched
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** Sticky note for threshold/routing logic
- **Configuration choices:** Documents that 200 sends per variant are required, and that there are three logical outcomes: first run, evaluate, or wait
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** Sticky note for AI evaluation and safety
- **Configuration choices:** Explains Claude analysis and safety floor behavior
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** Sticky note for logging/deployment
- **Configuration choices:** Describes history logging, winner promotion, and deployment of the new challenger
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Type version 1
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 2.2 Scheduled Entry Point

**Overview:**  
This block is the workflow’s single entry point. It triggers the optimization loop every 6 hours and ensures the run begins even if there is no inbound payload.

**Nodes Involved:**

- `Run Every 6 Hours`

### Node Details

#### Run Every 6 Hours
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; periodic workflow trigger
- **Configuration choices:** Interval schedule set to every 6 hours
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - **Outputs to:** `Pull Instantly Campaign Stats`
- **Version-specific requirements:** Type version 1.2
- **Edge cases or potential failure types:**
  - Workflow must be active for schedule execution
  - Timezone behavior depends on instance settings
  - If n8n is down during the planned time, execution may be skipped depending on deployment mode
- **Sub-workflow reference:** None

---

## 2.3 Data Collection

**Overview:**  
This block pulls the current operational data required for evaluation: campaign analytics from Instantly and three Google Sheets ranges representing the active test, experiment history, and program constraints.

**Nodes Involved:**

- `Pull Instantly Campaign Stats`
- `Read Experiment History`
- `Read Active Variants`
- `Read Program Doc`

### Node Details

#### Pull Instantly Campaign Stats
- **Type and technical role:** `n8n-nodes-base.httpRequest`; REST API call to Instantly analytics endpoint
- **Configuration choices:**
  - GET request to `https://api.instantly.ai/api/v2/campaigns/{{ $env.REMOVEFAST_CAMPAIGN_ID }}/analytics`
  - Authorization header uses bearer token from environment variable
- **Key expressions or variables used:**
  - `{{$env.REMOVEFAST_CAMPAIGN_ID}}`
  - `{{$env.INSTANTLY_API_KEY}}`
- **Input and output connections:**  
  - **Input from:** `Run Every 6 Hours`
  - **Outputs to:** `Read Experiment History`, `Read Active Variants`, `Read Program Doc`
- **Version-specific requirements:** HTTP Request type version 4.2
- **Edge cases or potential failure types:**
  - Missing or invalid `INSTANTLY_API_KEY`
  - Missing or invalid campaign ID
  - API schema drift if Instantly changes analytics response format
  - Rate limiting or 5xx API failures
  - The node notes mention open/reply/click rate, but downstream code does not currently rely on specific fields from this response
- **Sub-workflow reference:** None

#### Read Experiment History
- **Type and technical role:** HTTP Request to Google Sheets API
- **Configuration choices:**
  - GET request to `values/Experiments!A:M`
  - Uses generic header authentication credential
  - Reads columns A through M
- **Key expressions or variables used:**
  - `{{$env.OPTIMIZER_SHEET_ID}}`
- **Input and output connections:**  
  - **Input from:** `Pull Instantly Campaign Stats`
  - **Outputs to:** `Merge Stats and History`
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - Invalid Sheets credential or expired access token
  - Missing tab named `Experiments`
  - Column layout mismatch
  - If the sheet is empty or only has headers, downstream history becomes an empty array
- **Sub-workflow reference:** None

#### Read Active Variants
- **Type and technical role:** HTTP Request to Google Sheets API
- **Configuration choices:**
  - GET request to `values/Active!A:F`
  - Reads active champion/challenger rows
  - Uses generic header authentication
- **Key expressions or variables used:**
  - `{{$env.OPTIMIZER_SHEET_ID}}`
- **Input and output connections:**  
  - **Input from:** `Pull Instantly Campaign Stats`
  - **Outputs to:** `Merge Stats and History`
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - Missing `Active` tab
  - Missing required rows `champion` and `challenger`
  - Data type inconsistencies in sends/open/reply columns
  - If challenger body is blank, the workflow treats it as no challenger
- **Sub-workflow reference:** None

#### Read Program Doc
- **Type and technical role:** HTTP Request to Google Sheets API
- **Configuration choices:**
  - GET request to `values/Program!A:B`
  - Reads a rule table and concatenates it later into prompt text
  - Uses generic header authentication
- **Key expressions or variables used:**
  - `{{$env.OPTIMIZER_SHEET_ID}}`
- **Input and output connections:**  
  - **Input from:** `Pull Instantly Campaign Stats`
  - **Outputs to:** `Merge Stats and History`
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - Missing `Program` tab
  - Blank or malformed rules
  - If headers only exist, Claude receives an empty rules string
- **Sub-workflow reference:** None

---

## 2.4 Data Consolidation and Threshold Evaluation

**Overview:**  
This block merges all fetched datasets, parses rows into normalized objects, computes thresholds and safety flags, then determines which action the workflow should take next.

**Nodes Involved:**

- `Merge Stats and History`
- `Check Data Threshold`

### Node Details

#### Merge Stats and History
- **Type and technical role:** `n8n-nodes-base.code`; aggregation and transformation logic
- **Configuration choices:**
  - Reads the first item from each upstream node using `$('<Node Name>').first().json`
  - Parses:
    - active variants
    - recent experiment history
    - program rules
  - Creates normalized `champion` and `challenger` objects
  - Calculates boolean flags:
    - `bothHaveEnoughData`
    - `hasChallenger`
    - `belowFloor`
  - Uses environment variable for reply-rate floor with fallback `1.0`
- **Key expressions or variables used:**
  - `$('Pull Instantly Campaign Stats').first().json`
  - `$('Read Active Variants').first().json`
  - `$('Read Experiment History').first().json`
  - `$('Read Program Doc').first().json`
  - `$env.REPLY_RATE_FLOOR`
- **Input and output connections:**  
  - **Inputs from:** `Read Experiment History`, `Read Active Variants`, `Read Program Doc`
  - **Outputs to:** `Check Data Threshold`
- **Version-specific requirements:** Code node type version 2
- **Edge cases or potential failure types:**
  - If any upstream node fails, this node cannot safely merge the expected data
  - `parseInt` / `parseFloat` will coerce invalid values to `NaN`, then fall back to `0`
  - `belowFloor` requires champion sends `>= 200`, but not challenger sends `>= 200`; this is an intentional or accidental asymmetry worth reviewing
  - The history mapping assumes exact column positions; any sheet schema drift breaks semantic accuracy
  - Uses the last 10 rows as “recent history” without sorting by date, assuming the sheet is append-only in chronological order
- **Sub-workflow reference:** None

#### Check Data Threshold
- **Type and technical role:** Code node for action selection
- **Configuration choices:**
  - Routes into one of four actions:
    - `safety_pause`
    - `generate_first_challenger`
    - `evaluate_and_generate`
    - `wait`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `data.belowFloor`
  - `data.hasChallenger`
  - `data.bothHaveEnoughData`
- **Input and output connections:**  
  - **Input from:** `Merge Stats and History`
  - **Outputs to:** `Route by Action`
- **Version-specific requirements:** Code node version 2
- **Edge cases or potential failure types:**
  - If `hasChallenger` is false because the challenger body is empty but a subject exists, it still treats it as first-run generation
  - The threshold is hardcoded at 200 sends in the prior node, not parameterized
  - No statistical significance logic beyond send-count threshold and metric comparison
- **Sub-workflow reference:** None

---

## 2.5 Action Routing and Safety Branching

**Overview:**  
This block enforces the early exit on insufficient data and then assigns the execution route either to Telegram alerting or AI processing.

**Nodes Involved:**

- `Route by Action`
- `Safety Floor Check`
- `Telegram Safety Alert`

### Node Details

#### Route by Action
- **Type and technical role:** Code node used as a filter
- **Configuration choices:**
  - If action is `wait`, it returns no items and effectively stops the workflow
  - Otherwise it passes the input through unchanged
- **Key expressions or variables used:**
  - `$input.first().json`
  - `data.action`
- **Input and output connections:**  
  - **Input from:** `Check Data Threshold`
  - **Outputs to:** `Safety Floor Check`
- **Version-specific requirements:** Code node version 2
- **Edge cases or potential failure types:**
  - Returning an empty array is a silent exit; operators must inspect executions to understand why nothing happened
  - Because `alwaysOutputData` is set, behavior still depends on code return value, not on node configuration alone
- **Sub-workflow reference:** None

#### Safety Floor Check
- **Type and technical role:** Code node for branch marking
- **Configuration choices:**
  - Adds `_route: 'telegram'` for `safety_pause`
  - Adds `_route: 'claude'` for everything else
- **Key expressions or variables used:**
  - `$input.first().json`
  - `data.action`
- **Input and output connections:**  
  - **Input from:** `Route by Action`
  - **Outputs to:** `Claude Evaluate and Generate`, `Telegram Safety Alert`
- **Version-specific requirements:** Code node version 2
- **Edge cases or potential failure types:**
  - This node does not actually prevent both downstream nodes from receiving the same item; because both are connected, both nodes are triggered
  - As currently designed, the `_route` field is informational only, since no IF/Switch node filters execution
  - Result: the Telegram node and Claude node may both execute for nonmatching routes unless each service tolerates or ignores the payload
- **Sub-workflow reference:** None

#### Telegram Safety Alert
- **Type and technical role:** HTTP Request to Telegram Bot API
- **Configuration choices:**
  - POST to `https://api.telegram.org/bot{{ $env.TELEGRAM_BOT_TOKEN }}/sendMessage`
  - Sends JSON body with:
    - `chat_id`
    - `text`
    - `parse_mode`
- **Key expressions or variables used:**
  - `{{$env.TELEGRAM_BOT_TOKEN}}`
  - `{{$env.TELEGRAM_CHAT_ID}}`
  - `{{$json.replyRateFloor}}`
  - `{{$json.champion.replyRate}}`
  - `{{$json.champion.sends}}`
  - `{{$json.challenger.replyRate}}`
  - `{{$json.challenger.sends}}`
- **Input and output connections:**  
  - **Input from:** `Safety Floor Check`
  - **Outputs to:** none
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - Invalid Telegram bot token or chat ID
  - Misleading `parse_mode: HTML` since the text is plain text and does not currently include HTML formatting
  - Because there is no true route filter, this node may send alerts even when `_route` is `claude`
- **Sub-workflow reference:** None

**Important implementation note:**  
This is the most significant logic issue in the workflow. The `Safety Floor Check` node marks a route but does not branch execution. In n8n, connecting one node to two downstream nodes causes both to receive the item. To achieve the intended behavior, replace this pattern with an IF or Switch node keyed on `action` or `_route`.

---

## 2.6 AI Evaluation and Response Parsing

**Overview:**  
This block sends the current optimization state to Anthropic Claude and parses its structured response into fields suitable for logging and deployment.

**Nodes Involved:**

- `Claude Evaluate and Generate`
- `Parse Claude Response`

### Node Details

#### Claude Evaluate and Generate
- **Type and technical role:** HTTP Request to Anthropic Messages API
- **Configuration choices:**
  - POST to `https://api.anthropic.com/v1/messages`
  - Headers:
    - `x-api-key: {{$env.ANTHROPIC_API_KEY}}`
    - `anthropic-version: 2023-06-01`
    - `Content-Type: application/json`
  - Body includes:
    - model `claude-sonnet-4-6`
    - `max_tokens: 2000`
    - one user message containing full prompt and strict output JSON schema
- **Key expressions or variables used:**
  - `{{$env.ANTHROPIC_API_KEY}}`
  - `{{$json.programRules}}`
  - `{{$json.action}}`
  - `{{$json.champion.*}}`
  - `{{$json.challenger.*}}`
  - `{{ JSON.stringify($json.experimentHistory) }}`
  - `{{$json.totalExperiments}}`
- **Input and output connections:**  
  - **Input from:** `Safety Floor Check`
  - **Outputs to:** `Parse Claude Response`
- **Version-specific requirements:** HTTP Request 4.2; compatibility depends on Anthropic API model naming remaining valid
- **Edge cases or potential failure types:**
  - Invalid or missing `ANTHROPIC_API_KEY`
  - Model name `claude-sonnet-4-6` may not exist or may change in Anthropic availability
  - Prompt asks for strict JSON, but LLMs can still add prose or malformed JSON
  - Large prompt risk if program rules or history become too verbose, though capped to last 10 rounds
  - The generated body explicitly expects `{{personalization}}`; if omitted by Claude, deployment may break your campaign assumptions
  - Because of the routing issue above, Claude may run even during a safety pause
- **Sub-workflow reference:** None

#### Parse Claude Response
- **Type and technical role:** Code node for response extraction and normalization
- **Configuration choices:**
  - Reads `data.content?.[0]?.text`
  - Extracts the first JSON-like block using regex `/\{[\s\S]*\}/`
  - Parses it with `JSON.parse`
  - Retrieves prior workflow state from `Check Data Threshold`
  - Outputs normalized fields for downstream sheet logging and deployment
- **Key expressions or variables used:**
  - `$input.first().json`
  - `data.content?.[0]?.text`
  - `$('Check Data Threshold').first().json`
- **Input and output connections:**  
  - **Input from:** `Claude Evaluate and Generate`
  - **Outputs to:** `Log Experiment to History`, `Update Active Variants`
- **Version-specific requirements:** Code node version 2
- **Edge cases or potential failure types:**
  - If Claude returns no `content[0].text`, parsing fails
  - Regex may capture unintended braces if the model returns extra text with JSON-like fragments
  - `jsonMatch[0]` can throw if no match is found
  - No validation exists for required fields such as `winner`, `new_challenger_subject`, or `new_challenger_body`
  - If `winner` is `none`, downstream logic still promotes the previous champion by default because only `challenger` causes a switch
- **Sub-workflow reference:** None

---

## 2.7 Logging Results and Updating Active State

**Overview:**  
This block stores the completed experiment in the history sheet and updates the active sheet with the promoted champion and the newly generated challenger.

**Nodes Involved:**

- `Log Experiment to History`
- `Update Active Variants`

### Node Details

#### Log Experiment to History
- **Type and technical role:** HTTP Request to Google Sheets append endpoint
- **Configuration choices:**
  - POST to `values/Experiments!A:M:append?valueInputOption=USER_ENTERED`
  - Appends a single row containing experiment metadata and AI reasoning
- **Key expressions or variables used:**
  - `{{$env.OPTIMIZER_SHEET_ID}}`
  - `{{$json.experimentNumber}}`
  - `{{$json.timestamp}}`
  - `{{$json.previousChampion.*}}`
  - `{{$json.previousChallenger.*}}`
  - `{{$json.winner}}`
  - `{{$json.changeType}}`
  - `{{$json.analysis}}`
  - `{{$json.hypothesis}}`
- **Input and output connections:**  
  - **Input from:** `Parse Claude Response`
  - **Outputs to:** `Update Instantly Campaign Copy`
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - Google Sheets auth failure
  - Tab/column mismatch if the schema changes
  - Long analysis text may make cells hard to read but should still be accepted within Sheets limits
  - The node note says change type is in column L, but the overview sticky note described Experiments as A:M with Change Type and Reasoning; the actual append order places `Winner` in K, `Change Type` in L, `Reasoning` in M, which is consistent with the node
- **Sub-workflow reference:** None

#### Update Active Variants
- **Type and technical role:** HTTP Request to Google Sheets update endpoint
- **Configuration choices:**
  - PUT to `values/Active!A2:F3?valueInputOption=USER_ENTERED`
  - Rewrites rows 2 and 3:
    - row 2 becomes champion
    - row 3 becomes challenger
  - If winner is `challenger`, previous challenger is promoted; otherwise previous champion remains champion
  - Counters reset to zero
- **Key expressions or variables used:**
  - `{{$env.OPTIMIZER_SHEET_ID}}`
  - ternary expressions based on `{{$json.winner === 'challenger' ? ... : ...}}`
  - `{{$json.newChallengerSubject}}`
  - `{{$json.newChallengerBody}}`
- **Input and output connections:**  
  - **Input from:** `Parse Claude Response`
  - **Outputs to:** `Update Instantly Campaign Copy`
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - Requires the `Active` tab to have the expected layout
  - Resets sends/open/reply metrics to zero regardless of external truth; this assumes the sheet is purely local experiment state
  - If `winner` is `none`, champion remains unchanged and a new challenger still gets introduced
- **Sub-workflow reference:** None

---

## 2.8 Deployment Back to Instantly

**Overview:**  
This final block pushes the promoted champion and the new challenger into the Instantly campaign as two sequences, corresponding to the next A/B test round.

**Nodes Involved:**

- `Update Instantly Campaign Copy`

### Node Details

#### Update Instantly Campaign Copy
- **Type and technical role:** HTTP Request to Instantly API
- **Configuration choices:**
  - POST to `https://api.instantly.ai/api/v2/campaigns/{{ $env.REMOVEFAST_CAMPAIGN_ID }}/sequences`
  - Sends two sequence objects, each with one step:
    - champion sequence
    - challenger sequence
  - Authorization via bearer token in header
- **Key expressions or variables used:**
  - `{{$env.REMOVEFAST_CAMPAIGN_ID}}`
  - `{{$env.INSTANTLY_API_KEY}}`
  - ternary expressions selecting the promoted winner copy
  - `{{$json.newChallengerSubject}}`
  - `{{$json.newChallengerBody}}`
- **Input and output connections:**  
  - **Inputs from:** `Update Active Variants`, `Log Experiment to History`
  - **Outputs to:** none
- **Version-specific requirements:** HTTP Request 4.2
- **Edge cases or potential failure types:**
  - This node has two incoming connections, so it may execute twice per workflow run unless n8n execution semantics are carefully managed
  - If triggered once from each parent with separate items, you risk duplicate deployment calls
  - Instantly endpoint schema must match the exact `sequences` structure expected by your account/campaign type
  - Invalid API key or campaign ID
  - Generated copy may include characters or variable syntax that Instantly rejects
- **Sub-workflow reference:** None

**Important implementation note:**  
This node is connected from both `Log Experiment to History` and `Update Active Variants`. Without an explicit Merge/Wait-for-Both pattern, this can cause duplicate Instantly update calls. A safer design would merge both branches first, then call Instantly once.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Visual overview of workflow purpose, setup, sheet schema, env vars |  |  | ## Self-Improving Cold Email Copy Optimizer<br>Inspired by [Karpathy's Autoresearch](https://github.com/karpathy/autoresearch), this workflow treats your cold email campaigns as continuous experiments. Instead of writing copy once and hoping for the best, it runs an automated optimization loop that evolves your emails based on real performance data.<br>### How it works<br>1. Runs on a schedule (every 6 hours)<br>2. Pulls live campaign stats (open rate, reply rate) from Instantly<br>3. Reads the current champion and challenger variants from Google Sheets<br>4. Reads the full experiment history (last 10 rounds)<br>5. Checks if both variants have enough data (200+ sends each)<br>6. If below threshold, exits and waits for the next run<br>7. If above threshold, feeds everything to Claude: both variants, metrics, experiment history, and the cold email program rules<br>8. Claude declares a winner, explains why, and generates a new challenger with a specific hypothesis<br>9. Safety check: if reply rate drops below the floor threshold, pauses and sends a Telegram alert instead of continuing<br>10. Logs the full experiment (variants, metrics, winner, reasoning, change type) to Google Sheets<br>11. Promotes winner, deploys new challenger to Instantly<br>### What makes it different<br>- **Learns across rounds, not just the current test.** Claude sees the last 10 experiments and identifies patterns over time.<br>- **Program doc driven.** Brand voice, hard rules, and constraints live in a Google Sheet, not hardcoded in prompts. Update the rules without touching the workflow.<br>- **Structured change tracking.** Every experiment logs what specifically changed (subject, CTA, tone, length, opener style) so you can filter by change type.<br>- **Safety floor.** If reply rate drops below your minimum, the system pauses and alerts you instead of running bad copy.<br>### What you need<br>- Instantly account with active campaign<br>- Claude API key (Anthropic)<br>- Google Sheets (two tabs: Active, Experiments + one tab: Program)<br>- Telegram bot (for safety alerts)<br>- Enough send volume for meaningful data (200+ sends per variant)<br>### Google Sheets setup<br>**Active tab** (A:F): Variant \| Subject Line \| Email Body \| Sends \| Open Rate \| Reply Rate<br>**Experiments tab** (A:M): Experiment ID \| Date \| Champion Copy \| Challenger Copy \| Champion Subject \| Challenger Subject \| Champion Sends \| Challenger Sends \| Champion Reply Rate \| Challenger Reply Rate \| Winner \| Change Type \| Reasoning<br>**Program tab** (A:B): Rule Name \| Rule Content (brand voice, constraints, off-limits topics, max word count, required variables, etc.)<br>### Environment variables<br>- OPTIMIZER_SHEET_ID<br>- REMOVEFAST_CAMPAIGN_ID<br>- INSTANTLY_API_KEY<br>- ANTHROPIC_API_KEY<br>- TELEGRAM_BOT_TOKEN<br>- TELEGRAM_CHAT_ID<br>- REPLY_RATE_FLOOR (default: 1.0) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual note for data collection block |  |  | ### Data Collection<br>Pulls live stats from Instantly and reads current variants + experiment history from Google Sheets. Also reads the Program doc for Claude's rules. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual note for threshold/routing block |  |  | ### Threshold Check + Routing<br>Needs 200+ sends per variant before evaluating. If not enough data, the workflow exits cleanly. Three paths: first run, evaluate, or wait. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual note for AI and safety block |  |  | ### AI Evaluation + Safety<br>Claude analyzes both variants against 10 rounds of history. Generates new challenger with a hypothesis. Safety floor check pauses the loop if reply rate tanks. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual note for logging and deployment block |  |  | ### Log + Deploy<br>Logs full experiment to history sheet, promotes winner, deploys new challenger to Instantly. Every round compounds the knowledge base. |
| Run Every 6 Hours | n8n-nodes-base.scheduleTrigger | Scheduled entry point |  | Pull Instantly Campaign Stats |  |
| Pull Instantly Campaign Stats | n8n-nodes-base.httpRequest | Fetch Instantly campaign analytics | Run Every 6 Hours | Read Experiment History; Read Active Variants; Read Program Doc | ### Data Collection<br>Pulls live stats from Instantly and reads current variants + experiment history from Google Sheets. Also reads the Program doc for Claude's rules. |
| Read Experiment History | n8n-nodes-base.httpRequest | Read experiment log from Google Sheets | Pull Instantly Campaign Stats | Merge Stats and History | ### Data Collection<br>Pulls live stats from Instantly and reads current variants + experiment history from Google Sheets. Also reads the Program doc for Claude's rules. |
| Read Active Variants | n8n-nodes-base.httpRequest | Read current champion/challenger from Google Sheets | Pull Instantly Campaign Stats | Merge Stats and History | ### Data Collection<br>Pulls live stats from Instantly and reads current variants + experiment history from Google Sheets. Also reads the Program doc for Claude's rules. |
| Read Program Doc | n8n-nodes-base.httpRequest | Read program rules from Google Sheets | Pull Instantly Campaign Stats | Merge Stats and History | ### Data Collection<br>Pulls live stats from Instantly and reads current variants + experiment history from Google Sheets. Also reads the Program doc for Claude's rules. |
| Merge Stats and History | n8n-nodes-base.code | Normalize sheets/API data and compute decision flags | Read Experiment History; Read Active Variants; Read Program Doc | Check Data Threshold | ### Data Collection<br>Pulls live stats from Instantly and reads current variants + experiment history from Google Sheets. Also reads the Program doc for Claude's rules. |
| Check Data Threshold | n8n-nodes-base.code | Select workflow action based on threshold/safety state | Merge Stats and History | Route by Action | ### Threshold Check + Routing<br>Needs 200+ sends per variant before evaluating. If not enough data, the workflow exits cleanly. Three paths: first run, evaluate, or wait. |
| Route by Action | n8n-nodes-base.code | Exit on wait, pass through actionable runs | Check Data Threshold | Safety Floor Check | ### Threshold Check + Routing<br>Needs 200+ sends per variant before evaluating. If not enough data, the workflow exits cleanly. Three paths: first run, evaluate, or wait. |
| Safety Floor Check | n8n-nodes-base.code | Mark route for Telegram or Claude | Route by Action | Claude Evaluate and Generate; Telegram Safety Alert | ### AI Evaluation + Safety<br>Claude analyzes both variants against 10 rounds of history. Generates new challenger with a hypothesis. Safety floor check pauses the loop if reply rate tanks. |
| Telegram Safety Alert | n8n-nodes-base.httpRequest | Send pause alert to Telegram | Safety Floor Check |  | ### AI Evaluation + Safety<br>Claude analyzes both variants against 10 rounds of history. Generates new challenger with a hypothesis. Safety floor check pauses the loop if reply rate tanks. |
| Claude Evaluate and Generate | n8n-nodes-base.httpRequest | Ask Claude to evaluate winner and produce next challenger | Safety Floor Check | Parse Claude Response | ### AI Evaluation + Safety<br>Claude analyzes both variants against 10 rounds of history. Generates new challenger with a hypothesis. Safety floor check pauses the loop if reply rate tanks. |
| Parse Claude Response | n8n-nodes-base.code | Extract structured JSON from Claude output | Claude Evaluate and Generate | Log Experiment to History; Update Active Variants | ### AI Evaluation + Safety<br>Claude analyzes both variants against 10 rounds of history. Generates new challenger with a hypothesis. Safety floor check pauses the loop if reply rate tanks. |
| Log Experiment to History | n8n-nodes-base.httpRequest | Append completed experiment to history sheet | Parse Claude Response | Update Instantly Campaign Copy | ### Log + Deploy<br>Logs full experiment to history sheet, promotes winner, deploys new challenger to Instantly. Every round compounds the knowledge base. |
| Update Active Variants | n8n-nodes-base.httpRequest | Rewrite active champion/challenger rows in Google Sheets | Parse Claude Response | Update Instantly Campaign Copy | ### Log + Deploy<br>Logs full experiment to history sheet, promotes winner, deploys new challenger to Instantly. Every round compounds the knowledge base. |
| Update Instantly Campaign Copy | n8n-nodes-base.httpRequest | Push champion and challenger sequences to Instantly | Log Experiment to History; Update Active Variants |  | ### Log + Deploy<br>Logs full experiment to history sheet, promotes winner, deploys new challenger to Instantly. Every round compounds the knowledge base. |

---

# 4. Reproducing the Workflow from Scratch

Below is a manual reconstruction sequence for n8n.

## 4.1 Prepare prerequisites

1. **Create the required environment variables** in your n8n instance:
   - `OPTIMIZER_SHEET_ID`
   - `REMOVEFAST_CAMPAIGN_ID`
   - `INSTANTLY_API_KEY`
   - `ANTHROPIC_API_KEY`
   - `TELEGRAM_BOT_TOKEN`
   - `TELEGRAM_CHAT_ID`
   - `REPLY_RATE_FLOOR` with a value like `1.0`

2. **Prepare Google Sheets** with one spreadsheet containing three tabs:
   - `Active`
   - `Experiments`
   - `Program`

3. **Set up the `Active` tab** with columns `A:F`:
   - `Variant`
   - `Subject Line`
   - `Email Body`
   - `Sends`
   - `Open Rate`
   - `Reply Rate`

4. **Set up the `Experiments` tab** with columns `A:M`:
   - `Experiment ID`
   - `Date`
   - `Champion Copy`
   - `Challenger Copy`
   - `Champion Subject`
   - `Challenger Subject`
   - `Champion Sends`
   - `Challenger Sends`
   - `Champion Reply Rate`
   - `Challenger Reply Rate`
   - `Winner`
   - `Change Type`
   - `Reasoning`

5. **Set up the `Program` tab** with columns `A:B`:
   - `Rule Name`
   - `Rule Content`

6. **Create Google Sheets API authentication** for HTTP requests.
   - This workflow uses a generic HTTP Header Auth credential.
   - In practice, you typically need an `Authorization: Bearer <access_token>` header from Google OAuth2 or another valid Google access mechanism.
   - If you prefer, you may replace these HTTP Request nodes with native Google Sheets nodes, but the original workflow uses HTTP Request.

7. **Verify Instantly API access** for:
   - campaign analytics endpoint
   - campaign sequence update endpoint

8. **Verify Anthropic API access** for the Messages API.

9. **Verify Telegram bot access** and confirm your bot can message the target chat ID.

---

## 4.2 Create documentation sticky notes

10. Add a **Sticky Note** node and paste the overall workflow description, including:
    - purpose
    - sheet schemas
    - environment variables
    - dependency list

11. Add a second sticky note labeled **Data Collection** near the data nodes.

12. Add a third sticky note labeled **Threshold Check + Routing** near the threshold nodes.

13. Add a fourth sticky note labeled **AI Evaluation + Safety** near Claude and Telegram nodes.

14. Add a fifth sticky note labeled **Log + Deploy** near the final nodes.

These are optional for execution but useful for maintainability.

---

## 4.3 Build the trigger and data collection section

15. Add a **Schedule Trigger** node named `Run Every 6 Hours`.
    - Set interval rule to every `6` hours.
    - Leave it as the workflow entry point.

16. Add an **HTTP Request** node named `Pull Instantly Campaign Stats`.
    - Method: `GET`
    - URL:
      `https://api.instantly.ai/api/v2/campaigns/{{ $env.REMOVEFAST_CAMPAIGN_ID }}/analytics`
    - Enable headers.
    - Add header:
      - `Authorization` = `Bearer {{ $env.INSTANTLY_API_KEY }}`
    - Connect `Run Every 6 Hours` → `Pull Instantly Campaign Stats`

17. Add an **HTTP Request** node named `Read Experiment History`.
    - Method: `GET`
    - URL:
      `https://sheets.googleapis.com/v4/spreadsheets/{{ $env.OPTIMIZER_SHEET_ID }}/values/Experiments!A:M`
    - Authentication: Generic Credential Type
    - Generic Auth Type: HTTP Header Auth
    - Attach your Google header auth credential
    - Connect `Pull Instantly Campaign Stats` → `Read Experiment History`

18. Add an **HTTP Request** node named `Read Active Variants`.
    - Method: `GET`
    - URL:
      `https://sheets.googleapis.com/v4/spreadsheets/{{ $env.OPTIMIZER_SHEET_ID }}/values/Active!A:F`
    - Authentication: Generic Credential Type
    - Generic Auth Type: HTTP Header Auth
    - Use the same Google credential
    - Connect `Pull Instantly Campaign Stats` → `Read Active Variants`

19. Add an **HTTP Request** node named `Read Program Doc`.
    - Method: `GET`
    - URL:
      `https://sheets.googleapis.com/v4/spreadsheets/{{ $env.OPTIMIZER_SHEET_ID }}/values/Program!A:B`
    - Authentication: Generic Credential Type
    - Generic Auth Type: HTTP Header Auth
    - Use the same Google credential
    - Connect `Pull Instantly Campaign Stats` → `Read Program Doc`

---

## 4.4 Build the merge and threshold logic

20. Add a **Code** node named `Merge Stats and History`.

21. Paste this logic conceptually into the node:
    - Read the first JSON item from:
      - `Pull Instantly Campaign Stats`
      - `Read Active Variants`
      - `Read Experiment History`
      - `Read Program Doc`
    - Parse `Active` rows after the header row
    - Find rows where column A is `champion` and `challenger`
    - Parse the last 10 experiment rows from `Experiments`
    - Map history fields to:
      - experimentId
      - date
      - championCopy
      - challengerCopy
      - championSubject
      - challengerSubject
      - championSends
      - challengerSends
      - championReplyRate
      - challengerReplyRate
      - winner
      - changeType
      - reasoning
    - Join `Program` rows into a newline-separated string like:
      - `Rule Name: Rule Content`
    - Build normalized `champion` and `challenger` objects
    - Load `replyRateFloor` from `$env.REPLY_RATE_FLOOR` or default to `1.0`
    - Compute:
      - `bothHaveEnoughData = champion.sends >= 200 && challenger.sends >= 200`
      - `hasChallenger = challenger.body.length > 0`
      - `belowFloor = champion.replyRate < replyRateFloor && challenger.replyRate < replyRateFloor && champion.sends >= 200`

22. Connect:
    - `Read Experiment History` → `Merge Stats and History`
    - `Read Active Variants` → `Merge Stats and History`
    - `Read Program Doc` → `Merge Stats and History`

23. Add a **Code** node named `Check Data Threshold`.

24. Configure it to return one item with `action` set as follows:
    - `safety_pause` if `belowFloor` is true
    - `generate_first_challenger` if there is no challenger
    - `evaluate_and_generate` if both variants have enough data
    - `wait` otherwise

25. Connect `Merge Stats and History` → `Check Data Threshold`.

26. Add a **Code** node named `Route by Action`.

27. Configure it so:
    - if `action === 'wait'`, return `[]`
    - otherwise, return the original item

28. Connect `Check Data Threshold` → `Route by Action`.

---

## 4.5 Build the safety and AI section

29. Add a **Code** node named `Safety Floor Check`.

30. Configure it to set:
    - `_route = 'telegram'` if `action === 'safety_pause'`
    - `_route = 'claude'` otherwise

31. Connect `Route by Action` → `Safety Floor Check`.

32. Add an **HTTP Request** node named `Telegram Safety Alert`.
    - Method: `POST`
    - URL:
      `https://api.telegram.org/bot{{ $env.TELEGRAM_BOT_TOKEN }}/sendMessage`
    - Send body as JSON
    - JSON body:
      - `chat_id = {{ $env.TELEGRAM_CHAT_ID }}`
      - `text` containing the pause alert and current reply rates
      - `parse_mode = HTML`

33. Connect `Safety Floor Check` → `Telegram Safety Alert`.

34. Add an **HTTP Request** node named `Claude Evaluate and Generate`.
    - Method: `POST`
    - URL: `https://api.anthropic.com/v1/messages`
    - Headers:
      - `x-api-key = {{ $env.ANTHROPIC_API_KEY }}`
      - `anthropic-version = 2023-06-01`
      - `Content-Type = application/json`
    - JSON body:
      - `model = claude-sonnet-4-6`
      - `max_tokens = 2000`
      - `messages` with one `user` message containing:
        - program rules
        - action
        - champion data
        - challenger data
        - serialized experiment history
        - total experiments
        - instructions for either evaluation or first challenger generation
        - a strict JSON response schema with fields:
          - `winner`
          - `analysis`
          - `hypothesis`
          - `change_type`
          - `new_challenger_subject`
          - `new_challenger_body`
          - `confidence`

35. Connect `Safety Floor Check` → `Claude Evaluate and Generate`.

36. Add a **Code** node named `Parse Claude Response`.

37. Configure it to:
    - read `content[0].text` from the Anthropic response
    - extract the first JSON object using regex
    - `JSON.parse` it
    - retrieve the earlier structured item from `Check Data Threshold`
    - output:
      - `winner`
      - `analysis`
      - `hypothesis`
      - `changeType`
      - `newChallengerSubject`
      - `newChallengerBody`
      - `confidence`
      - `previousChampion`
      - `previousChallenger`
      - `experimentNumber = totalExperiments + 1`
      - `timestamp = new Date().toISOString()`

38. Connect `Claude Evaluate and Generate` → `Parse Claude Response`.

---

## 4.6 Build logging and sheet updates

39. Add an **HTTP Request** node named `Log Experiment to History`.
    - Method: `POST`
    - URL:
      `https://sheets.googleapis.com/v4/spreadsheets/{{ $env.OPTIMIZER_SHEET_ID }}/values/Experiments!A:M:append?valueInputOption=USER_ENTERED`
    - Authentication: Generic Credential Type / HTTP Header Auth
    - JSON body should append one row with:
      - experiment number
      - timestamp
      - previous champion body
      - previous challenger body
      - previous champion subject
      - previous challenger subject
      - previous champion sends
      - previous challenger sends
      - previous champion reply rate
      - previous challenger reply rate
      - winner
      - change type
      - combined analysis and hypothesis

40. Connect `Parse Claude Response` → `Log Experiment to History`.

41. Add an **HTTP Request** node named `Update Active Variants`.
    - Method: `PUT`
    - URL:
      `https://sheets.googleapis.com/v4/spreadsheets/{{ $env.OPTIMIZER_SHEET_ID }}/values/Active!A2:F3?valueInputOption=USER_ENTERED`
    - Authentication: Generic Credential Type / HTTP Header Auth
    - JSON body should overwrite rows 2 and 3:
      - row 2 = champion
      - row 3 = challenger
    - Promote the previous challenger only if `winner === 'challenger'`
    - Reset sends/open/reply values to `"0"`

42. Connect `Parse Claude Response` → `Update Active Variants`.

---

## 4.7 Build Instantly deployment

43. Add an **HTTP Request** node named `Update Instantly Campaign Copy`.
    - Method: `POST`
    - URL:
      `https://api.instantly.ai/api/v2/campaigns/{{ $env.REMOVEFAST_CAMPAIGN_ID }}/sequences`
    - Headers:
      - `Authorization = Bearer {{ $env.INSTANTLY_API_KEY }}`
      - `Content-Type = application/json`
    - JSON body:
      - `sequences` array with two entries
      - each entry contains one `steps` array
      - first sequence uses the promoted champion
      - second sequence uses the new challenger

44. Connect:
    - `Log Experiment to History` → `Update Instantly Campaign Copy`
    - `Update Active Variants` → `Update Instantly Campaign Copy`

---

## 4.8 Credential setup summary

45. **Instantly**
   - No formal credential object is used in the original workflow
   - API key is injected via header expression from `INSTANTLY_API_KEY`

46. **Google Sheets**
   - The original workflow uses HTTP Header Auth credentials
   - Configure a valid bearer token mechanism for Google Sheets API requests
   - Apply that credential to:
     - `Read Experiment History`
     - `Read Active Variants`
     - `Read Program Doc`
     - `Log Experiment to History`
     - `Update Active Variants`

47. **Anthropic**
   - No formal credential object is used in the original workflow
   - API key is set using `x-api-key` header from `ANTHROPIC_API_KEY`

48. **Telegram**
   - No formal credential object is used in the original workflow
   - Bot token is embedded in the request URL using `TELEGRAM_BOT_TOKEN`

---

## 4.9 Recommended fixes while rebuilding

To make the workflow behave as intended, implement these improvements during reconstruction:

49. **Replace `Safety Floor Check` with an IF or Switch node**
   - Condition on `action` or `_route`
   - Route only safety pauses to Telegram
   - Route only non-pause actions to Claude

50. **Prevent duplicate Instantly updates**
   - Add a Merge node that waits for both:
     - `Log Experiment to History`
     - `Update Active Variants`
   - Then connect the Merge node to `Update Instantly Campaign Copy`
   - This ensures Instantly is updated once, after both Google Sheets writes succeed

51. **Add response validation after Claude parsing**
   - Verify required keys exist
   - Verify `winner` is one of:
     - `champion`
     - `challenger`
     - `none`
   - Verify `newChallengerSubject` and `newChallengerBody` are nonempty when needed

52. **Optionally parameterize the send threshold**
   - Move the 200-send threshold into an environment variable or Program sheet rule

53. **Optionally add retry/error handling**
   - Google Sheets API
   - Instantly API
   - Anthropic API
   - Telegram API

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Inspired by Karpathy’s Autoresearch | https://github.com/karpathy/autoresearch |
| The workflow title in the JSON is `Self-Improving Cold Email Copy Optimizer`, while the user-provided title is `Optimize Cold Email Copy with Claude, Instantly, and Google Sheets` | Naming/context note |
| The environment variable name uses `REMOVEFAST_CAMPAIGN_ID`, but the actual API target is Instantly | Review naming consistency to avoid confusion |
| The workflow depends on append-order semantics in the `Experiments` tab when selecting the last 10 rounds | Operational note |
| The workflow assumes Google Sheets acts as the source of truth for champion/challenger state between optimization rounds | Operational note |
| The current branching design does not truly separate Telegram and Claude paths | Strongly recommended to use IF/Switch routing |
| The current final deployment design may call Instantly twice because two branches converge directly into one HTTP Request node | Strongly recommended to merge branches before deployment |