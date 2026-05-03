Monitor Cloudflare incidents and alert via Slack, Telegram, and Jira

https://n8nworkflows.xyz/workflows/monitor-cloudflare-incidents-and-alert-via-slack--telegram--and-jira-12086


# Monitor Cloudflare incidents and alert via Slack, Telegram, and Jira

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor Cloudflare incidents and alert via Slack, Telegram, and Jira  
**Workflow name (in JSON):** Cloudflare Incident Monitoring & Alerting Workflow  
**Purpose:** Poll Cloudflare’s public Status API on a schedule, detect unresolved incidents, enrich and deduplicate them, then notify teams via **Slack** and **Telegram**. For **high-impact incidents (major/critical)**, it escalates by creating a **Jira issue** and pinging an IT support Slack channel.

### 1.1 Ingestion & Detection
- Runs every 30 minutes.
- Fetches unresolved incidents from Cloudflare Status.
- If there are **no incidents**, sends an “all operational” message (only at the top of the hour) to avoid noise.

### 1.2 Enrichment, Scoring & Logging
- Normalizes incidents (severity score, impact emoji, Singapore time conversion, latest update extraction, component names).
- Creates a per-run “platform snapshot” (components + degraded components + maintenances).
- Logs incident rows into Google Sheets for auditing.

### 1.3 Noise Control & Deduplication
- Uses workflow static data (global) as an in-memory cache to suppress repeat alerts for the same incident delta within **30 minutes**.
- If nothing new is detected, sets `skipNotifications: true`.

### 1.4 Decision & Routing
- If an incident’s impact is **critical/major**, route to **Jira escalation** branch.
- Independently, build a combined notification payload; decide whether to broadcast (or skip).

### 1.5 Notification & Escalation
- Uses an LLM agent to craft **Telegram-safe plain text** alert content for Slack + Telegram.
- High-impact incidents trigger a second LLM agent to produce structured fields (via output parser) for a Jira ticket.
- Posts Jira tracking link to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Ingestion & Detection
**Overview:** Triggers periodically, fetches Cloudflare unresolved incidents via Decodo HTTP, then decides whether there are incidents; if none, it may send “all good” messages only at the top of the hour.

**Nodes involved:**
- Schedule Trigger
- Decodo HTTP Request
- CloudFlare OK?
- Is top of hour?
- Send message via Telegram
- Send message via Slack
- No Operation, do nothing

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point.
- **Config:** Runs every **30 minutes**.
- **Outputs:** Connects to **Decodo HTTP Request** (and also directly to **CloudFlare OK?**, but CloudFlare OK? requires HTTP output, so the Decodo path is the functional one).
- **Potential failures:** None typical; clock drift/timezone affects only “top of hour” logic later.

#### Node: Decodo HTTP Request
- **Type / role:** `@decodo/n8n-nodes-decodo.decodo` — HTTP fetch using Decodo.
- **Config:** GET `https://www.cloudflarestatus.com/api/v2/incidents/unresolved.json`
- **Credentials:** Decodo API credentials required.
- **Outputs:** To **CloudFlare OK?**
- **Failure types / edge cases:**
  - Decodo credential/auth failures.
  - Rate limiting / transient network errors.
  - Response shape changes (Cloudflare API schema drift).
  - Non-JSON or error payload causing later parsing issues.

#### Node: CloudFlare OK?
- **Type / role:** `n8n-nodes-base.if` — checks “no unresolved incidents”.
- **Key condition:** Treats incidents as empty if:
  - `{{ $json.data.results[0].content.parseJson().incidents }}` is an empty array.
- **Outputs:**
  - **True branch (no incidents):** → **Is top of hour?**
  - **False branch (incidents exist):** → **Enrich Incidents & Score**
- **Edge cases / failure types:**
  - This expression assumes the Decodo response structure includes `data.results[0].content` containing JSON text. If Decodo returns a different shape, `parseJson()` can fail or `results[0]` can be undefined.
  - If Cloudflare API returns directly as JSON (without wrapping), this check will not work as written.

#### Node: Is top of hour?
- **Type / role:** `n8n-nodes-base.if` — noise guard for “all clear” messages.
- **Condition:** `{{ $now.minute }} == 0`
- **Outputs:**
  - **True (minute==0):** → send operational messages via **Telegram** and **Slack**
  - **False:** → **No Operation, do nothing**
- **Edge cases:**
  - Depends on n8n server time and `$now` timezone settings.
  - Schedule is every 30 minutes; you’ll hit minute 0 and 30—so “all clear” is sent only on minute 0.

#### Node: Send message via Telegram (all operational)
- **Type / role:** `n8n-nodes-base.telegram` — sends “✅ operational” message.
- **Config:** `chatId: -1003460669186`; attribution disabled.
- **Text:** fixed “All systems are operational…”
- **Failure types:** Telegram bot token invalid, chat ID wrong, bot not in channel/group, Telegram API downtime.

#### Node: Send message via Slack (all operational)
- **Type / role:** `n8n-nodes-base.slack` — sends “✅ operational” message.
- **Config:** OAuth2 auth, channel `#cloudflare-alert`.
- **Failure types:** OAuth token revoked/expired, missing `chat:write`, channel not found, Slack API rate limits.

#### Node: No Operation, do nothing
- **Type / role:** `n8n-nodes-base.noOp` — explicit sink node when no message should be sent.
- **Failure types:** none.

---

### Block 2 — Enrichment, Scoring & History
**Overview:** When incidents exist, normalize incident objects, compute severity/emoji, extract latest updates, convert timestamps to SGT, build a snapshot of component states, then append incident logs to Google Sheets.

**Nodes involved:**
- Enrich Incidents & Score
- Log Incidents

#### Node: Enrich Incidents & Score
- **Type / role:** `n8n-nodes-base.code` — transforms Cloudflare payload into enriched incident items.
- **Key logic (interpreted):**
  - Reads `payload.incidents`, `payload.result.components`, `payload.scheduled_maintenances`.
  - Builds `degradedComponents` from any component not `operational`.
  - Creates `platform_snapshot` including run ID, maintenances, component summary.
  - If no incidents: returns a single item `{ skipNotifications: true, platform_snapshot }`.
  - Else maps each incident into an enriched record with:
    - `severityScore` from impact (`critical=4`, `major=3`, `minor=2`, `maintenance/none=1`)
    - `impact_emoji` mapping
    - latest update extraction from `incident_updates`
    - `created_at_sgt`, `updated_at_sgt` using `Intl.DateTimeFormat('Asia/Singapore')`
    - `component_names` string
    - `dedupeKey` = `${incident.id}-${incident.status}-${latestUpdate.status || latestUpdate.body || ''}`
- **Outputs:** → **Log Incidents** and → **UTIL: Filter Already Alerted**
- **Failure types / edge cases:**
  - Input payload shape mismatch (e.g., if incidents are nested differently).
  - Invalid ISO timestamps.
  - If `payload.result.components` is absent, degraded component reporting becomes empty (handled by defaults).
  - This node returns “enriched” as multiple items; downstream nodes should expect item-per-incident.

#### Node: Log Incidents
- **Type / role:** `n8n-nodes-base.googleSheets` — append incident rows for auditing.
- **Config:** Append operation to spreadsheet “Cloudflare Incidents”, sheet “Sheet1”.
- **Mapping mode:** auto-map input JSON fields to columns.
- **Failure types:**
  - OAuth expired/revoked; missing Google Sheets permissions.
  - Sheet schema mismatch: auto-map may not populate expected columns.
  - Rate limits on Sheets API.

---

### Block 3 — Noise Control & Deduplication
**Overview:** Filters out incident items that were already alerted recently (30-minute TTL) using workflow global static data. If nothing new remains, produces a single `skipNotifications: true` item.

**Nodes involved:**
- UTIL: Filter Already Alerted
- (Downstream dependencies: UTIL: Prepare Notification Payload, High Impact Escalation?)

#### Node: UTIL: Filter Already Alerted
- **Type / role:** `n8n-nodes-base.code` — time-based dedupe filter.
- **Key logic:**
  - Uses `$getWorkflowStaticData('global')` as `alertCache`.
  - TTL = **30 minutes**.
  - For each incident item:
    - Skip if `skipNotifications`.
    - Key is `dedupeKey` else `incident_id` else `name`.
    - If key unseen or older than TTL ⇒ keep and refresh timestamp.
  - If none kept ⇒ output one item:
    - `skipNotifications: true`
    - `dedupeWindowMinutes: 30`
    - `note: "No new incident deltas detected within the dedupe window."`
- **Outputs:** → **UTIL: Prepare Notification Payload** and → **High Impact Escalation?**
- **Edge cases / failure types:**
  - Static data is per-workflow and persists across executions; after edits/imports, cache might reset.
  - If `dedupeKey` changes frequently (e.g., update body changes), it will allow alerts (by design).
  - In clustered/multi-instance n8n setups, static data may not synchronize across instances, reducing dedupe effectiveness.

---

### Block 4 — Decision & Routing
**Overview:** Routes high-impact incidents to Jira escalation and prepares a combined payload for notification broadcast control.

**Nodes involved:**
- High Impact Escalation?
- Low Impact Buffer
- UTIL: Prepare Notification Payload
- Should broadcast notifications?
- Skip Notification Broadcast

#### Node: High Impact Escalation?
- **Type / role:** `n8n-nodes-base.if` — per-incident escalation gate.
- **Condition:** `impact == "critical"` OR `impact == "major"`.
- **Inputs:** Individual incident items (post-dedupe).
- **Outputs:**
  - **True:** → **Support Request Reader Agent** (Jira creation path)
  - **False:** → **Low Impact Buffer**
- **Edge cases:**
  - If impact values differ from expected (e.g., “Critical”), strict equals will not match.
  - “maintenance” incidents will not escalate.

#### Node: Low Impact Buffer
- **Type / role:** `n8n-nodes-base.noOp` — placeholder for future buffering logic.
- **Failure types:** none.

#### Node: UTIL: Prepare Notification Payload
- **Type / role:** `n8n-nodes-base.code` — aggregates incident items into a single notification context.
- **Key logic:**
  - Collects all incident items (filters those with `incident_id` and `!skipNotifications`).
  - Builds stats: totals, highImpact count, byImpact counts, impacted components, degraded components, maintenance count.
  - Builds `opsDigest` string and `aiPromptContext` JSON string for LLM.
  - Returns single item with:
    - `skipNotifications` (true if incidents list is empty)
    - `incidents[]`, `stats`, `opsDigest`, `aiPromptContext`
- **Output:** → **Should broadcast notifications?**
- **Edge cases / failure types:**
  - `opsDigestLines.join('')` concatenates without separators (likely intended `'\n'`); the digest becomes one long line.
  - If upstream emits only `skipNotifications` item, `incidents` becomes empty and notifications will be skipped.

#### Node: Should broadcast notifications?
- **Type / role:** `n8n-nodes-base.if` — decides whether to skip broadcasting.
- **Condition (as configured):** checks boolean true on `{{ $json.skipNotifications }}`
- **Outputs:**
  - **True (skipNotifications==true):** → **Skip Notification Broadcast**
  - **False:** → **CloudFlare Alert Bot**
- **Important note:** The node name suggests the opposite (broadcast?), but the condition is “skipNotifications is true”. Operationally it means “Should skip broadcast?”
- **Edge cases:**
  - If `skipNotifications` is missing, strict validation may fail or evaluate unexpectedly depending on n8n IF behavior.

#### Node: Skip Notification Broadcast
- **Type / role:** `n8n-nodes-base.noOp` — sink when skipping.
- **Failure types:** none.

---

### Block 5 — Notification & Escalation
**Overview:** Uses LLM agents to generate (1) Telegram-safe alert text for Slack/Telegram, and (2) structured Jira ticket fields. Sends messages to Telegram/Slack and creates Jira issues for critical/major incidents.

**Nodes involved:**
- OpenAI Chat Model
- CloudFlare Alert Bot
- Alert team via Telegram
- Send message via Slack1
- OpenAI Chat Model1
- Structured Output Parser
- Support Request Reader Agent
- Submit JIRA request ticket
- Setup Jira, Slack, Email
- Send message to IT Support team

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for the alert bot.
- **Model:** `gpt-5-mini`
- **Output connection:** Provides `ai_languageModel` to **CloudFlare Alert Bot**.
- **Failure types:** invalid API key, model not available in your account/region, rate limits.

#### Node: CloudFlare Alert Bot
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates the broadcast alert text.
- **Input text:** Uses `{{ $('UTIL: Prepare Notification Payload').item.json.aiPromptContext }}` (a JSON string).
- **System message highlights:**
  - Summarize each unresolved incident, add run stats.
  - Convert timestamps to Singapore time in a specific format.
  - Output must be **plain text**; avoid Telegram-breaking characters.
  - If `skipNotifications true`, output exactly: `No active Cloudflare incidents at the moment.`
- **Outputs:** → **Alert team via Telegram** and → **Send message via Slack1**
- **Edge cases / failure types:**
  - Telegram safety rule conflicts with downstream Telegram node using `parse_mode: HTML` (see below).
  - If `aiPromptContext` is very large (many incidents/updates), token limits may truncate.
  - If agent returns unexpected structure, downstream expects `{{$json.output}}`.

#### Node: Alert team via Telegram
- **Type / role:** `n8n-nodes-base.telegram`
- **Text:** `{{ $json.output }}`
- **Config:** `chatId: -1003460669186`, `parse_mode: HTML`, attribution disabled.
- **Potential issue:** The alert bot is instructed to output **plain text only** and avoid special characters, but this node enables HTML parsing. If the output contains `<`/`>` or entity-like strings, Telegram may interpret or reject formatting. Consider setting parse mode off or ensuring escaping.

#### Node: Send message via Slack1
- **Type / role:** `n8n-nodes-base.slack`
- **Text:** `{{ $json.output }}`
- **Channel:** `#social`
- **Option:** `includeLinkToWorkflow: false`
- **Failure types:** Slack credentials mismatch (this node uses a different credential name than “Send message via Slack”), missing permissions, channel not found.

#### Node: OpenAI Chat Model1
- **Type / role:** LangChain OpenAI chat model for Jira ticket extraction.
- **Model:** `gpt-4.1-mini`
- **Output connection:** Provides `ai_languageModel` to **Support Request Reader Agent**.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured`
- **Purpose:** Enforces a JSON output schema (example includes `summary`, `description`) for the Jira agent.
- **Output connection:** `ai_outputParser` → **Support Request Reader Agent**
- **Edge cases:** If the agent can’t comply, parsing fails and Jira creation will break.

#### Node: Support Request Reader Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — transforms a single incident JSON into Jira ticket fields.
- **Input text:** Embeds full incident JSON: `{{ $json.toJsonString() }}`
- **System message:** Requires structured text suitable for Jira mapping; but node is configured with `hasOutputParser: true`, so practically it must conform to the structured parser expectations.
- **Output:** → **Submit JIRA request ticket**
- **Potential inconsistency:** System message says “Do NOT return JSON”, but a structured output parser is attached with a JSON schema example. In practice, the parser will push the agent to output JSON-like structure. If the model follows the system message strictly, the parser may fail.

#### Node: Submit JIRA request ticket
- **Type / role:** `n8n-nodes-base.jira` — creates an issue in Jira Software Cloud.
- **Config:**
  - Project: “Support” (id `10003`)
  - Issue type: “Task” (id `10005`)
  - Summary: `{{ $json.output.summary }}`
  - Description: `{{ $json.output.description }}`
  - Additional: assignee fixed ID; priority set to “High” (value `2`)
- **Outputs:** → **Setup Jira, Slack, Email**
- **Failure types:** Jira auth issues, permission denied to create issues, invalid field IDs, assignee not assignable, rate limits.

#### Node: Setup Jira, Slack, Email
- **Type / role:** `n8n-nodes-base.set` — centralizes constants for notifications.
- **Fields set:**
  - `Jira base URL`: `https://your-company.atlassian.net`
  - `IT support slack channel`: `social`
  - `IT support email`: `it@your-company.com` (not used elsewhere in this workflow)
- **Outputs:** → **Send message to IT Support team**

#### Node: Send message to IT Support team
- **Type / role:** `n8n-nodes-base.slack` — posts the Jira link to IT support channel.
- **Text:** Includes Jira base URL + issue key from “Submit JIRA request ticket”.
- **Channel expression:** `={{ $json['IT support slack channel'] }}` (value “social”; Slack node expects a channel identifier depending on mode—here it uses “name” mode).
- **Failure types / edge cases:**
  - If channel should be `#social` but value is `social`, behavior depends on Slack node; may fail to resolve.
  - Jira issue key expression references `$('Submit JIRA request ticket').item.json.key`—fails if Jira node errors or returns a different structure.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled entry point (every 30 min) | — | Decodo HTTP Request, CloudFlare OK? | ## SECTION 1 — Ingestion & Detection<br><br>Fetches the latest Cloudflare service status on a fixed schedule and checks for active or unresolved incidents.<br>If no incidents are detected, the workflow exits early to avoid unnecessary processing. |
| Decodo HTTP Request | @decodo/n8n-nodes-decodo.decodo | Fetch Cloudflare unresolved incidents | Schedule Trigger | CloudFlare OK? | ## SECTION 1 — Ingestion & Detection<br><br>Fetches the latest Cloudflare service status on a fixed schedule and checks for active or unresolved incidents.<br>If no incidents are detected, the workflow exits early to avoid unnecessary processing. |
| CloudFlare OK? | if | Branch: no incidents vs incidents exist | Decodo HTTP Request, Schedule Trigger | Is top of hour?, Enrich Incidents & Score | ## SECTION 1 — Ingestion & Detection<br><br>Fetches the latest Cloudflare service status on a fixed schedule and checks for active or unresolved incidents.<br>If no incidents are detected, the workflow exits early to avoid unnecessary processing. |
| Is top of hour? | if | Noise guard for “all clear” message | CloudFlare OK? | Send message via Telegram, Send message via Slack, No Operation, do nothing | ## SECTION 3 — Noise Control & Deduplication<br><br>Applies time-based guards and deduplication rules to prevent repeated or excessive alerts.<br>Previously notified incidents are filtered out to avoid alert fatigue. |
| Send message via Telegram | telegram | Send “all operational” Telegram message | Is top of hour? | — | ## SECTION 3 — Noise Control & Deduplication<br><br>Applies time-based guards and deduplication rules to prevent repeated or excessive alerts.<br>Previously notified incidents are filtered out to avoid alert fatigue. |
| Send message via Slack | slack | Send “all operational” Slack message | Is top of hour? | — | ## SECTION 3 — Noise Control & Deduplication<br><br>Applies time-based guards and deduplication rules to prevent repeated or excessive alerts.<br>Previously notified incidents are filtered out to avoid alert fatigue. |
| No Operation, do nothing | noOp | Sink when no “all clear” broadcast | Is top of hour? | — | ## SECTION 3 — Noise Control & Deduplication<br><br>Applies time-based guards and deduplication rules to prevent repeated or excessive alerts.<br>Previously notified incidents are filtered out to avoid alert fatigue. |
| Enrich Incidents & Score | code | Normalize incidents, compute severity, SGT timestamps, snapshot | CloudFlare OK? | Log Incidents, UTIL: Filter Already Alerted | ## SECTION 2 — Enrichment, Scoring & History<br><br>Normalizes incident data and derives severity, impact, and affected components.<br>All incidents are logged to support auditing, reporting, and post-incident reviews. |
| Log Incidents | googleSheets | Append incident records to Google Sheets | Enrich Incidents & Score | — | ## SECTION 2 — Enrichment, Scoring & History<br><br>Normalizes incident data and derives severity, impact, and affected components.<br>All incidents are logged to support auditing, reporting, and post-incident reviews. |
| UTIL: Filter Already Alerted | code | Dedupe incidents using workflow static data (30 min TTL) | Enrich Incidents & Score | UTIL: Prepare Notification Payload, High Impact Escalation? | ## SECTION 3 — Noise Control & Deduplication<br><br>Applies time-based guards and deduplication rules to prevent repeated or excessive alerts.<br>Previously notified incidents are filtered out to avoid alert fatigue. |
| High Impact Escalation? | if | Route critical/major to Jira escalation | UTIL: Filter Already Alerted | Support Request Reader Agent, Low Impact Buffer | ## SECTION 4 — Decision & Routing<br><br>Evaluates incident impact and determines the appropriate handling path.<br>High-impact incidents are escalated, while low-impact incidents may be buffered or skipped. |
| Low Impact Buffer | noOp | Placeholder sink for low-impact incidents | High Impact Escalation? | — | ## SECTION 4 — Decision & Routing<br><br>Evaluates incident impact and determines the appropriate handling path.<br>High-impact incidents are escalated, while low-impact incidents may be buffered or skipped. |
| UTIL: Prepare Notification Payload | code | Aggregate incidents into one AI/broadcast payload + stats | UTIL: Filter Already Alerted | Should broadcast notifications? | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Should broadcast notifications? | if | Skip vs send broadcast (based on skipNotifications) | UTIL: Prepare Notification Payload | Skip Notification Broadcast, CloudFlare Alert Bot | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Skip Notification Broadcast | noOp | Sink when broadcast is skipped | Should broadcast notifications? | — | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for alert text generation | — | CloudFlare Alert Bot (ai_languageModel) | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| CloudFlare Alert Bot | @n8n/n8n-nodes-langchain.agent | Generate Telegram-safe alert content from aggregated payload | Should broadcast notifications? + OpenAI Chat Model | Alert team via Telegram, Send message via Slack1 | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Alert team via Telegram | telegram | Send incident alert to Telegram | CloudFlare Alert Bot | — | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Send message via Slack1 | slack | Send incident alert to Slack (#social) | CloudFlare Alert Bot | — | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Jira ticket extraction | — | Support Request Reader Agent (ai_languageModel) | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured output for Jira fields | — | Support Request Reader Agent (ai_outputParser) | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Support Request Reader Agent | @n8n/n8n-nodes-langchain.agent | Build structured Jira “summary/description” from incident JSON | High Impact Escalation? + OpenAI Chat Model1 + Structured Output Parser | Submit JIRA request ticket | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Submit JIRA request ticket | jira | Create Jira issue for high-impact incidents | Support Request Reader Agent | Setup Jira, Slack, Email | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Setup Jira, Slack, Email | set | Set Jira base URL, Slack channel, email constants | Submit JIRA request ticket | Send message to IT Support team |  |
| Send message to IT Support team | slack | Notify IT support Slack channel with Jira link | Setup Jira, Slack, Email | — |  |
| Sticky Note7 | stickyNote | Branding + high-level workflow notes | — | — | # Cloudflare Incident Monitoring & Escalation Workflow<br>## 🚀 Try Decodo — Web Scraping & Data API (Coupon: **TRUNG**)<br>![Decodo Logo](https://s3.ap-southeast-1.amazonaws.com/automatewith.me/decodo-logo-black.jpg)<br>… |
| Sticky Note | stickyNote | Section 1 description | — | — | ## SECTION 1 — Ingestion & Detection<br><br>Fetches the latest Cloudflare service status on a fixed schedule and checks for active or unresolved incidents.<br>If no incidents are detected, the workflow exits early to avoid unnecessary processing. |
| Sticky Note1 | stickyNote | Section 2 description | — | — | ## SECTION 2 — Enrichment, Scoring & History<br><br>Normalizes incident data and derives severity, impact, and affected components.<br>All incidents are logged to support auditing, reporting, and post-incident reviews. |
| Sticky Note2 | stickyNote | Section 3 description | — | — | ## SECTION 3 — Noise Control & Deduplication<br><br>Applies time-based guards and deduplication rules to prevent repeated or excessive alerts.<br>Previously notified incidents are filtered out to avoid alert fatigue. |
| Sticky Note4 | stickyNote | Section 4 description | — | — | ## SECTION 4 — Decision & Routing<br><br>Evaluates incident impact and determines the appropriate handling path.<br>High-impact incidents are escalated, while low-impact incidents may be buffered or skipped. |
| Sticky Note3 | stickyNote | Section 5 description | — | — | ## SECTION 5 — Notification & Escalation<br><br>Prepares human-readable alert payloads and sends notifications to Slack and Telegram.<br>Critical incidents are escalated by creating structured JIRA incident tickets for tracking and response. |
| Sticky Note6 | stickyNote | Image reference | — | — | ![Alt text](https://s3.ap-southeast-1.amazonaws.com/automatewith.me/cloudflare-incident-monitoring-and-alerting-workflow-1.png) |
| Sticky Note5 | stickyNote | Image reference | — | — | ![Alt text](https://s3.ap-southeast-1.amazonaws.com/automatewith.me/cloudflare-incident-monitoring-and-alerting-workflow-2.png) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name: **Cloudflare Incident Monitoring & Alerting Workflow**
   - Keep it inactive until credentials are set.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Interval: every **30 minutes**

3) **Add HTTP fetch via Decodo**
   - Node: **Decodo HTTP Request** (Decodo node)
   - URL: `https://www.cloudflarestatus.com/api/v2/incidents/unresolved.json`
   - Credential: create/select **Decodo API** credential
   - Connect: `Schedule Trigger → Decodo HTTP Request`

4) **Add incident presence check**
   - Node: **IF** named `CloudFlare OK?`
   - Condition: array **is empty** on:
     - `{{ $json.data.results[0].content.parseJson().incidents }}`
   - Connect: `Decodo HTTP Request → CloudFlare OK?`
   - (Recommended fix when rebuilding: verify Decodo output; if Decodo returns raw JSON, change expression to `{{ $json.incidents }}`.)

5) **No-incident “top of hour” guard**
   - Node: **IF** named `Is top of hour?`
   - Condition: number equals `{{ $now.minute }}` == `0`
   - Connect (true/no-incident path): `CloudFlare OK? (true) → Is top of hour?`

6) **All-clear notifications**
   - Node: **Telegram** named `Send message via Telegram`
     - Chat ID: `-1003460669186`
     - Text: `✅ Cloudflare Status: All systems are operational. No active incidents at the moment.`
   - Node: **Slack** named `Send message via Slack`
     - Auth: **OAuth2**
     - Channel: `#cloudflare-alert`
     - Text: same as above
   - Connect: `Is top of hour? (true) → Telegram` and `Is top of hour? (true) → Slack`
   - Add **NoOp** node `No Operation, do nothing`
   - Connect: `Is top of hour? (false) → NoOp`

7) **Enrichment & scoring**
   - Node: **Code** named `Enrich Incidents & Score`
   - Paste the JS logic (severity map, emoji map, SGT conversion, `dedupeKey`, `platform_snapshot`).
   - Connect (incidents present): `CloudFlare OK? (false) → Enrich Incidents & Score`

8) **Logging to Google Sheets**
   - Node: **Google Sheets** named `Log Incidents`
   - Operation: **Append**
   - Document: select your sheet (create one named e.g. “Cloudflare Incidents”)
   - Sheet: `Sheet1`
   - Mapping: Auto-map (or define explicit column mapping for stability)
   - Credential: Google Sheets OAuth2
   - Connect: `Enrich Incidents & Score → Log Incidents`

9) **Deduplication**
   - Node: **Code** named `UTIL: Filter Already Alerted`
   - Implement static data TTL cache (30 minutes).
   - Connect: `Enrich Incidents & Score → UTIL: Filter Already Alerted`

10) **High-impact routing**
   - Node: **IF** named `High Impact Escalation?`
   - Condition: `impact` equals `critical` OR equals `major`
   - Connect: `UTIL: Filter Already Alerted → High Impact Escalation?`
   - Add **NoOp** named `Low Impact Buffer`
   - Connect: `High Impact Escalation? (false) → Low Impact Buffer`

11) **Prepare aggregated notification payload**
   - Node: **Code** named `UTIL: Prepare Notification Payload`
   - Build `incidents[]`, `stats`, `aiPromptContext` (stringified JSON), `skipNotifications`
   - Connect: `UTIL: Filter Already Alerted → UTIL: Prepare Notification Payload`

12) **Broadcast gate**
   - Node: **IF** named `Should broadcast notifications?`
   - Condition: boolean true on `{{ $json.skipNotifications }}`
     - True branch = skip
     - False branch = broadcast
   - Connect: `UTIL: Prepare Notification Payload → Should broadcast notifications?`
   - Add **NoOp** named `Skip Notification Broadcast`
   - Connect: `Should broadcast notifications? (true) → Skip Notification Broadcast`

13) **LLM for Slack/Telegram alerts**
   - Node: **OpenAI Chat Model** (LangChain)
     - Model: `gpt-5-mini`
     - Credential: OpenAI API key
   - Node: **AI Agent** named `CloudFlare Alert Bot`
     - Prompt: include `{{ $('UTIL: Prepare Notification Payload').item.json.aiPromptContext }}`
     - System message: Telegram-safe plain text + SGT conversion + run stats rules
   - Connect:
     - `Should broadcast notifications? (false) → CloudFlare Alert Bot`
     - OpenAI Chat Model `ai_languageModel → CloudFlare Alert Bot`

14) **Send alerts**
   - Node: **Telegram** named `Alert team via Telegram`
     - Text: `{{ $json.output }}`
     - Chat ID: `-1003460669186`
     - (Recommended: disable HTML parse mode to match “plain text only”.)
   - Node: **Slack** named `Send message via Slack1`
     - Channel: `#social`
     - Text: `{{ $json.output }}`
   - Connect: `CloudFlare Alert Bot → Telegram` and `CloudFlare Alert Bot → Slack1`

15) **LLM for Jira ticket fields (high impact only)**
   - Node: **OpenAI Chat Model** named `OpenAI Chat Model1`
     - Model: `gpt-4.1-mini`
   - Node: **Structured Output Parser**
     - Schema example: `{ "summary": "", "description": "" }`
   - Node: **AI Agent** named `Support Request Reader Agent`
     - Input text includes full incident JSON: `{{ $json.toJsonString() }}`
     - Attach output parser
   - Connect:
     - `High Impact Escalation? (true) → Support Request Reader Agent`
     - `OpenAI Chat Model1 ai_languageModel → Support Request Reader Agent`
     - `Structured Output Parser ai_outputParser → Support Request Reader Agent`

16) **Create Jira issue**
   - Node: **Jira** named `Submit JIRA request ticket`
   - Configure:
     - Project: Support
     - Issue type: Task
     - Summary: `{{ $json.output.summary }}`
     - Description: `{{ $json.output.description }}`
     - Assignee + Priority as desired
   - Credential: Jira Software Cloud OAuth/API token
   - Connect: `Support Request Reader Agent → Submit JIRA request ticket`

17) **Notify IT support in Slack with the Jira link**
   - Node: **Set** named `Setup Jira, Slack, Email`
     - `Jira base URL`, `IT support slack channel`, `IT support email`
   - Node: **Slack** named `Send message to IT Support team`
     - Channel: `={{ $json['IT support slack channel'] }}`
     - Text includes link: `{{ $json["Jira base URL"] }}/browse/{{ $('Submit JIRA request ticket').item.json.key }}`
   - Connect: `Submit JIRA request ticket → Setup Jira, Slack, Email → Send message to IT Support team`

18) **Add sticky notes (optional)**
   - Add sticky notes for Sections 1–5 and branding/images if you want to preserve the same canvas documentation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Decodo branding, discount coupon `TRUNG`, overview of Decodo scraping platform | Image: https://s3.ap-southeast-1.amazonaws.com/automatewith.me/decodo-logo-black.jpg |
| Workflow diagram image | https://s3.ap-southeast-1.amazonaws.com/automatewith.me/cloudflare-incident-monitoring-and-alerting-workflow-1.png |
| Workflow diagram image | https://s3.ap-southeast-1.amazonaws.com/automatewith.me/cloudflare-incident-monitoring-and-alerting-workflow-2.png |
| Workflow intent notes: reduce alert fatigue, dedupe, severity routing, Jira escalation | Captured in sticky notes “SECTION 1–5” on the canvas |

