Triage product UAT feedback with OpenAI, Jira, Slack, Notion and Google Sheets

https://n8nworkflows.xyz/workflows/triage-product-uat-feedback-with-openai--jira--slack--notion-and-google-sheets-12135


# Triage product UAT feedback with OpenAI, Jira, Slack, Notion and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Triage product UAT feedback with OpenAI, Jira, Slack, Notion and Google Sheets  
**Internal name:** *Product UAT Feedback Triage (AI) + Routing + Closed Loop*  
**Purpose:** Accept UAT feedback from a webhook, normalize it, run AI triage (type/severity/sentiment/summary/confidence), enforce quality control (JSON parsing + confidence threshold), then route to Jira/Notion/Sheets/archive and close the loop by notifying the tester (Slack or email) and responding to the webhook with a status payload.

### Logical blocks
**1.1 Ingestion & Configuration Merge**  
Webhook receives feedback; a configuration object is created and merged with the incoming payload.

**1.2 Normalization & Text Cleanup**  
Transforms various possible input shapes into a single `uat` model and creates `uat.message_clean`.

**1.3 AI Triage & Output Validation**  
OpenAI (LangChain) classifies the feedback and returns JSON; a parser validates/normalizes allowed values and confidence.

**1.4 Quality Gate (Human-in-the-loop)**  
If AI parse fails or confidence is low, escalates to manual review via email + Slack channel.

**1.5 Routing & Actions**  
Switch routes by triage type:
- Critical bugs → Jira + engineering Slack alert
- Feature requests → Notion dedupe search → update or create Notion database entry
- UX improvements → Google Sheets append (digest/log)
- Noise → Notion “archive” + Slack ask-for-more-info

**1.6 Closed Loop (Notify tester + Webhook response)**  
Composes a response message depending on branch and sends to tester via Slack DM or Gmail, then responds to the webhook.

---

## 2. Block-by-Block Analysis

### 2.1 Ingestion & Configuration Merge
**Overview:** Receives feedback via HTTP POST webhook. Builds a `cfg` object (thresholds, IDs, channels) and merges it with the incoming request for downstream nodes.

**Nodes involved:** `trigger`, `data mapping`, `data merge`

#### Node: `trigger`
- **Type/role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point.
- **Config choices:**  
  - HTTP Method: `POST`  
  - Path: `50b5cb50-43bb-42a5-9dda-1f50bf7ee356`
- **Inputs/outputs:** Entry node → outputs to `data merge`.
- **Version notes:** TypeVersion 2.1.
- **Failure modes:** Invalid/empty payload; caller timeouts if workflow is slow and webhook expects fast response (mitigated by `Respond to Webhook` at end, but still total runtime matters).

#### Node: `data mapping`
- **Type/role:** Set (`n8n-nodes-base.set`) — defines workflow configuration under `cfg.*`.
- **Key fields:**  
  - `cfg.jiraProjectKey = "UAT"`  
  - `cfg.jiraIssueTypeBug = "Bug"`  
  - `cfg.slackChannelEng = "#eng-uat"` (note: later Slack nodes use channel IDs, not this string)  
  - `cfg.slackChannelPm = "#product-uat"`  
  - `cfg.sheetIdDigest = "YourID"` (not used by the Sheets node, which uses a fixed documentId)  
  - `cfg.manualReviewEmail = "user@example.com"`  
  - `cfg.confidenceThreshold = 0.6`  
  - `cfg.dedupeEnabled = false` (not used anywhere)  
  - `cfg.llmProvider = "openai"`
- **Inputs/outputs:** No incoming connection shown (it feeds into `data merge` input 1). In n8n, nodes without an incoming connection won’t execute unless manually triggered or connected; in this JSON it *is* connected to `data merge` (as the second input), but you still need a path that triggers it. Here, it is connected only as an input to `data merge`, so it will execute when `data merge` runs.
- **Version notes:** TypeVersion 3.4.
- **Failure modes:** Misconfigured email/threshold values lead to misrouting or missed escalations.

#### Node: `data merge`
- **Type/role:** Merge (`n8n-nodes-base.merge`) — combines webhook payload and config.
- **Config choices:**  
  - Mode: `combine`  
  - Combine by: `combineByPosition` (item 0 from webhook combined with item 0 from config set)
- **Connections:**  
  - Input 0: `trigger`  
  - Input 1: `data mapping`  
  - Output → `normalize`
- **Version notes:** TypeVersion 3.2.
- **Failure modes:** If one input produces 0 items, combine-by-position can yield empty output (downstream nodes won’t run). Ensure webhook always yields one item and `data mapping` yields one item.

**Sticky note coverage:**  
- “## Ingestion & Normalization …” applies to nodes in the intake/normalize area (see table).

---

### 2.2 Normalization & Text Cleanup
**Overview:** Normalizes disparate incoming payload formats to a consistent `uat` object and prepares cleaned text for AI.

**Nodes involved:** `normalize`, `clean text`

#### Node: `normalize`
- **Type/role:** Code (`n8n-nodes-base.code`) — standardizes input.
- **Config choices (interpreted):**
  - Extracts from multiple possible fields: `source`, tester name/email, message, build version, URLs.
  - Defaults: `source="webhook"`, build `"unknown"`, missing strings become `""`.
  - Adds `uat.received_at` as ISO timestamp.
- **Key variables produced:**  
  - `uat.source`, `uat.tester_name`, `uat.tester_email`, `uat.message_raw`, `uat.build_version`, `uat.page_url`, `uat.screenshot_url`, `uat.received_at`
- **Connections:** `data merge` → `normalize` → `clean text`
- **Version notes:** TypeVersion 2.
- **Failure modes:** If inbound payload is not JSON or has unexpected structure, fields may be empty (AI triage quality drops).

#### Node: `clean text`
- **Type/role:** Code — sanitizes `uat.message_raw`.
- **Config choices:**
  - Removes HTML tags, normalizes whitespace, trims.
  - Limits to 3000 chars (`slice(0, 3000)`).
  - Outputs `uat.message_clean`.
- **Connections:** `normalize` → `clean text` → `AI agent`
- **Failure modes:** If `uat.message_raw` is missing, `message_clean` becomes empty, leading to “Noise” or low confidence.

**Sticky note coverage:**  
- “## Ingestion & Normalization …” also contextually covers these steps.

---

### 2.3 AI Triage & Output Validation
**Overview:** Calls an OpenAI model to produce strict JSON triage output, then parses and normalizes it to safe allowed values.

**Nodes involved:** `AI agent`, `parsing and validation`

#### Node: `AI agent`
- **Type/role:** OpenAI via LangChain (`@n8n/n8n-nodes-langchain.openAi`) — text classification + structured generation.
- **Config choices:**
  - Model: `gpt-5.2`
  - Prompt instructs: “return ONLY JSON” with strict schema:
    - `sentiment`: Positive|Negative  
    - `type`: CriticalBug|UXImprovement|FeatureRequest|Noise  
    - `severity`: Blocker|Critical|Major|Minor  
    - `summary` (≤160 chars), `suggested_title` (≤80 chars)
    - `components`: constrained list `[login, onboarding, checkout, search, profile, settings, performance, ui, api, other]`
    - `repro_steps`: array; empty if not inferable
    - `confidence`: numeric
  - Includes context values from `uat`.
- **Connections:** `clean text` → `AI agent` → `parsing and validation`
- **Version notes:** TypeVersion 2.
- **Failure modes:**  
  - Model returns non-JSON or extra text → parse failure downstream.  
  - Rate limiting/auth errors from OpenAI credential.  
  - Hallucinated fields not matching allowed enums (handled downstream by normalization).

#### Node: `parsing and validation`
- **Type/role:** Code — parses JSON and clamps/validates.
- **Config choices:**
  - Reads raw model output from several possible fields (`$json.output`, `$json.text`, `$json.response...`) to be resilient across node output shapes.
  - If JSON parse fails: creates fallback triage (`type=Noise`, `severity=Minor`, `confidence=0`, `parse_ok=false`).
  - Validates enums and forces defaults if invalid.
  - Clamps confidence to `[0,1]`.
- **Outputs:** Adds/overwrites `$json.triage` with `parse_ok`.
- **Connections:** `AI agent` → `parsing and validation` → `manual review`
- **Failure modes:** If the OpenAI node output structure changes and none of the probed paths contain content, `raw=""` → parse fails → forced manual review path.

**Sticky note coverage:**  
- “## AI Triage & Quality Control …” applies to these nodes and the next quality gate.

---

### 2.4 Quality Gate (Human-in-the-loop)
**Overview:** Decides whether to accept AI triage or escalate to humans based on parse success and confidence threshold. Escalation sends a detailed email and a Slack message.

**Nodes involved:** `manual review`, `email code`, `manual review needed`, `Send a message`

#### Node: `manual review`
- **Type/role:** IF (`n8n-nodes-base.if`) — quality gate.
- **Logic (interpreted):** Uses **OR**:
  1) `triage.parse_ok` is false  
  2) `triage.confidence < cfg.confidenceThreshold` (0.6 by default)
- **Connections:**  
  - True (manual review needed) → `email code`  
  - False (ok to auto-route) → `route by triage type`
- **Version notes:** TypeVersion 2.2.
- **Potential issue:** The first condition’s rightValue references `cfg.confidenceThreshold` but is not used by the boolean operator; it’s harmless but confusing.
- **Failure modes:** Missing `cfg.confidenceThreshold` would make the numeric comparison expression fail or evaluate unexpectedly.

#### Node: `email code`
- **Type/role:** Code — composes `mail.subject` and `mail.body`.
- **Key content:** Includes full context (source/build/page/screenshot), tester info, cleaned feedback, AI triage output, and instructions.
- **Connections:** `manual review` (true) → `email code` → `manual review needed`
- **Failure modes:** If any expected fields are undefined, template still mostly works due to nullish coalescing; body may contain blanks.

#### Node: `manual review needed`
- **Type/role:** Gmail send (`n8n-nodes-base.gmail`) — emails reviewers.
- **Config:**  
  - To: `cfg.manualReviewEmail`  
  - Subject: `Manual UAT triage needed (confidence X)`  
  - Body: `mail.body`
- **Connections:** `email code` → `manual review needed` → `Send a message`
- **Version notes:** TypeVersion 2.1.
- **Failure modes:** Gmail OAuth scope/consent issues; quota limits.

#### Node: `Send a message`
- **Type/role:** Slack message (`n8n-nodes-base.slack`) — pings a channel for manual review.
- **Config:**  
  - Text: “Manual review needed”  
  - Channel: `C0A030UC0BW` (cached name “data”)  
  - Auth: OAuth2
- **Connections:** `manual review needed` → `Send a message` → `Webhook response`
- **Failure modes:** Wrong channel ID, Slack token revoked, missing chat:write.

---

### 2.5 Routing, Actions & Closed Loop Preparation
**Overview:** Routes validated feedback to the right system depending on `triage.type`, then composes an appropriate reply for the tester.

**Nodes involved:** `route by triage type`, plus branch-specific nodes:
- Critical bug: `critical bug` → `engeneering alert` → `compose reply branch 1`
- Feature request: `double check` → `if found` → (`update notion database` or `create notion database`) → `compose reply branch 2`
- UX improvement: `Append row in sheet` → `compose reply branch 3`
- Noise / unclear: `archive` → `ask for more information` → `compose reply branch 4`

#### Node: `route by triage type`
- **Type/role:** Switch (`n8n-nodes-base.switch`) — router.
- **Rules:** exact equals on `triage.type`:
  1) `CriticalBug`
  2) `FeatureRequest`
  3) `UXImprovement`
  4) `Noise`
- **Connections (one output per rule in order):**
  - 1 → `critical bug`
  - 2 → `double check`
  - 3 → `Append row in sheet`
  - 4 → `archive`
- **Version notes:** TypeVersion 3.3.
- **Failure modes:** If `triage.type` is missing/invalid (should be normalized earlier), item may not match any rule and silently drop depending on node settings (here options are empty; ensure a default route if desired).

---

#### Branch A — Critical Bug (Jira + Eng Slack)
**Overview:** Creates a Jira bug with rich description, then alerts engineering in Slack, then prepares a confirmation reply to the tester.

**Nodes involved:** `critical bug`, `engeneering alert`, `compose reply branch 1`

##### Node: `critical bug`
- **Type/role:** Jira create issue (`n8n-nodes-base.jira`).
- **Config choices:**
  - Project: ID `10002` (cached “test2”) (note: `cfg.jiraProjectKey` is not used here)
  - Issue Type: ID `10013` (“Bug”)
  - Priority: ID `2` (“High”)
  - Summary: `triage.suggested_title || triage.summary`
  - Description includes build/page/screenshot, repro steps, reporter, source.
- **Connections:** `route by triage type` → `critical bug` → `engeneering alert`
- **Failure modes:** Jira auth; project/issueType IDs wrong across environments; required fields missing; Jira description formatting.

##### Node: `engeneering alert`
- **Type/role:** Slack channel message.
- **Config:** Posts “🚨 UAT Critical Bug (severity) … Jira: key/id” to channel `C09V1228324` (“tous-n8n”).
- **Connections:** `critical bug` → `engeneering alert` → `compose reply branch 1`
- **Failure modes:** Slack auth or channel ID mismatch.

##### Node: `compose reply branch 1`
- **Type/role:** Set — prepares tester reply for critical bug.
- **Fields set:**
  - `reply.subject = "UAT feedback received — Ticket <key>"`  
  - `reply.body` includes severity, ticket key, summary, build.
- **Connections:** `engeneering alert` → `compose reply branch 1` → `how to contact`
- **Failure modes:** If Jira node output doesn’t include `key`/`id`, message will be less useful.

---

#### Branch B — Feature Request (Notion dedupe + create/update)
**Overview:** Searches Notion for similar items (by suggested title). If found, updates an existing database page; otherwise, creates a new database page. Then prepares tester acknowledgement.

**Nodes involved:** `double check`, `if found`, `update notion database`, `create notion database`, `compose reply branch 2`

##### Node: `double check`
- **Type/role:** Notion search (`n8n-nodes-base.notion`) — dedupe-like step.
- **Config:** Operation `search` with query text = `triage.suggested_title`.
- **Connections:** `route by triage type` (FeatureRequest) → `double check` → `if found`
- **Failure modes:** Notion search is broad; results may be irrelevant; requires Notion integration access.

##### Node: `if found`
- **Type/role:** IF — checks search result length.
- **Condition:** `($json.results || []).length > 0`
- **Connections:**
  - True → `update notion database`
  - False → `create notion database`
- **Failure modes:** Notion node output shape changes; `results` missing.

##### Node: `update notion database`
- **Type/role:** Notion database page update.
- **Config:** Resource `databasePage`, operation `update`, `pageId` is a placeholder URL expression `=youridpage.com` (not derived from search result).
- **Connections:** `if found` (true) → `update notion database` → `compose reply branch 2`
- **Critical integration gap:** This node does not map the found page ID from `double check` results; it uses a static placeholder. In practice, it will fail unless replaced with an expression pointing to the first match’s page ID.
- **Failure modes:** Page ID invalid; missing properties; Notion permissions.

##### Node: `create notion database`
- **Type/role:** Notion create database page.
- **Config:** Database ID `2b311ca2-096c-8049-a5ab-de07d643edca`. Title “Add Roadmap Idea”. Adds a block with template-like content (not real property mapping).
- **Connections:** `if found` (false) → `create notion database` → `compose reply branch 2`
- **Failure modes:** Database schema mismatch; missing required properties; the node content appears to add blocks, not populate database properties (may not achieve intended structured fields).

##### Node: `compose reply branch 2`
- **Type/role:** Set — tester reply for feature request.
- **Fields:**
  - Subject: “UAT feedback received — Feature request logged”
  - Body: thanks + includes `triage.summary`
- **Connections:** `create notion database`/`update notion database` → `compose reply branch 2` → `how to contact`

---

#### Branch C — UX Improvement (Log to Google Sheets)
**Overview:** Appends the feedback to a Google Sheet (digest/log), then prepares a UX acknowledgement reply.

**Nodes involved:** `Append row in sheet`, `compose reply branch 3`

##### Node: `Append row in sheet`
- **Type/role:** Google Sheets append row.
- **Config choices:**
  - Operation: `append`
  - Spreadsheet: `users feedback` (documentId `1jah3Nvy3GkFp_2fDd7NTkTgqpiI6G91WIlRDd-5y-uo`)
  - Sheet tab: “Feuille 1” (`gid=0`)
  - Columns schema includes fields like Timestamp, Feedback ID, Source, Category, Sentiment, Priority, Jira URL, Status, etc.
  - Mapping mode: `autoMapInputData` but **no explicit mapping provided**.
- **Connections:** `route by triage type` (UXImprovement) → `Append row in sheet` → `compose reply branch 3`
- **Failure modes:** With auto-map and no prepared matching keys, appended rows may be blank or misaligned. Requires upstream `Set`/`Code` that creates keys matching the column names.

##### Node: `compose reply branch 3`
- **Type/role:** Set — tester reply for UX feedback.
- **Fields:** subject “UX note captured”; body thanks and references `triage.components.join(', ')`.
- **Connections:** `Append row in sheet` → `compose reply branch 3` → `how to contact`
- **Failure modes:** If `triage.components` is not an array, `.join` may error (parser normalizes `components` but doesn’t guarantee array type if model returns wrong type; consider hardening).

---

#### Branch D — Noise / Unclear (Archive + ask for more info)
**Overview:** Archives/logs in Notion (placeholder), pings a Slack user to ask for more info, then prepares a follow-up message to the tester.

**Nodes involved:** `archive`, `ask for more information`, `compose reply branch 4`

##### Node: `archive`
- **Type/role:** Notion node (operation unclear from config; appears to create/update a page).
- **Config:** Has `pageId` placeholder `=yourpage./notion.com` and title “Choose a title”.
- **Connections:** `route by triage type` (Noise) → `archive` → `ask for more information`
- **Failure modes:** Placeholder pageId will fail; also unclear whether this matches desired “archive” semantics.

##### Node: `ask for more information`
- **Type/role:** Slack DM to a user.
- **Config:** Sends “We need more info for the UAT, please have a look” to user `U09UKKK9R25` (“analyticsn8n”).
- **Connections:** `archive` → `ask for more information` → `compose reply branch 4`
- **Failure modes:** User ID incorrect; bot cannot DM user; token lacks permission.

##### Node: `compose reply branch 4`
- **Type/role:** Set — follow-up asking tester for details.
- **Connections:** `ask for more information` → `compose reply branch 4` → `how to contact`

---

### 2.6 Closed Loop: Notify Tester + Respond to Webhook
**Overview:** Chooses Slack DM vs email based on `uat.source`, sends the message, then returns a structured response to the original webhook caller.

**Nodes involved:** `how to contact`, `slack tester`, `tester email`, `Webhook response`

#### Node: `how to contact`
- **Type/role:** IF — decides communication channel.
- **Condition:** `uat.source == "slack"`
- **Connections:**
  - True → `slack tester`
  - False → `tester email`
- **Failure modes:** If `uat.source` is missing or not set correctly upstream, may default to email even for Slack-originated feedback.

#### Node: `slack tester`
- **Type/role:** Slack DM.
- **Config:** Sends `reply.body` to user `U09UKKK9R25` (static).  
  *Note:* This does not DM the actual tester; it always messages the same user.
- **Connections:** `how to contact` (true) → `slack tester` → `Webhook response`
- **Failure modes:** Static user is likely wrong for real testers; needs mapping from inbound payload.

#### Node: `tester email`
- **Type/role:** Gmail send — emails tester.
- **Config:**  
  - To: `uat.tester_email`  
  - Subject/body from `reply.subject` / `reply.body`
- **Connections:** `how to contact` (false) → `tester email` → `Webhook response`
- **Failure modes:** Missing tester email; Gmail OAuth/auth/quota.

#### Node: `Webhook response`
- **Type/role:** Respond to Webhook (`n8n-nodes-base.respondToWebhook`) — closes HTTP request.
- **Config choices:**
  - Respond with: `allIncomingItems`
  - Response code: `200`
  - “responseKey” is used to shape a payload, but it contains templating inside a JSON-like string:
    ```json
    { "status":"received", "type":"{{ $json.triage.type }}", ... }
    ```
    In practice, n8n expects an expression that evaluates to an object/string. This configuration is likely to be brittle depending on node behavior.
- **Connections:** Receives from `slack tester` or `tester email` or manual-review Slack `Send a message`.
- **Failure modes:** If webhook expects immediate response, waiting for all actions (Jira/Notion/Sheets) can cause timeouts. Consider responding earlier and processing asynchronously.

**Sticky note coverage:**  
- “## Closed Loop …” applies to `how to contact`, `slack tester`, `tester email`, `Webhook response`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| trigger | Webhook | Receives UAT feedback via HTTP POST | — | data merge | ## Ingestion & Normalization  Collects UAT feedback from a webhook and normalizes all inputs (tester, source, project, message) into a clean, AI-ready format. |
| data mapping | Set | Defines `cfg.*` configuration (thresholds, emails, IDs) | — (executes via merge input) | data merge | ## Ingestion & Normalization  Collects UAT feedback from a webhook and normalizes all inputs (tester, source, project, message) into a clean, AI-ready format. |
| data merge | Merge | Combines webhook payload with config | trigger; data mapping | normalize | ## Ingestion & Normalization  Collects UAT feedback from a webhook and normalizes all inputs (tester, source, project, message) into a clean, AI-ready format. |
| normalize | Code | Normalizes input into `uat.*` model | data merge | clean text | ## Ingestion & Normalization  Collects UAT feedback from a webhook and normalizes all inputs (tester, source, project, message) into a clean, AI-ready format. |
| clean text | Code | Sanitizes and truncates feedback text | normalize | AI agent | ## Ingestion & Normalization  Collects UAT feedback from a webhook and normalizes all inputs (tester, source, project, message) into a clean, AI-ready format. |
| AI agent | OpenAI (LangChain) | Produces structured triage JSON | clean text | parsing and validation | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| parsing and validation | Code | Parses/validates AI JSON; clamps enums/confidence | AI agent | manual review | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| manual review | IF | Quality gate: low confidence/parse failure → escalate | parsing and validation | email code; route by triage type | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| email code | Code | Builds manual review email payload | manual review (true) | manual review needed | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| manual review needed | Gmail | Emails reviewer for human triage | email code | Send a message | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| Send a message | Slack | Notifies Slack channel that manual review is needed | manual review needed | Webhook response | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| route by triage type | Switch | Routes by `triage.type` | manual review (false) | critical bug; double check; Append row in sheet; archive | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| critical bug | Jira | Creates Jira bug for critical issues | route by triage type | engeneering alert | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| engeneering alert | Slack | Alerts engineering channel about critical bug | critical bug | compose reply branch 1 | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| compose reply branch 1 | Set | Builds tester reply for critical bug branch | engeneering alert | how to contact | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| double check | Notion | Searches Notion to detect duplicates (feature requests) | route by triage type | if found | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| if found | IF | If Notion search has results → update else create | double check | update notion database; create notion database | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| update notion database | Notion | Updates existing Notion DB page (placeholder pageId) | if found (true) | compose reply branch 2 | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| create notion database | Notion | Creates new Notion DB page for feature request | if found (false) | compose reply branch 2 | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| compose reply branch 2 | Set | Builds tester reply for feature request branch | update notion database; create notion database | how to contact | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| Append row in sheet | Google Sheets | Appends UX feedback entry to spreadsheet | route by triage type | compose reply branch 3 | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| compose reply branch 3 | Set | Builds tester reply for UX improvement branch | Append row in sheet | how to contact | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| archive | Notion | Archives/logs Noise feedback (placeholder pageId) | route by triage type | ask for more information | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| ask for more information | Slack | DMs internal user to request more details | archive | compose reply branch 4 | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| compose reply branch 4 | Set | Builds tester follow-up asking for details | ask for more information | how to contact | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| how to contact | IF | Chooses Slack DM vs email based on `uat.source` | compose reply branch 1; compose reply branch 2; compose reply branch 3; compose reply branch 4 | slack tester; tester email | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| slack tester | Slack | Sends reply to a Slack user (static user ID) | how to contact (true) | Webhook response | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| tester email | Gmail | Emails reply to tester | how to contact (false) | Webhook response | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| Webhook response | Respond to Webhook | Returns status payload to the webhook caller | slack tester; tester email; Send a message | — | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |
| Sticky Note4 | Sticky Note | Comment: How it works | — | — | ## How it works  This workflow automates Product UAT feedback triage by combining AI analysis, human-in-the-loop validation, and smart routing across product tools.  When a tester submits feedback via a webhook (form, Slack, or internal tool), the workflow normalizes and cleans the input into a consistent data model. An AI model then analyzes the feedback to classify its type, assess urgency and sentiment, generate a short summary, and assign a confidence score.  Low-confidence or invalid AI results are automatically escalated to manual review via email and Slack, ensuring safe and reliable automation.  When confidence is sufficient, feedback is routed to the appropriate tools (Jira, Notion, logs) and the workflow closes the loop by notifying the tester and responding to the original webhook with a structured status payload. |
| Sticky Note | Sticky Note | Comment: Ingestion & Normalization | — | — | ## Ingestion & Normalization  Collects UAT feedback from a webhook and normalizes all inputs (tester, source, project, message) into a clean, AI-ready format. |
| Sticky Note1 | Sticky Note | Comment: AI Triage & Quality Control | — | — | ## AI Triage & Quality Control  An AI model analyzes the feedback and returns a structured triage (type, urgency, summary, confidence).  Low-confidence or invalid results are automatically flagged for manual review to ensure safe automation. |
| Sticky Note2 | Sticky Note | Comment: Routing, Actions & Closed Loop | — | — | ## Routing, Actions & Closed Loop  Routes validated feedback to the right tools: bugs to Jira, feature and UX feedback to Notion, and non-actionable inputs to archive or logs. |
| Sticky Note3 | Sticky Note | Comment: Closed Loop | — | — | ## Closed Loop  Notifies the tester via Slack or email and responds to the original webhook with a structured status payload. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook Trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: generate or set to your desired endpoint
   - Keep response handled later by “Respond to Webhook”.

2) **Create Configuration Node**
   - Add node: **Set** (name it `data mapping`)
   - Add fields (as `cfg.*`):
     - `cfg.manualReviewEmail` (string)
     - `cfg.confidenceThreshold` (number, e.g., 0.6)
     - (Optional) `cfg.jiraProjectKey`, etc. (note: current Jira node uses numeric project/issuetype IDs, not this key)

3) **Merge Webhook Payload + Config**
   - Add node: **Merge** (name `data merge`)
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - Webhook → Merge input 1
     - Set (`data mapping`) → Merge input 2

4) **Normalize Incoming Payload**
   - Add node: **Code** (name `normalize`)
   - Implement logic to produce:
     - `uat.source`, `uat.tester_name`, `uat.tester_email`, `uat.message_raw`, `uat.build_version`, `uat.page_url`, `uat.screenshot_url`, `uat.received_at`
   - Connect: Merge → normalize

5) **Clean Text**
   - Add node: **Code** (name `clean text`)
   - Produce `uat.message_clean` by stripping HTML, condensing whitespace, truncating length.
   - Connect: normalize → clean text

6) **AI Triage**
   - Add node: **OpenAI (LangChain)**
   - Credential: **OpenAI API**
   - Model: choose your model (workflow uses `gpt-5.2`)
   - Prompt: instruct to return **ONLY JSON** with the schema and rules (type/severity/sentiment/components/repro_steps/confidence).
   - Connect: clean text → AI node

7) **Parse and Validate AI Output**
   - Add node: **Code** (name `parsing and validation`)
   - Parse JSON; on failure set fallback triage with `parse_ok=false`.
   - Enforce allowed enums; clamp confidence to `[0,1]`.
   - Connect: AI node → parsing and validation

8) **Quality Gate**
   - Add node: **IF** (name `manual review`)
   - Conditions (OR):
     - `triage.parse_ok` is false
     - `triage.confidence` < `cfg.confidenceThreshold`
   - Connect: parsing and validation → manual review

9) **Manual Review Escalation Path**
   1. Add **Code** node `email code` to build `mail.subject` and `mail.body`.
   2. Add **Gmail** node `manual review needed`
      - OAuth2 credential (Gmail)
      - To: `{{$json.cfg.manualReviewEmail}}`
      - Subject/body from `mail.*`
   3. Add **Slack** node `Send a message`
      - OAuth2 credential (Slack)
      - Post to a channel (choose channel)
   4. Connect:
      - manual review (true) → email code → manual review needed → Send a message

10) **Auto-routing Path**
   - Add **Switch** node `route by triage type`
   - Route on `{{$json.triage.type}}` with cases:
     - CriticalBug, FeatureRequest, UXImprovement, Noise
   - Connect: manual review (false) → route by triage type

11) **CriticalBug Branch**
   1. Add **Jira Software Cloud** node `critical bug` (Create Issue)
      - Configure project + issue type (IDs or names)
      - Summary from `triage.suggested_title` / `triage.summary`
      - Description include build/page/screenshot/repro/tester
      - Credential: Jira OAuth/API token
   2. Add **Slack** node `engeneering alert` to post alert to engineering channel
   3. Add **Set** node `compose reply branch 1` with `reply.subject` and `reply.body`
   4. Connect: Switch(CriticalBug) → Jira → Slack → Set

12) **FeatureRequest Branch (Notion)**
   1. Add **Notion** node `double check` (Search) using `triage.suggested_title`
   2. Add **IF** node `if found`: `results.length > 0`
   3. Add **Notion** node `update notion database` (Update database page)
      - IMPORTANT: set `pageId` from the **search results** (e.g., first match)
   4. Add **Notion** node `create notion database` (Create database page)
      - Select your databaseId
      - Map database properties (recommended) rather than only adding blocks
   5. Add **Set** node `compose reply branch 2`
   6. Connect: Switch(FeatureRequest) → Search → IF → (Update/Create) → compose reply branch 2

13) **UXImprovement Branch (Google Sheets)**
   1. Add **Google Sheets** node (Append)
      - OAuth2 credential
      - Pick document and sheet
      - Create a prior **Set/Code** step (recommended) to map keys matching your column names (Timestamp, Source, Sentiment, etc.)
   2. Add **Set** node `compose reply branch 3`
   3. Connect: Switch(UXImprovement) → Sheets → compose reply branch 3

14) **Noise Branch (Archive + internal ping)**
   1. Add **Notion** node `archive`
      - Configure a valid target (database/page) for archiving/logging
   2. Add **Slack** node `ask for more information` (DM internal user or channel)
   3. Add **Set** node `compose reply branch 4`
   4. Connect: Switch(Noise) → Notion archive → Slack DM → compose reply branch 4

15) **Decide Reply Channel (Slack vs Email)**
   - Add **IF** node `how to contact`: `uat.source == "slack"`
   - Connect all `compose reply branch *` nodes → `how to contact`

16) **Send Reply**
   - Add **Slack** node `slack tester` (DM)
     - Recommended: use tester Slack user ID from input, not a static ID
   - Add **Gmail** node `tester email`
     - To: `uat.tester_email`
     - Subject/body: `reply.subject` / `reply.body`
   - Connect:
     - how to contact (true) → slack tester
     - how to contact (false) → tester email

17) **Respond to Webhook**
   - Add node: **Respond to Webhook**
   - Return code: `200`
   - Return a structured object containing `triage.type`, `triage.severity`, `triage.confidence`
   - Connect:
     - slack tester → Respond to Webhook
     - tester email → Respond to Webhook
     - (manual review Slack message) → Respond to Webhook (if you also want the webhook caller notified of escalation)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” sticky note explains the intended architecture: normalization → AI triage → confidence gate → routing → closed loop response. | Internal workflow documentation (Sticky Note “How it works”). |
| Several nodes contain placeholder Notion page URLs/IDs (`=youridpage.com`, `=yourpage./notion.com`). These must be replaced with real Notion page/database IDs and proper expressions (especially for updating the page found by search). | Notion integration correctness. |
| Slack “tester” notification uses a static user ID (`U09UKKK9R25`) rather than the tester identity from the webhook payload; for real closed-loop messaging, pass tester Slack user ID in the webhook and map it. | Closed loop reliability. |
| Google Sheets append is set to auto-map but upstream data does not create column-matching keys; add a mapping step to populate the sheet consistently. | Data logging accuracy. |
| Jira project/issue type are configured by numeric IDs; migrating environments typically requires updating these IDs or switching to key/name-based configuration if supported. | Portability across Jira instances. |