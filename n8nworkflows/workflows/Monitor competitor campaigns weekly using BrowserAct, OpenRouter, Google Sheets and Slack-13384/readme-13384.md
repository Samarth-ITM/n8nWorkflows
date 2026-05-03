Monitor competitor campaigns weekly using BrowserAct, OpenRouter, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/monitor-competitor-campaigns-weekly-using-browseract--openrouter--google-sheets-and-slack-13384


# Monitor competitor campaigns weekly using BrowserAct, OpenRouter, Google Sheets and Slack

## 1. Workflow Overview

**Purpose:**  
This workflow performs **weekly competitor campaign monitoring** by scraping a list of competitor landing pages, comparing the latest scrape to the previously stored snapshot, storing the updated analysis back into Google Sheets, and finally generating an **executive Slack digest** of the most important changes.

**Target use cases:**
- Track competitor offer changes (pricing, bundles, copy hooks) over time
- Get weekly “what changed and why it matters” summaries in Slack
- Maintain a lightweight historical database in Google Sheets

### 1.1 Scheduling & Target Intake
Runs weekly, pulls the list of page URLs to monitor from Google Sheets, then iterates through each URL.

### 1.2 Per-URL Scrape + AI Comparison + Database Update (Loop)
For each URL, BrowserAct scrapes the page; an AI agent compares “current scrape” vs “previous stored data” and outputs structured JSON; the sheet row is updated with the new context + comparison results.

### 1.3 Aggregate Results + Executive Digest + Slack Delivery
After processing all rows, the workflow aggregates the analyses, prompts a second AI agent to create Slack-ready message blocks, splits them into individual Slack messages, and posts them to a configured Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Targets
**Overview:** Triggers weekly and loads the monitoring targets (URLs + stored historical fields) from Google Sheets to start the loop.  
**Nodes involved:** `Weekly Trigger`, `Extract the target URLs`, `Loop Over Items`

#### Node: Weekly Trigger
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point; starts the workflow weekly.
- **Configuration (interpreted):** Interval-based schedule: every **1 week**.
- **Connections:**  
  - Output → `Extract the target URLs`
- **Failure/edge cases:** Instance timezone differences; missed runs if n8n is down at trigger time (depends on n8n scheduling reliability/version).

#### Node: Extract the target URLs
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads the sheet that acts as both target list and storage.
- **Configuration (interpreted):**
  - Document: **“Competitor Campaign Monitoring”** (Google Spreadsheet)
  - Sheet/tab: **“Page URLs”** (`gid=0`)
  - Operation is not explicitly shown in JSON (typical default is **Read / Get Many**). Practically, it outputs one item per row, including at least:
    - `Page URL`
    - `Page Context` (previous run snapshot) if present
    - `Comparison Analysis` if present
    - `row_number` (used later for updates)
- **Connections:**  
  - Input ← `Weekly Trigger`  
  - Output → `Loop Over Items`
- **Failure/edge cases:** OAuth expiry; sheet permissions; missing expected columns (`Page URL`, `row_number`, `Page Context`) causing downstream expression errors.

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates through rows in controlled batches.
- **Configuration (interpreted):** Uses defaults (batch size default in n8n UI unless changed; JSON shows only `options:{}`).
- **Connections:**
  - Input ← `Extract the target URLs`
  - Output (loop body) → `Scrape the target pages`
  - Output (done/secondary path in this workflow’s wiring) → `Retrieve database items`
  - Input back-edge ← `Update Database` (this continues the loop by requesting the next batch)
- **Failure/edge cases:** If `Update Database` fails, the loop may stop early; if no items are returned from Sheets, downstream blocks won’t run meaningfully.

---

### Block 2 — Scrape + AI Comparative Analysis + Update Sheet (Per URL)
**Overview:** For each URL, scrape the current page via BrowserAct, compare against the previously stored snapshot, output structured JSON, and update the sheet row.  
**Nodes involved:** `Scrape the target pages`, `Analyze the pages`, `OpenRouter Chat Model`, `Structured Output Parser`, `Update Database`

#### Node: Scrape the target pages
- **Type / role:** BrowserAct (`n8n-nodes-browseract.browserAct`) — runs a BrowserAct cloud workflow to scrape/structure the page.
- **Configuration (interpreted):**
  - Runs BrowserAct workflow type: **WORKFLOW**
  - BrowserAct workflow/template ID: **`77068334856978491`**
  - Passes input parameter `Target_Page` mapped from the current sheet row:
    - `Target_Page = {{$json["Page URL"]}}`
  - Expected output: structured scrape data accessible as **`$json.output.string`** (as referenced downstream).
- **Connections:**
  - Input ← `Loop Over Items`
  - Output → `Analyze the pages`
- **Failure/edge cases:**
  - BrowserAct auth/API key invalid
  - BrowserAct workflow ID not found or not shared with the API key
  - Target page blocks scraping (CAPTCHA, bot protection), timeouts, or changed page structure causing missing `output.string`

#### Node: Analyze the pages
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — performs page context synthesis and change detection.
- **Configuration (interpreted):**
  - Prompt mode: “define” (custom system message + custom text input)
  - **Text input expression:**
    - `Current Data: {{ $json.output.string }},`
    - `Previous Data: {{ $('Loop Over Items').item.json["Page Context"] }}`
  - **System message:** Detailed competitor-intel instructions:
    - Summarize page context (campaign title, banner, item list)
    - Compare all fields; detect price/content/messaging changes
    - Output JSON only; if previous is empty → mark as new tracking started
    - Output must be in English
  - **Has Output Parser:** enabled (connected to structured parser node)
- **AI dependencies / connections:**
  - Uses `OpenRouter Chat Model` as language model (AI connection)
  - Uses `Structured Output Parser` for schema enforcement (AI output parser connection)
- **Connections:**
  - Input ← `Scrape the target pages`
  - Output → `Update Database`
- **Failure/edge cases:**
  - `$('Loop Over Items').item.json["Page Context"]` may be missing/blank; the prompt accounts for it, but expressions can still produce `undefined` strings
  - If `Current Data` is not valid JSON (e.g., BrowserAct returns text), the model may hallucinate structure; parser auto-fix may or may not recover
  - Model refusal or rate limits; token limits if scrape payload is large

#### Node: OpenRouter Chat Model
- **Type / role:** OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) — LLM provider for per-page analysis.
- **Configuration (interpreted):**
  - Model: `openai/gpt-5`
  - Uses OpenRouter API credential: “OpenRouter account”
- **Connections:**
  - AI language model → `Analyze the pages`
  - AI language model → `Structured Output Parser` (as wired in the JSON)
- **Failure/edge cases:** OpenRouter key invalid; model name not available; provider outages; cost/rate limits.

#### Node: Structured Output Parser
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — validates/coerces the agent output into a required JSON schema.
- **Configuration (interpreted):**
  - `autoFix: true` attempts to repair near-valid JSON
  - Schema example requires:
    - `page_context.summary`
    - `comparison_analysis.has_changes`, `change_severity`
    - `price_change` object (detected/direction/difference_amount)
    - `content_changes` array
    - `verdict` string
- **Connections:**
  - AI output parser → `Analyze the pages`
- **Failure/edge cases:** If the model output is too malformed, auto-fix fails; schema mismatch can stop execution.

#### Node: Update Database
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — persists the latest analysis back to the same row.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Document: “Competitor Campaign Monitoring”
  - Sheet: “Page URLs”
  - Matching column: `row_number` (acts as primary key)
  - Columns written:
    - `Page URL = {{ $('Loop Over Items').item.json["Page URL"] }}`
    - `row_number = {{ $('Loop Over Items').item.json.row_number }}`
    - `Page Context = {{ $json.output.page_context }}`
    - `Comparison Analysis = {{ $json.output.comparison_analysis }}`
- **Connections:**
  - Input ← `Analyze the pages`
  - Output → `Loop Over Items` (back-edge to continue batch loop)
- **Failure/edge cases:**
  - If `row_number` is missing or not stable, updates may fail or overwrite wrong rows
  - Writing objects (`page_context`, `comparison_analysis`) into Sheets: depending on n8n/Sheets behavior, they may be stringified or may error if Sheets expects strings; ensure the column types tolerate JSON text
  - Concurrent runs could race and overwrite row data

---

### Block 3 — Aggregate + Digest Generation + Slack Posting
**Overview:** After the per-URL loop finishes, the workflow reads/collects the results, asks an AI reporter to generate Slack-ready message objects, splits them, and posts to Slack.  
**Nodes involved:** `Retrieve database items`, `Aggregate`, `Analyze all the items and generate a report`, `OpenRouter Chat Model1`, `Structured Output Parser1`, `Split out Slack messages`, `Send a message`

#### Node: Retrieve database items
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads the sheet content for reporting.
- **Configuration (interpreted):**
  - Document: “Competitor Campaign Monitoring”
  - Sheet: “Page URLs”
  - `executeOnce: true` — in n8n, this typically means the node runs only once even if triggered multiple times in a looped context (important because it is connected from the batch loop).
- **Connections:**
  - Input ← `Loop Over Items` (the “done/secondary” output line)
  - Output → `Aggregate`
- **Failure/edge cases:** Same Sheets/OAuth issues; if it runs too early (before loop finished) you could read partial data—however the wiring suggests it’s intended to run after the loop completes.

#### Node: Aggregate
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — converts multiple sheet rows into one combined payload.
- **Configuration (interpreted):**
  - Mode: `aggregateAllItemData` — packs all incoming items into a single array under a field (commonly `data`).
- **Connections:**
  - Input ← `Retrieve database items`
  - Output → `Analyze all the items and generate a report`
- **Failure/edge cases:** Large datasets can create big payloads and increase LLM token usage.

#### Node: Analyze all the items and generate a report
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — creates a Slack digest.
- **Configuration (interpreted):**
  - Text input: `Input : {{ JSON.stringify($json.data, null, 2) }}`
  - System message: filters items where `has_changes` is false; prioritizes `High`; outputs a JSON array of `{ slack_text }` objects (header + per-change messages), each under 2000 chars.
  - Has output parser enabled.
- **AI dependencies / connections:**
  - Uses `OpenRouter Chat Model1`
  - Uses `Structured Output Parser1`
- **Connections:**
  - Input ← `Aggregate`
  - Output → `Split out Slack messages`
- **Failure/edge cases:** If any rows contain non-JSON strings in `Page Context` / `Comparison Analysis`, the summarizer may misinterpret; token limits if too many pages.

#### Node: OpenRouter Chat Model1
- **Type / role:** OpenRouter Chat Model — LLM provider for the digest.
- **Configuration:** Model `openai/gpt-5`, same OpenRouter credential.
- **Connections:**
  - AI language model → `Analyze all the items and generate a report`
  - AI language model → `Structured Output Parser1`
- **Failure/edge cases:** Same as other OpenRouter model node.

#### Node: Structured Output Parser1
- **Type / role:** Structured Output Parser — enforces digest schema.
- **Configuration (interpreted):**
  - `autoFix: true`
  - Schema example: an array of objects `[{ "slack_text": "..." }, ...]`
- **Connections:**
  - AI output parser → `Analyze all the items and generate a report`
- **Failure/edge cases:** If the model adds extra fields or outputs non-array JSON, parser may fail.

#### Node: Split out Slack messages
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — turns the array of messages into individual items for Slack sending.
- **Configuration (interpreted):**
  - Field to split: `output` (expects the agent result array to be in `$json.output`)
- **Connections:**
  - Input ← `Analyze all the items and generate a report`
  - Output → `Send a message`
- **Failure/edge cases:** If the agent output is not stored under `output` (or is a string), split fails.

#### Node: Send a message
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts each message to a Slack channel.
- **Configuration (interpreted):**
  - Operation: send message to channel
  - Channel ID: `C09KLV9DJSX` (cached name: `all-browseract-workflow-test`)
  - Message text: `{{ $json.slack_text }}`
- **Connections:**
  - Input ← `Split out Slack messages`
- **Failure/edge cases:** Slack token missing scopes; channel not accessible; Slack rate limits if many messages.

---

### Block 4 — Documentation / Annotations (Non-executing)
**Overview:** Sticky notes provide usage requirements and external links; they do not affect execution.  
**Nodes involved:** `Documentation`, `Step 1 Explanation`, `Step 2 Explanation`, `Step 4 Explanation`, `Sticky Note`

#### Node: Documentation (Sticky Note)
- **Type / role:** Sticky Note — contains requirements and links.
- **Key content:** Credentials needed (BrowserAct, OpenRouter, Google Sheets, Slack), BrowserAct template requirement, setup steps, and BrowserAct docs links.

#### Node: Step 1 Explanation (Sticky Note)
- **Type / role:** Sticky Note — explains scheduling + target retrieval.

#### Node: Step 2 Explanation (Sticky Note)
- **Type / role:** Sticky Note — explains scrape + compare + update.

#### Node: Step 4 Explanation (Sticky Note)
- **Type / role:** Sticky Note — explains aggregation + Slack digest.

#### Node: Sticky Note (YouTube embed)
- **Type / role:** Sticky Note — contains `@[youtube](roIRCG3DryU)`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly entry point | — | Extract the target URLs | ### ⏰ Step 1: Scheduling & Targets |
| Extract the target URLs | Google Sheets | Read target URLs + stored history | Weekly Trigger | Loop Over Items | ### ⏰ Step 1: Scheduling & Targets |
| Loop Over Items | Split In Batches | Iterate over sheet rows | Extract the target URLs; Update Database | Retrieve database items; Scrape the target pages | ### ⏰ Step 1: Scheduling & Targets |
| Scrape the target pages | BrowserAct | Scrape each competitor page | Loop Over Items | Analyze the pages | ### 🕵️ Step 2: Comparative Analysis |
| Analyze the pages | LangChain Agent | Compare current scrape vs previous snapshot; produce structured analysis | Scrape the target pages | Update Database | ### 🕵️ Step 2: Comparative Analysis |
| OpenRouter Chat Model | OpenRouter Chat Model | LLM for per-page analysis | — (AI wired) | Analyze the pages; Structured Output Parser (AI) | ### 🕵️ Step 2: Comparative Analysis |
| Structured Output Parser | Structured Output Parser | Enforce per-page JSON schema | — (AI wired) | Analyze the pages (AI) | ### 🕵️ Step 2: Comparative Analysis |
| Update Database | Google Sheets | Update row with new context + comparison | Analyze the pages | Loop Over Items | ### 🕵️ Step 2: Comparative Analysis |
| Retrieve database items | Google Sheets | Read all rows for reporting | Loop Over Items | Aggregate | ### 📢 Step 4: Executive Digest |
| Aggregate | Aggregate | Combine all rows into a single dataset | Retrieve database items | Analyze all the items and generate a report | ### 📢 Step 4: Executive Digest |
| Analyze all the items and generate a report | LangChain Agent | Create Slack digest JSON array | Aggregate | Split out Slack messages | ### 📢 Step 4: Executive Digest |
| OpenRouter Chat Model1 | OpenRouter Chat Model | LLM for digest generation | — (AI wired) | Analyze all the items and generate a report; Structured Output Parser1 (AI) | ### 📢 Step 4: Executive Digest |
| Structured Output Parser1 | Structured Output Parser | Enforce digest array schema | — (AI wired) | Analyze all the items and generate a report (AI) | ### 📢 Step 4: Executive Digest |
| Split out Slack messages | Split Out | Split digest array into individual items | Analyze all the items and generate a report | Send a message | ### 📢 Step 4: Executive Digest |
| Send a message | Slack | Post each digest message to Slack | Split out Slack messages | — | ### 📢 Step 4: Executive Digest |
| Documentation | Sticky Note | Requirements + links | — | — | ## ⚡ Competitor Campaign Monitoring  \n**Summary:** This automation performs weekly surveillance…  \n[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)  \n[How to Connect n8n to BrowserAct](https://docs.browseract.com)  \n[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Annotation | — | — | ### ⏰ Step 1: Scheduling & Targets |
| Step 2 Explanation | Sticky Note | Annotation | — | — | ### 🕵️ Step 2: Comparative Analysis |
| Step 4 Explanation | Sticky Note | Annotation | — | — | ### 📢 Step 4: Executive Digest |
| Sticky Note | Sticky Note | Video embed annotation | — | — | @[youtube](roIRCG3DryU) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (example): *Automate competitor campaign monitoring using BrowserAct & Openrouter*
- Ensure workflow setting **Execution Order** is set to `v1` (Settings → Execution).

2) **Add trigger**
- Add node: **Schedule Trigger**
- Configure: **Every 1 week**
- Name: `Weekly Trigger`

3) **Create/prepare Google Sheet**
- Spreadsheet name (example): *Competitor Campaign Monitoring*
- Tab name: *Page URLs*
- Required columns (minimum):
  - `Page URL` (string)
  - `Page Context` (string/JSON text) — will store latest `page_context`
  - `Comparison Analysis` (string/JSON text) — will store latest `comparison_analysis`
  - `row_number` (n8n uses this internally when reading rows; ensure your read node returns it)
- Add your competitor URLs under `Page URL`.

4) **Read targets from Google Sheets**
- Add node: **Google Sheets**
- Name: `Extract the target URLs`
- Credentials: **Google Sheets OAuth2** (connect your Google account)
- Document: select your spreadsheet (*Competitor Campaign Monitoring*)
- Sheet: select *Page URLs*
- Operation: configure to **Read / Get Many** rows (any mode that outputs one item per row including `row_number`).
- Connect: `Weekly Trigger` → `Extract the target URLs`

5) **Add looping**
- Add node: **Split In Batches**
- Name: `Loop Over Items`
- Keep default options (or set a batch size appropriate for rate limits, e.g., 5–20).
- Connect: `Extract the target URLs` → `Loop Over Items`

6) **Configure BrowserAct scrape**
- Add node: **BrowserAct**
- Name: `Scrape the target pages`
- Credentials: **BrowserAct API**
- Choose Type: **WORKFLOW**
- Set Workflow ID to your BrowserAct workflow/template ID (example from JSON: `77068334856978491`)
- In the BrowserAct node “workflow config” inputs, map:
  - `Target_Page` = `{{$json["Page URL"]}}`
- Connect: `Loop Over Items` → `Scrape the target pages`

7) **Add OpenRouter model (per-page)**
- Add node: **OpenRouter Chat Model**
- Name: `OpenRouter Chat Model`
- Credentials: OpenRouter API key
- Model: `openai/gpt-5` (or another supported chat model)

8) **Add Structured Output Parser (per-page)**
- Add node: **Structured Output Parser**
- Enable **Auto-fix**
- Provide a schema example requiring:
  - `page_context.summary`
  - `comparison_analysis.has_changes`, `change_severity`
  - `comparison_analysis.price_change.detected/direction/difference_amount`
  - `comparison_analysis.content_changes[]`
  - `comparison_analysis.verdict`

9) **Add AI Agent (per-page analysis)**
- Add node: **AI Agent (LangChain Agent)**
- Name: `Analyze the pages`
- Prompt mode: **Define**
- Text field:
  - `Current Data: {{ $json.output.string }},`
  - `Previous Data: {{ $('Loop Over Items').item.json["Page Context"] }}`
- System message: paste the competitor intelligence analyst instructions (as in the workflow), ensuring it demands **JSON only**.
- Attach AI connections:
  - Language Model: connect `OpenRouter Chat Model` to this agent
  - Output Parser: connect `Structured Output Parser` to this agent
- Connect main flow: `Scrape the target pages` → `Analyze the pages`

10) **Update row in Google Sheets**
- Add node: **Google Sheets**
- Name: `Update Database`
- Credentials: same Google account
- Document: *Competitor Campaign Monitoring*
- Sheet: *Page URLs*
- Operation: **Update**
- Matching column: **row_number**
- Map fields:
  - `row_number` = `{{ $('Loop Over Items').item.json.row_number }}`
  - `Page URL` = `{{ $('Loop Over Items').item.json["Page URL"] }}`
  - `Page Context` = `{{ $json.output.page_context }}`
  - `Comparison Analysis` = `{{ $json.output.comparison_analysis }}`
- Connect: `Analyze the pages` → `Update Database`
- Connect loop-back: `Update Database` → `Loop Over Items` (to continue batches)

11) **Read all rows for digest (after loop)**
- Add node: **Google Sheets**
- Name: `Retrieve database items`
- Same document/sheet as above
- Configure to read all relevant rows for the report.
- Enable **Execute Once** (so it doesn’t run repeatedly due to loop context).
- Connect: `Loop Over Items` → `Retrieve database items` (from the appropriate output so it runs after batching completes in your design)

12) **Aggregate**
- Add node: **Aggregate**
- Name: `Aggregate`
- Mode: **Aggregate All Item Data** (so output contains a single item with a `data` array)
- Connect: `Retrieve database items` → `Aggregate`

13) **Add OpenRouter model (digest)**
- Add node: **OpenRouter Chat Model**
- Name: `OpenRouter Chat Model1`
- Model: `openai/gpt-5`
- Same OpenRouter credentials (or separate)

14) **Add Structured Output Parser (digest)**
- Add node: **Structured Output Parser**
- Name: `Structured Output Parser1`
- Auto-fix: enabled
- Schema example: JSON array of objects with `slack_text` strings:
  - `[ { "slack_text": "..." }, ... ]`

15) **Add AI Agent (digest writer)**
- Add node: **AI Agent**
- Name: `Analyze all the items and generate a report`
- Text input: `Input : {{ JSON.stringify($json.data, null, 2) }}`
- System message: paste the digest instructions (filter has_changes=false, prioritize High, output array of Slack messages under 2000 chars).
- Connect AI:
  - Language model: `OpenRouter Chat Model1`
  - Output parser: `Structured Output Parser1`
- Connect: `Aggregate` → `Analyze all the items and generate a report`

16) **Split digest into individual Slack messages**
- Add node: **Split Out**
- Name: `Split out Slack messages`
- Field to split: `output`
- Connect: `Analyze all the items and generate a report` → `Split out Slack messages`

17) **Send to Slack**
- Add node: **Slack**
- Name: `Send a message`
- Credentials: Slack OAuth/token with permission to post to the target channel
- Select: **Channel**
- Channel: choose your channel (example ID: `C09KLV9DJSX`)
- Text: `{{ $json.slack_text }}`
- Connect: `Split out Slack messages` → `Send a message`

18) **(Optional) Add sticky notes**
- Add sticky notes for requirements/links and block explanations, including:
  - BrowserAct docs: https://docs.browseract.com
  - YouTube embed text: `@[youtube](roIRCG3DryU)`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This automation performs weekly surveillance on competitor landing pages, comparing current offers against historical data to detect strategy shifts (prices, bundles, copy) and reporting findings to Slack.” | Documentation sticky note |
| Credentials required: BrowserAct, OpenRouter (GPT-4/5), Google Sheets, Slack. Mandatory: BrowserAct API template “Competitor Campaign Monitoring (Huel)”. | Documentation sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| `@[youtube](roIRCG3DryU)` | Video reference sticky note |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.