Triage product UAT feedback with OpenAI, Notion, Slack and Gmail

https://n8nworkflows.xyz/workflows/triage-product-uat-feedback-with-openai--notion--slack-and-gmail-12208


# Triage product UAT feedback with OpenAI, Notion, Slack and Gmail

## 1. Workflow Overview

**Purpose:** This workflow receives Product UAT feedback via a webhook, uses an OpenAI model to triage/classify the feedback into structured metadata, deduplicates against a Notion database (search + update/create), and closes the loop by notifying the tester via **Slack DM or Gmail**, then returns a structured webhook response.

**Target use cases:**
- Automating triage of UAT feedback (feature requests, bugs, UX improvements, noise)
- Preventing duplicate backlog items in Notion
- Providing confirmation back to testers with consistent messaging and traceability

### 1.1 Ingestion & Configuration Merge
Receives inbound feedback and merges it with static configuration values (channels, thresholds, etc.) before normalizing fields.

### 1.2 Normalization & Text Cleaning
Standardizes different possible payload shapes into a single `uat.*` schema and cleans the text for LLM processing.

### 1.3 AI Triage + JSON Parsing/Validation
Calls an OpenAI chat model with a strict JSON schema, then parses/validates the response and applies safe defaults on parse failure.

### 1.4 Notion Dedupe & Upsert
Searches Notion by suggested title/summary; updates an existing page if found, otherwise creates a new page in a specified database.

### 1.5 Closed Loop Notifications + Webhook Response
Composes a tester reply, routes by `uat.source` (Slack vs email), sends the message, and responds to the original webhook with triage metadata.

---

## 2. Block-by-Block Analysis

### Block 1 — Ingestion & Configuration Merge

**Overview:** Receives feedback via webhook and attaches static configuration values via a Set node, then merges both streams into one item for downstream processing.

**Nodes Involved:** `trigger`, `data mapping`, `data merge`

#### Node: `trigger`
- **Type / Role:** Webhook (Trigger) — entry point.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: `0c47919b-ae34-4016-b11b-ff84c49c036e` (also used as webhookId)
- **Inputs/Outputs:**
  - No inputs (entry node)
  - Output → `data merge` (index 0)
- **Edge cases / failures:**
  - Missing or unexpected payload fields (handled later in `normalize`)
  - If webhook is not activated / wrong URL used by client
- **Version notes:** typeVersion **2.1** (Webhook node behavior differs slightly across n8n versions; keep compatible with current instance)

#### Node: `data mapping`
- **Type / Role:** Set — injects configuration constants under `cfg.*`.
- **Configuration choices:**
  - Creates:
    - `cfg.jiraProjectKey = "UAT"`
    - `cfg.jiraIssueTypeBug = "Bug"`
    - `cfg.slackChannelEng = "#eng-uat"`
    - `cfg.slackChannelPm = "#product-uat"`
    - `cfg.sheetIdDigest = "YourID"`
    - `cfg.manualReviewEmail = "user@example.com"`
    - `cfg.confidenceThreshold = 0.6`
    - `cfg.dedupeEnabled = false`
    - `cfg.llmProvider = "openai"`
  - Note: Several cfg fields are **not used** elsewhere in this workflow (Jira, Sheets, confidence threshold, dedupeEnabled).
- **Inputs/Outputs:**
  - No incoming connection shown in the JSON; it outputs → `data merge` (index 1).
  - **Important:** As wired, this node will not execute unless it is manually executed or connected to a trigger/flow. In a normal run, only `trigger` fires.
- **Edge cases / failures:**
  - None typical; but “unused config” can mislead operators.
  - If you intend the merge to always include these values, connect `trigger` → `data mapping` and then merge.

#### Node: `data merge`
- **Type / Role:** Merge — combines webhook item with config item.
- **Configuration choices:**
  - Mode: **combine**
  - Combine by: **position**
- **Inputs/Outputs:**
  - Input 0: from `trigger`
  - Input 1: from `data mapping` (but see caveat above)
  - Output → `normalize`
- **Edge cases / failures:**
  - If input 1 never arrives, output may only contain webhook data (depending on merge behavior and execution path).
  - With “combine by position,” mismatched item counts can drop/duplicate items.

---

### Block 2 — Normalization & Text Cleaning

**Overview:** Converts arbitrary inbound field names into a stable `uat` object and produces `uat.message_clean` for reliable LLM analysis.

**Nodes Involved:** `normalize`, `clean text`

#### Node: `normalize`
- **Type / Role:** Code — payload normalization.
- **Configuration choices (logic):**
  - Reads from `$json` using multiple fallbacks:
    - `source`: `input.source` OR `input.uat.source` OR `"webhook"`
    - `tester_name`: `input.tester_name` OR `input.tester.name` OR `input.uat.tester_name` OR `""`
    - `tester_email`: `input.tester_email` OR `input.tester.email` OR `input.uat.tester_email` OR `""`
    - `message_raw`: `input.message` OR `input.text` OR `input.feedback` OR `input.uat.message_raw` OR `""`
    - `build_version`: fallback to `"unknown"`
    - `page_url`, `screenshot_url`: default `""`
  - Adds `received_at` as ISO timestamp.
  - Returns **only** `{ uat: {...} }` (it does not preserve other top-level fields unless present in cfg and explicitly included before this node).
- **Inputs/Outputs:**
  - Input ← `data merge`
  - Output → `clean text`
- **Edge cases / failures:**
  - If feedback text is missing, `message_raw` becomes empty string; LLM output may become “Noise”.
  - If you need `cfg.*` downstream, ensure it survives: current return object overwrites the item and drops non-uat fields.

#### Node: `clean text`
- **Type / Role:** Code — sanitization and length limiting.
- **Configuration choices (logic):**
  - Takes `uat.message_raw`, then:
    - strips HTML tags: `/<[^>]*>/g`
    - collapses whitespace
    - trims
    - truncates to **3000 characters**
  - Outputs original structure plus `uat.message_clean`.
- **Inputs/Outputs:**
  - Input ← `normalize`
  - Output → `AI agent`
- **Edge cases / failures:**
  - Aggressive HTML stripping may remove meaningful content (e.g., angle-bracket snippets).
  - Truncation may cut off repro steps; consider increasing or summarizing first for long reports.

---

### Block 3 — AI Triage + JSON Parsing/Validation

**Overview:** Sends cleaned UAT feedback to an OpenAI model with a strict JSON schema request, then parses/validates the model output and normalizes values to allowed enums.

**Nodes Involved:** `AI agent`, `parsing and validation`

#### Node: `AI agent`
- **Type / Role:** OpenAI (LangChain) — generates structured triage JSON.
- **Configuration choices:**
  - Model: `gpt-5.2`
  - Prompt instructs: “return ONLY JSON” with schema:
    - `sentiment`: Positive|Negative
    - `type`: CriticalBug|UXImprovement|FeatureRequest|Noise
    - `severity`: Blocker|Critical|Major|Minor
    - `summary` max 160 chars
    - `components`: must choose from `[login, onboarding, checkout, search, profile, settings, performance, ui, api, other]`
    - `repro_steps` array
    - `suggested_title` max 80 chars
    - `confidence` numeric
  - Includes context fields (build/page/screenshot URLs).
- **Expressions used:**
  - `{{ $json.uat.build_version }}`, `{{ $json.uat.page_url }}`, `{{ $json.uat.screenshot_url }}`, `{{ $json.uat.message_clean }}`
- **Inputs/Outputs:**
  - Input ← `clean text`
  - Output → `parsing and validation`
- **Credentials:** OpenAI API credential required.
- **Edge cases / failures:**
  - Model returns non-JSON or JSON wrapped in Markdown/code fences (handled later but may still fail if not stripped).
  - Rate limits / quota errors.
  - Prompt asks for strict schema but model may violate max lengths or component enum.

#### Node: `parsing and validation`
- **Type / Role:** Code — robust parsing + enum enforcement.
- **Configuration choices (logic):**
  - Attempts to locate raw model output from multiple possible fields:
    - `$json.output`, `$json.text`, `$json.message`, `$json.response.text`, `$json.data[0].content`, `$json.response[0].message.content`
  - `JSON.parse(raw)`; if it fails:
    - `parse_ok = false`
    - sets fallback triage:
      - sentiment Negative, type Noise, severity Minor
      - summary indicates parsing failure
      - confidence 0
  - Validates/enforces enums:
    - `type` ∈ [CriticalBug, UXImprovement, FeatureRequest, Noise]
    - `severity` ∈ [Blocker, Critical, Major, Minor]
    - `sentiment` ∈ [Positive, Negative]
  - Clamps confidence to [0..1].
  - Outputs original `$json` plus `triage` with `parse_ok`.
- **Inputs/Outputs:**
  - Input ← `AI agent`
  - Output → `double check`
- **Edge cases / failures:**
  - If the OpenAI node output structure differs from expected, `raw` may be empty string; parse fails and you get fallback triage.
  - If model outputs JSON with trailing comments or code fences, parse will fail (consider stripping ```json fences).

---

### Block 4 — Notion Dedupe & Upsert

**Overview:** Searches Notion to detect duplicate entries, then updates an existing page or creates a new roadmap item.

**Nodes Involved:** `double check`, `if found`, `update notion database`, `create notion database`

#### Node: `double check`
- **Type / Role:** Notion — Search operation.
- **Configuration choices:**
  - Operation: `search`
  - Search text expression: `{{ $json.triage.suggested_title || $json.triage.summary }}`
  - Searches across Notion (workspace-wide search behavior), not strictly within a single database.
- **Inputs/Outputs:**
  - Input ← `parsing and validation`
  - Output → `if found`
- **Credentials:** Notion API credential required.
- **Edge cases / failures:**
  - Notion search is fuzzy and can return unrelated pages; can cause accidental “update existing” decisions.
  - Auth / permission issues if integration lacks access to target pages.
  - Rate limiting on Notion API.

#### Node: `if found`
- **Type / Role:** IF — branching on whether Notion search returned results.
- **Configuration choices:**
  - Condition: `(($json.results || $json.data || []).length) > 0`
- **Inputs/Outputs:**
  - Input ← `double check`
  - True output (index 0) → `update notion database`
  - False output (index 1) → `create notion database`
- **Edge cases / failures:**
  - If Notion node returns results in a different key than `results`/`data`, this condition may misroute.

#### Node: `update notion database`
- **Type / Role:** Notion — Update database page.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `update`
  - PageId is set via URL mode but currently hard-coded-like: `youridpage.com` (placeholder).
- **Inputs/Outputs:**
  - Input ← `if found` (true branch)
  - Output → `compose reply branch 2`
- **Credentials:** Notion API credential required.
- **Edge cases / failures:**
  - **Critical:** This node does not select the page ID from the search result. It uses a fixed placeholder, so “update” will fail or update the wrong page unless manually configured.
  - Missing property mappings: no page properties are being updated (only pageId + operation), so even if it succeeds it may do nothing meaningful.

#### Node: `create notion database`
- **Type / Role:** Notion — Create new page in a database.
- **Configuration choices:**
  - Resource: `databasePage`
  - Operation: `create`
  - DatabaseId: `2b311ca2-096c-8049-a5ab-de07d643edca`
  - Title: “Add Roadmap Idea”
  - Adds a content block with a template-like text:
    - “Title = suggested_title”, “Summary”, “Component(s)”, “Build version”, “Tester”, “Source”, Status=New, Impact
  - Note: This appears to create block content, not mapped Notion database properties (depends on node configuration).
- **Inputs/Outputs:**
  - Input ← `if found` (false branch)
  - Output → `compose reply branch 2`
- **Edge cases / failures:**
  - If the database requires properties (Title, Status select, etc.) and they are not set as properties, Notion may reject the create call.
  - Permission issues if integration not shared with that database.

---

### Block 5 — Closed Loop Notifications + Webhook Response

**Overview:** Builds a confirmation message (feature-request phrasing), chooses Slack DM vs Gmail based on `uat.source`, sends to tester, and returns a structured response payload to the original webhook caller.

**Nodes Involved:** `compose reply branch 2`, `how to contact`, `slack tester`, `tester email`, `Webhook response`

#### Node: `compose reply branch 2`
- **Type / Role:** Set — creates `reply.subject` and `reply.body`.
- **Configuration choices:**
  - `reply.subject`: “UAT feedback received — Feature request logged”
  - `reply.body`: templated message using:
    - `{{ $json.uat.tester_name }}`
    - `{{ $json.triage.summary }}`
- **Inputs/Outputs:**
  - Input ← `update notion database` OR `create notion database`
  - Output → `how to contact`
- **Edge cases / failures:**
  - This reply is always “Feature request logged” even if AI triage says CriticalBug/UXImprovement/Noise. No branching on `triage.type`.

#### Node: `how to contact`
- **Type / Role:** IF — routes notifications based on source.
- **Configuration choices:**
  - Condition: `{{ $json.uat.source }}` equals `"slack"`
- **Inputs/Outputs:**
  - Input ← `compose reply branch 2`
  - True → `slack tester`
  - False → `tester email`
- **Edge cases / failures:**
  - If source is `"Slack"` or `"slack_dm"` etc., it will fall back to email.
  - If tester email is missing, Gmail send will fail.

#### Node: `slack tester`
- **Type / Role:** Slack — send DM/message to a user.
- **Configuration choices:**
  - Authentication: OAuth2
  - Sends `text = {{ $json.reply.body }}`
  - Select: user (hardcoded user ID `U09UKKK9R25`, cached name “analyticsn8n”)
- **Inputs/Outputs:**
  - Input ← `how to contact` (true)
  - Output → `Webhook response`
- **Credentials:** Slack OAuth2 credential required.
- **Edge cases / failures:**
  - **Critical:** It does not DM the actual tester; it always messages the configured Slack user.
  - Slack API errors if bot cannot message user or missing scopes.

#### Node: `tester email`
- **Type / Role:** Gmail — send email to tester.
- **Configuration choices:**
  - To: `{{ $json.uat.tester_email }}`
  - Subject: `{{ $json.reply.subject }}`
  - Body: `{{ $json.reply.body }}`
- **Inputs/Outputs:**
  - Input ← `how to contact` (false)
  - Output → `Webhook response`
- **Credentials:** Gmail OAuth2 credential required.
- **Edge cases / failures:**
  - Missing/invalid email address.
  - Gmail sending limits, OAuth expiry, insufficient scopes.

#### Node: `Webhook response`
- **Type / Role:** Respond to Webhook — closes the HTTP request with a JSON status.
- **Configuration choices:**
  - Response code: **200**
  - “Respond with”: `allIncomingItems`
  - Response body configured via `responseKey` expression (intended JSON):
    - status received
    - type/severity/confidence from `$json.triage.*`
- **Inputs/Outputs:**
  - Input ← `slack tester` OR `tester email`
  - End node (responds to the requester)
- **Edge cases / failures:**
  - The `responseKey` is set to a JSON-like string with embedded `{{ }}`; depending on node behavior, it may return a string rather than structured JSON.
  - If notification fails and execution stops, webhook may time out unless error handling is configured.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| trigger | Webhook | Receive UAT feedback (POST entry point) | — | data merge | ## Ingestion & Normalization<br>Receives feedback via webhook and standardizes fields (tester, build, page, message) into a consistent uat.* structure, then cleans the message for AI processing. |
| data mapping | Set | Define `cfg.*` constants | — | data merge |  |
| data merge | Merge | Combine webhook payload + cfg values | trigger, data mapping | normalize | ## Ingestion & Normalization<br>Receives feedback via webhook and standardizes fields (tester, build, page, message) into a consistent uat.* structure, then cleans the message for AI processing. |
| normalize | Code | Normalize inbound payload into `uat.*` | data merge | clean text | ## Ingestion & Normalization<br>Receives feedback via webhook and standardizes fields (tester, build, page, message) into a consistent uat.* structure, then cleans the message for AI processing. |
| clean text | Code | Strip HTML/whitespace, truncate, create `message_clean` | normalize | AI agent | ## Ingestion & Normalization<br>Receives feedback via webhook and standardizes fields (tester, build, page, message) into a consistent uat.* structure, then cleans the message for AI processing. |
| AI agent | OpenAI (LangChain) | Classify + summarize into strict JSON | clean text | parsing and validation | ## AI Triage<br>Uses an AI model to classify feedback (type, severity, summary, title, confidence) and outputs structured JSON for automation. |
| parsing and validation | Code | Parse model output; enforce enums; fallback on parse errors | AI agent | double check | ## AI Triage<br>Uses an AI model to classify feedback (type, severity, summary, title, confidence) and outputs structured JSON for automation. |
| double check | Notion | Search Notion for duplicates by title/summary | parsing and validation | if found | ## Notion Dedupe & Upsert<br>Searches Notion by suggested title to avoid duplicates. If found → update the existing page. If not → create a new roadmap/backlog entry. |
| if found | IF | Branch on Notion search results length | double check | update notion database, create notion database | ## Notion Dedupe & Upsert<br>Searches Notion by suggested title to avoid duplicates. If found → update the existing page. If not → create a new roadmap/backlog entry. |
| update notion database | Notion | Update an existing Notion page | if found (true) | compose reply branch 2 | ## Notion Dedupe & Upsert<br>Searches Notion by suggested title to avoid duplicates. If found → update the existing page. If not → create a new roadmap/backlog entry. |
| create notion database | Notion | Create a new page in Notion database | if found (false) | compose reply branch 2 | ## Notion Dedupe & Upsert<br>Searches Notion by suggested title to avoid duplicates. If found → update the existing page. If not → create a new roadmap/backlog entry. |
| compose reply branch 2 | Set | Compose confirmation email/DM content | update notion database, create notion database | how to contact | ## Closed Loop<br>Sends a confirmation to the tester (Slack DM or email) and responds to the original webhook with status + triage metadata. |
| how to contact | IF | Route by `uat.source` (slack vs email) | compose reply branch 2 | slack tester, tester email | ## Closed Loop<br>Sends a confirmation to the tester (Slack DM or email) and responds to the original webhook with status + triage metadata. |
| slack tester | Slack | Send Slack message (DM/user message) | how to contact (true) | Webhook response | ## Closed Loop<br>Sends a confirmation to the tester (Slack DM or email) and responds to the original webhook with status + triage metadata. |
| tester email | Gmail | Send confirmation email to tester | how to contact (false) | Webhook response | ## Closed Loop<br>Sends a confirmation to the tester (Slack DM or email) and responds to the original webhook with status + triage metadata. |
| Webhook response | Respond to Webhook | Return status + triage metadata to caller | slack tester, tester email | — | ## Closed Loop<br>Sends a confirmation to the tester (Slack DM or email) and responds to the original webhook with status + triage metadata. |
| Sticky Note4 | Sticky Note | Documentation container | — | — | ## How it works<br>This workflow automates Product UAT feature request triage using AI and Notion.<br><br>When feedback is submitted via a webhook, the workflow normalizes and cleans the input into a consistent, AI-ready structure. An AI model analyzes the feedback to classify its type, generate a short summary and suggested title, and assign a confidence score.<br><br>For feature requests, the workflow searches an existing Notion database to prevent duplicates. If a matching entry exists, it is updated; otherwise, a new roadmap item is created.<br><br>Finally, the workflow notifies the tester via Slack or email and responds to the original webhook with a structured status payload, ensuring full traceability. |
| Sticky Note | Sticky Note | Documentation container | — | — | ## Ingestion & Normalization<br>Receives feedback via webhook and standardizes fields (tester, build, page, message) into a consistent uat.* structure, then cleans the message for AI processing. |
| Sticky Note1 | Sticky Note | Documentation container | — | — | ## AI Triage<br>Uses an AI model to classify feedback (type, severity, summary, title, confidence) and outputs structured JSON for automation. |
| Sticky Note2 | Sticky Note | Documentation container | — | — | ## Notion Dedupe & Upsert<br>Searches Notion by suggested title to avoid duplicates. If found → update the existing page. If not → create a new roadmap/backlog entry. |
| Sticky Note3 | Sticky Note | Documentation container | — | — | ## Closed Loop<br>Sends a confirmation to the tester (Slack DM or email) and responds to the original webhook with status + triage metadata. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: `Product UAT Feedback → AI Triage + Notion Upsert + Closed Loop` (or your preferred name)
   - (Optional) Add tags like “product management”, “Productivity”.

2) **Add Webhook trigger**
   - Node: **Webhook**
   - Method: `POST`
   - Path: generate or set a path (store it for your UAT form/tool)
   - Connect: `Webhook` → `Merge` (later)

3) **Add configuration Set node (optional but intended)**
   - Node: **Set** (name it `data mapping`)
   - Add fields:
     - `cfg.confidenceThreshold` (Number) = `0.6`
     - `cfg.dedupeEnabled` (Boolean) = `false`
     - (Optional) other cfg fields if you plan to use them
   - **Important wiring fix (recommended):**
     - Connect: `Webhook` → `data mapping` (so it runs)
     - Then feed both into Merge (next step).  
     - Alternative: keep it as a separate branch triggered by Webhook, then merge.

4) **Add Merge node**
   - Node: **Merge** (name: `data merge`)
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - Input 1: `Webhook` → `data merge` (index 0)
     - Input 2: `data mapping` → `data merge` (index 1)
   - Output: `data merge` → `normalize`

5) **Add Code node: normalize**
   - Node: **Code** (name: `normalize`)
   - Paste logic equivalent to:
     - Build `uat` object with:
       - `source`, `tester_name`, `tester_email`, `message_raw`, `build_version`, `page_url`, `screenshot_url`, `received_at`
     - Ensure defaults if missing.
   - Connect: `normalize` → `clean text`

6) **Add Code node: clean text**
   - Node: **Code** (name: `clean text`)
   - Implement:
     - Strip HTML tags, normalize whitespace, trim, limit to 3000 chars
     - Set `uat.message_clean`
   - Connect: `clean text` → `AI agent`

7) **Add OpenAI (LangChain) node**
   - Node: **OpenAI** (the `@n8n/n8n-nodes-langchain.openAi` chat-style node)
   - Credentials: configure **OpenAI API** key
   - Model: `gpt-5.2` (or available equivalent)
   - Prompt: ask to “return ONLY JSON” using the schema and rules (type/severity rules, components enum, etc.)
   - Connect: `AI agent` → `parsing and validation`

8) **Add Code node: parsing and validation**
   - Node: **Code** (name: `parsing and validation`)
   - Implement:
     - Extract raw model text from likely fields
     - `JSON.parse`
     - On parse error: fallback triage object + `parse_ok=false`
     - Validate enums and clamp confidence
   - Connect: `parsing and validation` → `double check`

9) **Add Notion Search node (dedupe)**
   - Node: **Notion**
   - Credentials: Notion integration token; ensure the integration is shared with relevant pages/databases
   - Operation: **Search**
   - Query text: `{{ $json.triage.suggested_title || $json.triage.summary }}`
   - Connect: `double check` → `if found`

10) **Add IF node: found?**
   - Node: **IF** (name: `if found`)
   - Condition: `(results.length > 0)` using expression `( $json.results || $json.data || [] ).length > 0`
   - True → `update notion database`
   - False → `create notion database`

11) **Add Notion Update node (upsert branch: found)**
   - Node: **Notion** (name: `update notion database`)
   - Resource: **Database Page**
   - Operation: **Update**
   - **Required fix to make it functional:**
     - Set `pageId` dynamically from the search result (e.g., first result’s id), rather than a placeholder URL.
   - Map desired properties (title, summary, severity, etc.) according to your database schema.

12) **Add Notion Create node (upsert branch: not found)**
   - Node: **Notion** (name: `create notion database`)
   - Resource: **Database Page**
   - Operation: **Create**
   - Database ID: select your target UAT/Roadmap database
   - Set properties required by the database (recommended) and optionally add child blocks.
   - Connect both Notion branches to the reply composer:
     - `update notion database` → `compose reply branch 2`
     - `create notion database` → `compose reply branch 2`

13) **Add Set node: compose reply**
   - Node: **Set** (name: `compose reply branch 2`)
   - Fields:
     - `reply.subject` = “UAT feedback received — Feature request logged”
     - `reply.body` = templated text referencing `uat.tester_name` and `triage.summary`
   - Connect: `compose reply branch 2` → `how to contact`

14) **Add IF node: how to contact**
   - Node: **IF** (name: `how to contact`)
   - Condition: `{{ $json.uat.source }}` equals `slack`
   - True → Slack
   - False → Gmail

15) **Add Slack node**
   - Node: **Slack**
   - Credentials: Slack OAuth2 (scopes for chat:write and DM capability)
   - Operation: send message to **user**
   - Text: `{{ $json.reply.body }}`
   - **Recommended adjustment:** select user dynamically from payload (e.g., `uat.tester_slack_id`) instead of a fixed user ID.
   - Connect: `slack tester` → `Webhook response`

16) **Add Gmail node**
   - Node: **Gmail**
   - Credentials: Gmail OAuth2 with send permissions
   - To: `{{ $json.uat.tester_email }}`
   - Subject/body from `reply.*`
   - Connect: `tester email` → `Webhook response`

17) **Add Respond to Webhook node**
   - Node: **Respond to Webhook**
   - Response code: 200
   - Response content: include status + triage fields (type/severity/confidence)
   - Ensure output is valid JSON (structured fields), not a JSON-like string.
   - Finalize connections from Slack/Gmail to this node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer (applies to the whole workflow). |
| Several `cfg.*` fields (Jira/Sheets/channels) are defined but not used downstream; `data mapping` is also not connected to the trigger, so the merge may not include config values. | Workflow maintainability / execution correctness note. |
| Notion “update” branch is not wired to update the found result (pageId is a placeholder). | Critical functional gap to address before production. |
| Slack notification uses a fixed user ID rather than the tester identity. | Closed-loop accuracy / personalization issue. |