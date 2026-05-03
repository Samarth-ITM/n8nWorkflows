Analyze failed workflows with Claude via OpenRouter and log to Sheets with Slack, Email, Discord alerts

https://n8nworkflows.xyz/workflows/analyze-failed-workflows-with-claude-via-openrouter-and-log-to-sheets-with-slack--email--discord-alerts-12502


# Analyze failed workflows with Claude via OpenRouter and log to Sheets with Slack, Email, Discord alerts

## 1. Workflow Overview

**Purpose:**  
This workflow automatically reacts to **failed n8n executions**, prepares a compact but information-rich context, asks **Claude 3.5 Sonnet via OpenRouter** to analyze the failure (root cause + fixes), extracts an AI confidence score, generates a stable error signature for deduplication, logs the incident to **Google Sheets**, and sends notifications via **Email, Slack, Discord, and/or a generic Webhook**. It includes a basic alert-fatigue mechanism to suppress repeated alerts for the same error within a time window.

**Target use cases:**
- Operations monitoring for n8n instances (self-hosted or cloud)
- Faster incident triage with AI-generated remediation steps
- Centralized error tracking in Sheets + multi-channel alerting

### 1.1 Failure Intake & Normalization
Captures the failed execution payload from n8n’s Error Trigger, then normalizes key fields (workflow/execution IDs, failed node, stack, timestamp, etc.).

### 1.2 Context Reduction (“Middle-out” extraction)
Builds a filtered workflow/execution JSON focused around the failed node and its neighbors to stay within LLM context limits.

### 1.3 AI Root Cause Analysis (Claude via OpenRouter)
Sends the filtered context to the LLM chain node and receives structured analysis including a “CONFIDENCE: x.x” line.

### 1.4 Confidence + Signature + Logging Record
Extracts confidence, generates a dedupe-friendly signature, then prepares a structured log record for storage.

### 1.5 Logging + Alert Fatigue Control
Appends to Google Sheets and checks recent rows for duplicate signatures within a time window to optionally suppress/downgrade notifications.

### 1.6 Notifications (Email → Slack/Discord/Webhook fan-out)
Sends an email first, then fans out to Slack, Discord, and a webhook endpoint.

---

## 2. Block-by-Block Analysis

### Block 1 — Failure Intake & Normalization
**Overview:**  
Triggers when any workflow fails and reshapes the error payload into a consistent schema used by later steps.

**Nodes involved:**
- **Error Trigger**
- **Normalize Error Payload1**

#### Node: Error Trigger
- **Type / role:** `n8n-nodes-base.errorTrigger` — entry point fired on workflow errors.
- **Configuration choices:** No parameters (default behavior).
- **Inputs / outputs:** No input; outputs the failed execution context (`workflow`, `execution`, etc.).
- **Failure modes / edge cases:**
  - Requires n8n error workflow support; ensure the workflow is enabled and allowed to run on failures.
  - Payload structure can differ by n8n version; expressions assuming fields exist may break.

#### Node: Normalize Error Payload1
- **Type / role:** `n8n-nodes-base.set` — normalizes fields for downstream consistency.
- **Key fields created (interpreted):**
  - `workflow_name` = `$json.workflow.name`
  - `workflow_id` = `$json.workflow.id`
  - `execution_id` = `$json.execution.id`
  - `execution_mode` = `$json.execution.mode`
  - `failed_node` = `$json.execution.lastNodeExecuted`
  - `error_message` = `$json.execution.error.message`
  - `error_stack` = `$json.execution.error.stack || 'N/A'`
  - `timestamp` = `$now.toISO()`
  - `trigger_type` = `$json.execution.data?.triggerData?.type || 'unknown'`
  - `input_payload` = JSON string of `execution.data.startData` (truncated to 1000 chars)
  - `environment` = `"production"`
  - `receiver emails` = empty array `[]` (intended recipients list)
- **Connections:**
  - Input: **Error Trigger**
  - Output: **Prepare AI Context1**
- **Edge cases / failure types:**
  - If `execution.error` is missing or shaped differently, expressions can evaluate to `undefined` (usually safe in n8n but may produce blank fields).
  - `receiver emails` defaults to `[]`; if not populated, Gmail sendTo becomes an empty string.

---

### Block 2 — Context Reduction (“Middle-out” extraction)
**Overview:**  
Creates a compact view of the workflow definition + execution run data centered around the failed node, bounded by a character budget (default 50k).

**Nodes involved:**
- **Prepare AI Context1**

#### Node: Prepare AI Context1
- **Type / role:** `n8n-nodes-base.code` — builds filtered `workflow_json` and `execution_json`.
- **Configuration choices (interpreted):**
  - Uses a BFS-like “middle-out” expansion from `failed_node`
  - Budget: `MAX_CONTEXT_CHARS = 50000`
  - Max hops safety cap: 10
  - Includes node definitions + per-node execution runData where available
  - Stops expanding before exceeding budget and avoids mid-node truncation by estimating size
- **Key variables produced:**
  - `workflow_json` (stringified filtered workflow)
  - `execution_json` (stringified filtered execution)
  - `_extraction_stats` (nodes included, hops, estimated chars, compression ratio)
- **Connections:**
  - Input: **Normalize Error Payload1**
  - Output: **AI Error Analysis1**
- **Edge cases / failure types:**
  - If `workflowData.nodes` or `workflowData.connections` are absent, graph traversal will be incomplete (AI context weaker).
  - If `failed_node` is empty or not found, inclusion set may be minimal/empty, reducing usefulness.
  - Execution run data path assumes `execution.data.resultData.runData` exists (may vary with n8n versions or error types).

---

### Block 3 — AI Root Cause Analysis (Claude via OpenRouter)
**Overview:**  
Submits the filtered context and error metadata to a LangChain LLM chain node, backed by OpenRouter’s Claude model, requesting structured output + explicit confidence score.

**Nodes involved:**
- **OpenRouter Chat Model1**
- **AI Error Analysis1**

#### Node: OpenRouter Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — chat LLM provider connector.
- **Configuration choices:**
  - Model: `anthropic/claude-3.5-sonnet`
  - Credentials: OpenRouter API credential named **“Reddit Replies”** (user-supplied)
- **Connections:**
  - Output (AI languageModel): to **AI Error Analysis1** (`ai_languageModel` port)
- **Failure modes:**
  - Invalid/expired OpenRouter API key
  - Model availability or OpenRouter rate limits
  - Timeouts on large prompts (especially if context budget is increased)

#### Node: AI Error Analysis1
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — sends prompt + context to LLM and returns analysis text.
- **Configuration choices (interpreted):**
  - Prompt is “define” type, includes:
    - `workflow_json` (filtered) and `execution_json` (filtered) from **Prepare AI Context1**
    - error fields (`failed_node`, `error_message`, `error_stack`, timestamp)
    - reproduction context fields
  - Requests: Root cause, error chain tracing, fix steps, prevention, and `CONFIDENCE: [score]`
- **Outputs:**
  - Commonly returns `text` (and sometimes `output`, depending on node version/config)
- **Connections:**
  - Input: **Prepare AI Context1**
  - Output: **Extract AI Confidence**
- **Edge cases / failure types:**
  - Prompt references `{{ $json.workflow_json }}` / `{{ $json.execution_json }}`: if missing, analysis quality drops.
  - Output field naming mismatch (some downstream nodes reference `.text`, others `.output`; see “Known issues” below).

---

### Block 4 — Confidence + Signature + Logging Record
**Overview:**  
Extracts the AI confidence score, creates a stable error signature for deduplication, and prepares a unified row record for logging.

**Nodes involved:**
- **Extract AI Confidence**
- **Generate Error Signature**
- **Prepare Log Record**

#### Node: Extract AI Confidence
- **Type / role:** `n8n-nodes-base.code` — parses `CONFIDENCE: x` from AI output.
- **Configuration choices:**
  - Reads `text || output || ''`
  - Regex: `/CONFIDENCE:\s*([0-9.]+)/i`
  - Validates 0–1 range, defaults to `0.5`
- **Connections:**
  - Input: **AI Error Analysis1**
  - Output: **Generate Error Signature**
- **Edge cases:**
  - If the model omits the confidence line or formats differently, confidence remains default 0.5.

#### Node: Generate Error Signature
- **Type / role:** `n8n-nodes-base.code` — generates normalized signature hash for deduplication.
- **Configuration choices (interpreted):**
  - Uses djb2-style hash (custom, no crypto module)
  - Normalizes error message by replacing:
    - ISO timestamps → `TIMESTAMP`
    - long digit sequences → `TIMESTAMP`
    - UUIDs → `UUID`
    - generic numbers → `NUMBER`
    - quoted strings → `STRING`
    - normalizes whitespace
  - Signature input: `${failedNode}|${normalizedError}|${workflowId}`
- **Outputs:**
  - `error_signature`
  - `normalized_error`
- **Connections:**
  - Input: **Extract AI Confidence**
  - Output: **Prepare Log Record**
- **Edge cases:**
  - Over-normalization may cause distinct errors to collide into the same signature.
  - Under-normalization may fail to dedupe (e.g., dynamic URLs not caught by regex).

#### Node: Prepare Log Record
- **Type / role:** `n8n-nodes-base.set` — shapes the final log row and carries forward fields.
- **Configuration choices / key expressions:**
  - Pulls canonical fields from **Normalize Error Payload1**
  - `errorSignature` = `$json.error_signature`
  - `aiRootCause` = `$('AI Error Analysis1').item.json.text.split('\n\n')[0].substring(0, 500)`
  - `aiFixSummary` = `$('AI Error Analysis1').item.json.text.split('\n\n')[2]?.substring(0, 500) || 'N/A'`
  - `aiConfidence` = `$json.ai_confidence || 0.5`
  - `severity` hard-coded to `"high"`
- **Connections:**
  - Input: **Generate Error Signature**
  - Output: **Log to Google Sheets1**
- **Edge cases / failure types:**
  - Assumes AI output is in `.text` and uses `split('\n\n')`; if formatting differs, root cause / fix summary extraction may be wrong.
  - If `AI Error Analysis1` returns `.output` instead of `.text`, these expressions may fail or produce empty strings.

---

### Block 5 — Logging + Alert Fatigue Control
**Overview:**  
Appends the incident to Google Sheets, then checks recent sheet rows to detect duplicate signatures within a time window and sets suppression/downgrade flags.

**Nodes involved:**
- **Log to Google Sheets1**
- **Check Alert Fatigue**

#### Node: Log to Google Sheets1
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row to a spreadsheet.
- **Configuration choices:**
  - Operation: **append**
  - Document: “Test Test Test” (Spreadsheet ID: `1OgkqYM1vYDi5bLrxfOBete2Og7hL8JVjBAvfskXZk5Y`)
  - Sheet/tab: “Sheet1” (gid=0)
  - Mapping mode: auto-map input data
- **Connections:**
  - Input: **Prepare Log Record**
  - Output: **Check Alert Fatigue**
- **Failure modes:**
  - OAuth credential not configured/expired
  - Sheet/tab renamed, permissions issues
  - Column mismatch if the sheet headers do not match incoming fields (auto-map behavior may be unpredictable)

#### Node: Check Alert Fatigue
- **Type / role:** `n8n-nodes-base.code` — detects recurring errors.
- **Configuration choices:**
  - Looks at last `LOOKBACK_COUNT = 10` entries
  - Time window: `TIME_WINDOW_HOURS = 1`
  - Reads “recent entries” from `$('Log to Google Sheets1').all()`
  - Sets:
    - `alert_suppressed` (true if duplicates found)
    - `downgrade_urgency` (true if duplicates found)
    - `duplicate_count`, `last_occurrence`
- **Connections:**
  - Input: **Log to Google Sheets1**
  - Output: **Send Email1**
- **Edge cases / failure types:**
  - **Logical issue:** An “append” operation does not inherently return “all sheet rows”; it typically returns only the appended row(s). Therefore, `$('Log to Google Sheets1').all()` likely does **not** provide prior history, making dedupe ineffective.
  - Timestamp parsing: `new Date(currentItem.timestamp)` and `new Date(entryData.timestamp)` require consistent ISO timestamps in the sheet.

---

### Block 6 — Notifications (Email → Slack/Discord/Webhook fan-out)
**Overview:**  
Sends an HTML email alert, then fans out to Slack, Discord, and an HTTP webhook endpoint.

**Nodes involved:**
- **Send Email1**
- **Send Slack Message1**
- **Send Discord Message1**
- **Webhook Notification1**

#### Node: Send Email1
- **Type / role:** `n8n-nodes-base.gmail` — sends the primary email alert.
- **Configuration choices (interpreted):**
  - `sendTo` = `{{ $('Normalize Error Payload1').item.json['receiver emails'].join(', ') }}`
  - HTML body includes key error fields + AI analysis and confidence label
  - Subject includes `[⚠️ Low Confidence]` vs `[High]` based on confidence threshold
- **Connections:**
  - Input: **Check Alert Fatigue**
  - Output: **Send Slack Message1**, **Send Discord Message1**, **Webhook Notification1** (fan-out)
- **Edge cases / failure types:**
  - If `receiver emails` is empty, the node may fail (invalid recipient) or send nowhere.
  - Uses `$('AI Error Analysis1').item.json.text` in the body—may break if the LLM node returns `output` instead.
  - Uses confidence from `$('Check Alert Fatigue').item.json.ai_confidence` (present only if prior nodes preserve it correctly).

#### Node: Send Slack Message1
- **Type / role:** `n8n-nodes-base.slack` — posts an alert to a Slack channel.
- **Configuration choices:**
  - Channel selection: `select: channel`
  - `channelId` is a placeholder: `<__PLACEHOLDER_VALUE__Slack Channel ID or Name__>`
  - Message references:
    - `$json.downgrade_urgency` and `$json.error_signature`
    - `$('AI Error Analysis1').item.json.output.split('\n')...` (note: uses `.output`)
    - `$json.reproduction_context` (not created anywhere in this workflow)
- **Connections:**
  - Input: **Send Email1**
  - Output: none
- **Failure modes / edge cases:**
  - Slack credentials missing/invalid
  - Placeholder channel not replaced
  - **Expression issues:**
    - `AI Error Analysis1` may not have `.output`
    - `reproduction_context` is undefined unless added upstream

#### Node: Send Discord Message1
- **Type / role:** `n8n-nodes-base.discord` — sends message via Discord webhook.
- **Configuration choices:**
  - Authentication: webhook
  - Content references:
    - `$('AI Error Analysis1').item.json.output.substring(0, 500)` (uses `.output`)
    - Confidence is incorrectly read from `$('Generate Error Signature').item.json.ai_confidence` (that node does not set `ai_confidence`; it’s set earlier in **Extract AI Confidence**)
    - `$('Prepare Log Record').item.json.reproduction_context` (not created in Prepare Log Record)
- **Connections:**
  - Input: **Send Email1**
  - Output: none
- **Failure modes / edge cases:**
  - Webhook URL/credential misconfiguration
  - Multiple broken references (`output`, `reproduction_context`, wrong confidence source)

#### Node: Webhook Notification1
- **Type / role:** `n8n-nodes-base.httpRequest` — posts alert data to an external endpoint.
- **Configuration choices:**
  - URL: `<__PLACEHOLDER_VALUE__Webhook URL__>` (must be replaced)
  - Method: POST
  - Sends body parameters including workflow name, failed node, error message, AI analysis, signature, confidence, and suppression flag
  - `reproduction_context` is generated inline via `JSON.stringify({ trigger_type, execution_mode, environment })`
- **Connections:**
  - Input: **Send Email1**
  - Output: none
- **Failure modes:**
  - Placeholder URL not replaced
  - Endpoint auth requirements not implemented (no headers/auth configured)
  - AI analysis references `$('AI Error Analysis1').item.json.output` (may need `.text`)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Error Trigger | n8n-nodes-base.errorTrigger | Trigger on any workflow failure | — | Normalize Error Payload1 | # Analyze failed n8n workflows with AI and send Slack, Email, or Discord alerts / How it works / Setup steps |
| Normalize Error Payload1 | n8n-nodes-base.set | Normalize error payload fields | Error Trigger | Prepare AI Context1 | # Analyze failed n8n workflows with AI and send Slack, Email, or Discord alerts / How it works / Setup steps |
| Prepare AI Context1 | n8n-nodes-base.code | Middle-out context extraction and filtering | Normalize Error Payload1 | AI Error Analysis1 | # Analyze failed n8n workflows with AI and send Slack, Email, or Discord alerts / How it works / Setup steps |
| OpenRouter Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (Claude via OpenRouter) | — | AI Error Analysis1 (ai_languageModel) | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| AI Error Analysis1 | @n8n/n8n-nodes-langchain.chainLlm | Runs LLM analysis chain with structured prompt | Prepare AI Context1 + OpenRouter Chat Model1 | Extract AI Confidence | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| Extract AI Confidence | n8n-nodes-base.code | Parse CONFIDENCE score from AI output | AI Error Analysis1 | Generate Error Signature |  |
| Generate Error Signature | n8n-nodes-base.code | Normalize error message + hash signature for dedupe | Extract AI Confidence | Prepare Log Record |  |
| Prepare Log Record | n8n-nodes-base.set | Create structured record for logging | Generate Error Signature | Log to Google Sheets1 |  |
| Log to Google Sheets1 | n8n-nodes-base.googleSheets | Append log row to Google Sheets | Prepare Log Record | Check Alert Fatigue | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| Check Alert Fatigue | n8n-nodes-base.code | Detect recent duplicates and suppress/downgrade | Log to Google Sheets1 | Send Email1 |  |
| Send Email1 | n8n-nodes-base.gmail | Primary email alert (HTML) | Check Alert Fatigue | Send Slack Message1; Send Discord Message1; Webhook Notification1 | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| Send Slack Message1 | n8n-nodes-base.slack | Slack alert message | Send Email1 | — | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| Send Discord Message1 | n8n-nodes-base.discord | Discord webhook alert | Send Email1 | — | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| Webhook Notification1 | n8n-nodes-base.httpRequest | POST alert payload to external system | Send Email1 | — | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note (no execution role) | — | — | # Analyze failed n8n workflows with AI and send Slack, Email, or Discord alerts / How it works / Setup steps |
| Sticky Note5 | n8n-nodes-base.stickyNote | Credential requirements note (no execution role) | — | — | ## 🔐 Credentials Required / OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *Analyze failed n8n workflows with AI and send Slack, Email, or Discord alerts* (or your preferred name)
- Leave it inactive until credentials and placeholders are set.

2) **Add Trigger**
- Add node: **Error Trigger** (`Error Trigger`)
- No configuration needed.
- This is the workflow entry point.

3) **Normalize payload**
- Add node: **Set** → name it **Normalize Error Payload1**
- Add fields (String unless noted):
  - `workflow_name` = `{{$json.workflow.name}}`
  - `workflow_id` = `{{$json.workflow.id}}`
  - `execution_id` = `{{$json.execution.id}}`
  - `execution_mode` = `{{$json.execution.mode}}`
  - `failed_node` = `{{$json.execution.lastNodeExecuted}}`
  - `error_message` = `{{$json.execution.error.message}}`
  - `error_stack` = `{{$json.execution.error.stack || 'N/A'}}`
  - `timestamp` = `{{$now.toISO()}}`
  - `receiver emails` (Type: Array) = `=[]`
  - `trigger_type` = `{{$json.execution.data?.triggerData?.type || 'unknown'}}`
  - `input_payload` = `{{ JSON.stringify($json.execution.data?.startData?.destinationNode ? $json.execution.data.startData : {}, null, 2).substring(0, 1000) }}`
  - `environment` = `production`
- Connect: **Error Trigger → Normalize Error Payload1**

4) **Add context extraction**
- Add node: **Code** → name it **Prepare AI Context1**
- Paste the “middle-out context extraction algorithm” code (as provided).
- Connect: **Normalize Error Payload1 → Prepare AI Context1**

5) **Add OpenRouter chat model**
- Add node: **OpenRouter Chat Model** → name it **OpenRouter Chat Model1**
- Model: `anthropic/claude-3.5-sonnet`
- Credentials: create/select **OpenRouter API** credentials (API key from OpenRouter).

6) **Add LLM chain**
- Add node: **LLM Chain** (`@n8n/n8n-nodes-langchain.chainLlm`) → name it **AI Error Analysis1**
- Prompt type: **Define**
- Prompt content: use the provided structured prompt that embeds:
  - `{{$json.workflow_json}}`, `{{$json.execution_json}}`
  - `failed_node`, `error_message`, `error_stack`, `timestamp`, etc.
- Connect model: **OpenRouter Chat Model1 → AI Error Analysis1** using the **ai_languageModel** connection.
- Connect data: **Prepare AI Context1 → AI Error Analysis1**

7) **Extract confidence**
- Add node: **Code** → name it **Extract AI Confidence**
- Paste the confidence extraction code (regex for `CONFIDENCE:`).
- Connect: **AI Error Analysis1 → Extract AI Confidence**

8) **Generate error signature**
- Add node: **Code** → name it **Generate Error Signature**
- Paste the hashing + normalization code.
- Connect: **Extract AI Confidence → Generate Error Signature**

9) **Prepare log record**
- Add node: **Set** → name it **Prepare Log Record**
- Add fields (matching the workflow’s intent):
  - `timestamp` = `{{ $('Normalize Error Payload1').item.json.timestamp }}`
  - `workflowName` = `{{ $('Normalize Error Payload1').item.json.workflow_name }}`
  - `workflowId` = `{{ $('Normalize Error Payload1').item.json.workflow_id }}`
  - `executionId` = `{{ $('Normalize Error Payload1').item.json.execution_id }}`
  - `failedNode` = `{{ $('Normalize Error Payload1').item.json.failed_node }}`
  - `errorMessage` = `{{ $('Normalize Error Payload1').item.json.error_message }}`
  - `errorSignature` = `{{ $json.error_signature }}`
  - `aiRootCause` = `{{ $('AI Error Analysis1').item.json.text.split('\n\n')[0].substring(0, 500) }}`
  - `aiFixSummary` = `{{ $('AI Error Analysis1').item.json.text.split('\n\n')[2]?.substring(0, 500) || 'N/A' }}`
  - `aiConfidence` (Number) = `{{ $json.ai_confidence || 0.5 }}`
  - `severity` = `high`
  - `executionMode` = `{{ $('Normalize Error Payload1').item.json.execution_mode }}`
  - `triggerType` = `{{ $('Normalize Error Payload1').item.json.trigger_type }}`
  - `environment` = `{{ $('Normalize Error Payload1').item.json.environment }}`
  - `inputPayload` = `{{ $('Normalize Error Payload1').item.json.input_payload }}`
- Connect: **Generate Error Signature → Prepare Log Record**

10) **Log to Google Sheets**
- Add node: **Google Sheets** → name it **Log to Google Sheets1**
- Operation: **Append**
- Authenticate with **Google Sheets OAuth2** credentials.
- Select:
  - Spreadsheet (Document)
  - Sheet tab (e.g., “Sheet1”)
- Mapping: Auto-map (or map explicitly to known headers).
- Connect: **Prepare Log Record → Log to Google Sheets1**

11) **Add alert-fatigue checker**
- Add node: **Code** → name it **Check Alert Fatigue**
- Paste the provided code (lookback count + 1-hour window).
- Connect: **Log to Google Sheets1 → Check Alert Fatigue**

12) **Add Email notification**
- Add node: **Gmail** → name it **Send Email1**
- Credentials: Gmail OAuth2.
- To: `{{ $('Normalize Error Payload1').item.json['receiver emails'].join(', ') }}`
- Subject and HTML body: use the provided templates (includes execution link and AI analysis).
- Connect: **Check Alert Fatigue → Send Email1**

13) **Add Slack notification**
- Add node: **Slack** → name it **Send Slack Message1**
- Credentials: Slack OAuth2/bot token as appropriate.
- Choose channel and replace the placeholder channel ID/name.
- Paste the message template.
- Connect: **Send Email1 → Send Slack Message1**

14) **Add Discord notification**
- Add node: **Discord** → name it **Send Discord Message1**
- Authentication: webhook (configure webhook URL in credentials/node as required by your n8n setup).
- Paste the message template.
- Connect: **Send Email1 → Send Discord Message1**

15) **Add generic webhook notification**
- Add node: **HTTP Request** → name it **Webhook Notification1**
- Method: POST
- URL: replace `<__PLACEHOLDER_VALUE__Webhook URL__>`
- Enable “Send Body” and add body parameters as provided.
- Connect: **Send Email1 → Webhook Notification1**

16) **Add documentation sticky notes (optional)**
- Add two **Sticky Note** nodes and paste the provided content:
  - Description/how it works/setup steps
  - Credential requirements

17) **Finalize**
- Replace placeholders:
  - Slack channel ID/name
  - Webhook URL
  - Add recipient emails (populate `receiver emails` array)
- Save and **activate** the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically runs whenever an n8n workflow fails… logs the error and sends notifications… includes basic alert fatigue control…” | Sticky Note (workflow description and setup steps) |
| “Credentials Required: OpenRouter API, Google Sheets, Slack/Discord/Email, optional Webhook URL. No credentials included; users must add their own.” | Sticky Note (credential requirements) |

### Known integration/logic issues to anticipate (important)
- **Alert fatigue check likely ineffective as written:** it tries to read historical rows from `Log to Google Sheets1`, but an **append** operation usually returns only the appended row(s). To truly check history, add a **Google Sheets “Read/Get Many”** node before the fatigue check.
- **Inconsistent use of AI output fields:** some nodes use `AI Error Analysis1.item.json.text`, others use `.output`. Standardize to the actual field produced by your Chain node (often `text`).
- **Undefined `reproduction_context`:** Slack and Discord templates reference `reproduction_context`, but no node creates it. Either add it in **Normalize Error Payload1** or **Prepare Log Record**, or remove those references.
- **Discord confidence reference is wrong:** it reads confidence from **Generate Error Signature** output, but confidence is produced in **Extract AI Confidence** and should be carried forward (e.g., use `{{ $('Check Alert Fatigue').item.json.ai_confidence }}` or `{{ $json.ai_confidence }}` depending on the input to the Discord node).