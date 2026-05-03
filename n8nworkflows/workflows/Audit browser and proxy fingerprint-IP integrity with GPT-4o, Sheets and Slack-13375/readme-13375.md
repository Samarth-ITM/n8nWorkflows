Audit browser and proxy fingerprint/IP integrity with GPT-4o, Sheets and Slack

https://n8nworkflows.xyz/workflows/audit-browser-and-proxy-fingerprint-ip-integrity-with-gpt-4o--sheets-and-slack-13375


# Audit browser and proxy fingerprint/IP integrity with GPT-4o, Sheets and Slack

## 1. Workflow Overview

**Title:** Audit browser and proxy fingerprint/IP integrity with GPT-4o, Sheets and Slack  
**Workflow name (internal):** Audit browser fingerprint and IP integrity to Slack reports  
**Purpose:** Automatically test multiple “bot-detection / IP reputation / fingerprint” websites using BrowserAct, have GPT‑4o analyze the raw outputs for bot-detection signals and inconsistencies, store each per-site diagnostic in Google Sheets, then produce a consolidated “GO / NO‑GO” security verdict and post it to Slack.

### 1.1 Trigger & Data Reset
Starts manually, then clears the Google Sheet “database” to store fresh run results.

### 1.2 Target URL Definition & Iteration
Defines a list of test URLs, splits them into individual items, and loops over them in batches.

### 1.3 BrowserAct Collection (Two test tracks)
For each URL, executes BrowserAct workflows to fetch raw diagnostic outputs:
- A “guarded” site accessibility check (fixed heavy-protected site)
- The regular “detection site” checks (using the URL list)

### 1.4 AI Forensic Analysis (Per-site)
Each BrowserAct output is analyzed by a GPT‑4o security analyst agent, producing a **structured JSON** with a single key `text` (markdown report).

### 1.5 Persistence (Google Sheets)
Appends each per-site report (`Result`) into a Google Sheet.

### 1.6 Aggregation, Final AI Verdict, Slack Delivery
Reads all stored results, aggregates them, has GPT‑4o produce a final “GO / NO‑GO” summary, and posts it to a Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Sheet Reset
**Overview:** Manually starts the workflow and clears the reporting sheet while preserving the header row. Ensures each run starts with an empty dataset.

**Nodes involved:**
- Execute manually
- Clear Database

#### Node: Execute manually
- **Type / role:** `Manual Trigger` — entry point for on-demand execution.
- **Configuration:** No parameters.
- **Outputs:** Sends a single trigger item to **Clear Database**.
- **Edge cases:** None (manual operation).

#### Node: Clear Database
- **Type / role:** `Google Sheets` — clears existing rows to reset run state.
- **Configuration (interpreted):**
  - Operation: **Clear**
  - Document: “IP, Fingerprint Integrity and Bot Detection Check” (Spreadsheet ID `1HFo...`)
  - Sheet/tab: “DataBase” (`gid=0`)
  - **keepFirstRow: true** (header preserved)
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”).
- **Outputs:** Passes control to **Define Target URLs**.
- **Failure modes / edge cases:**
  - OAuth token expired / missing scopes → auth errors.
  - Sheet renamed / gid changed → “sheet not found”.
  - If header row is not present, later “append” still works but your data model may drift.

---

### Block 2 — Define Targets & Loop Control
**Overview:** Creates the list of target detection URLs, converts it into one item per URL, then iterates over them using a batching loop.

**Nodes involved:**
- Define Target URLs
- Split Out
- Loop Over URLs

#### Node: Define Target URLs
- **Type / role:** `Set` — defines configuration data for the run.
- **Configuration:**
  - Creates field `urls` as an **array** expression:
    - `https://www.browserscan.net/bot-detection`
    - `https://www.ipqualityscore.com/free-ip-lookup-proxy-vpn-test`
    - `https://www.ip-score.com/`
  - Note: node comment says “Define your 10 target sites here.” (currently 3).
- **Inputs:** From **Clear Database**.
- **Outputs:** To **Split Out**.
- **Edge cases:**
  - Invalid expression syntax breaks execution.
  - Empty list → downstream BrowserAct nodes receive no URLs; loop may effectively do nothing.

#### Node: Split Out
- **Type / role:** `Split Out` — converts the `urls` array into individual items.
- **Configuration:** Field to split: `urls`.
- **Inputs:** From **Define Target URLs**.
- **Outputs:** To **Loop Over URLs**.
- **Edge cases:** If `urls` is not an array, the node will error.

#### Node: Loop Over URLs
- **Type / role:** `Split In Batches` — controls iteration over items.
- **Configuration:**
  - `reset: false` (does not reset automatically; relies on normal loop behavior).
  - Batch size uses node default (not explicitly set in JSON).
- **Inputs:** From **Split Out** and also loop-back from **Update Database**.
- **Outputs (two branches):**
  1. To **Add guarded test step** (fixed “heavy guarded” test)
  2. To **Extract the agent checking sites** (per-URL detection site run)
- **Edge cases / failure modes:**
  - If downstream path errors, the loop can stop mid-run.
  - If loop-back connection is broken, it will only process the first batch.

---

### Block 3 — BrowserAct Runs (Guarded + Regular)
**Overview:** Uses BrowserAct to load pages and return raw diagnostic output (`$json.output.string`). One path always tests a heavy-guarded site; the other tests each URL from the list.

**Nodes involved:**
- Add guarded test step
- Check the site accessibility
- Extract the agent checking sites

#### Node: Add guarded test step
- **Type / role:** `Set` — injects a constant “heavy guarded” target.
- **Configuration:**
  - Sets `Heavy_Guarded_Site` to `https://www.footlocker.co.uk/`
  - Note text is identical to the earlier URL list note (likely copied).
- **Inputs:** From **Loop Over URLs** (branch 1).
- **Outputs:** To **Check the site accessibility**.
- **Edge cases:** This value is not actually mapped into the BrowserAct call here; the subsequent BrowserAct node uses `{{ $json.urls }}`, not `Heavy_Guarded_Site` (see below). That means this “guarded step” may not do what the name implies unless BrowserAct uses defaults.

#### Node: Check the site accessibility
- **Type / role:** `BrowserAct` — runs a BrowserAct workflow to open the site and capture results.
- **Configuration (interpreted):**
  - Mode: `WORKFLOW`
  - BrowserAct workflowId: `76240546993093673`
  - Workflow config input mapping:
    - `Ip_Bot_check_Link` = `{{ $json.urls }}`
  - `open_incognito_mode: false`
- **Credentials:** BrowserAct API (“BrowserAct account”).
- **Inputs:** From **Add guarded test step**.
- **Outputs:** To **Analyze the site results**.
- **Failure modes / edge cases:**
  - Wrong/disabled BrowserAct workflow ID → execution failure.
  - If `urls` is undefined on this branch (likely), BrowserAct may:
    - use a template default (as hinted by the schema description), or
    - fail due to missing required input, depending on BrowserAct template configuration.
  - Site may block, present CAPTCHA, or return incomplete output; analysis must handle it.

#### Node: Extract the agent checking sites
- **Type / role:** `BrowserAct` — runs the same BrowserAct workflow per target URL.
- **Configuration (interpreted):**
  - Mode: `WORKFLOW`
  - workflowId: `76240546993093673`
  - `Ip_Bot_check_Link` = `{{ $json.urls }}` (here it exists: each loop item is a single URL from Split Out)
  - `open_incognito_mode: false`
- **Credentials:** BrowserAct API (“BrowserAct account”).
- **Inputs:** From **Loop Over URLs** (branch 2).
- **Outputs:** To **Analyze the site results and generate a report**.
- **Failure modes / edge cases:**
  - Same as above (auth, workflowId, site blocks).
  - BrowserAct output schema changes (e.g., `output.string` missing) will break downstream LLM agent prompts.

**Sub-workflow reference:** Both BrowserAct nodes invoke BrowserAct workflow/template `76240546993093673` (external to n8n). It is expected to return something like:
- `output.string` (raw textual page/diagnostic content)

---

### Block 4 — AI Forensic Analysis (Per-site)
**Overview:** Two LLM “agent” nodes analyze BrowserAct outputs and return strictly structured JSON: `{ "text": "..." }`. Each agent relies on an OpenRouter GPT‑4o language model and a structured output parser.

**Nodes involved:**
- OpenRouter
- Structured Output Parser2
- Analyze the site results
- OpenRouter Chat Model
- Structured Output Parser
- Analyze the site results and generate a report

#### Node: OpenRouter
- **Type / role:** `LM Chat OpenRouter` — provides GPT‑4o model for an agent.
- **Configuration:** Model `openai/gpt-4o`.
- **Credentials:** OpenRouter API (“OpenRouter account”).
- **Connections:** Feeds the AI language model input of:
  - **Analyze the site results**
  - **Structured Output Parser2**
- **Failure modes:**
  - OpenRouter auth / quota / model availability issues.
  - Rate limiting when looping through many URLs.

#### Node: Structured Output Parser2
- **Type / role:** `Structured Output Parser` — enforces JSON schema and auto-fixes malformed LLM JSON.
- **Configuration:**
  - `autoFix: true`
  - Example schema: `{ "text": "🛡️ SECURITY DIAGNOSTIC REPORT ..."}`
- **Connections:** Provides `ai_outputParser` to **Analyze the site results**.
- **Failure modes:**
  - If LLM output is too malformed, autoFix may still fail.
  - If agent returns extra keys, parser may reject or attempt to coerce.

#### Node: Analyze the site results
- **Type / role:** `LangChain Agent` — validates whether the site actually loaded and checks for block/anti-bot indicators.
- **Key configuration:**
  - Prompt input: `Input : {{ $json.output.string }}`
  - System message: “Senior Scraping Integrity & Security Analyst” with step-by-step “load validation + forensic deep dive”
  - Output constraint: single JSON object with exactly one key `"text"` containing markdown string
  - `hasOutputParser: true` (uses Structured Output Parser2)
- **Inputs:** From **Check the site accessibility**.
- **Outputs:** To **Update Database1**.
- **Failure modes / edge cases:**
  - `$json.output.string` missing → expression resolves to empty; LLM analysis becomes low-signal.
  - Very large text can hit token limits or cause truncation.
  - If block pages are images/scripts, raw text may be minimal → false “success”/“failure”.

#### Node: OpenRouter Chat Model
- **Type / role:** `LM Chat OpenRouter` — GPT‑4o model for the second agent.
- **Configuration:** Model `openai/gpt-4o`.
- **Connections:** Feeds the AI language model input of:
  - **Analyze the site results and generate a report**
  - **Structured Output Parser**
- **Failure modes:** Same as other OpenRouter node.

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser` — enforces JSON `{ "text": "# 🛡️ SECURITY DIAGNOSTIC REPORT ..." }`.
- **Configuration:**
  - `autoFix: true`
  - Example schema includes a markdown header and report body
- **Connections:** Provides `ai_outputParser` to **Analyze the site results and generate a report**.
- **Failure modes:** Same as other structured parser.

#### Node: Analyze the site results and generate a report
- **Type / role:** `LangChain Agent` — deeper forensic “ruthless line-by-line” analysis and classification: Identified (Bot) / Suspicious / Clean (Human).
- **Key configuration:**
  - Prompt input: `Input : {{ $json.output.string }}`
  - System message emphasizes:
    - classify artifact (block page vs fingerprint leak vs success)
    - network forensics, UA/platform mismatch, automation leaks, CAPTCHA error codes
  - Output: strict JSON with only `"text"` markdown report
  - `hasOutputParser: true` (uses Structured Output Parser)
- **Inputs:** From **Extract the agent checking sites**.
- **Outputs:** To **Update Database** (append and continue loop).
- **Failure modes / edge cases:**
  - Same as above (missing output, oversized text).
  - Overconfident classification if site returns partial HTML.

---

### Block 5 — Storage in Google Sheets (Per-site) + Loop-back
**Overview:** Appends each AI-produced report into a single-column “Result” field in Google Sheets. One append path loops back to continue processing URLs.

**Nodes involved:**
- Update Database
- Update Database1

#### Node: Update Database
- **Type / role:** `Google Sheets` — appends results for the detection-site analysis track.
- **Configuration:**
  - Operation: **Append**
  - Document: “IP, Fingerprint Integrity and Bot Detection Check”
  - Sheet: “DataBase”
  - Column mapping: `Result` = `{{ $json.output.text }}`
  - Mapping mode: “define below”
- **Inputs:** From **Analyze the site results and generate a report**.
- **Outputs:** Back to **Loop Over URLs** (loop continuation).
- **Failure modes / edge cases:**
  - If the sheet doesn’t have a `Result` header, append may create misaligned columns.
  - Large text may exceed Google Sheets cell limits (~50k chars) → truncation/failure.
  - Auth / quota limits.

#### Node: Update Database1
- **Type / role:** `Google Sheets` — appends results for the “site accessibility” guarded track.
- **Configuration:** Same as Update Database (append `Result` from `{{ $json.output.text }}`).
- **Inputs:** From **Analyze the site results**.
- **Outputs:** To **Fetch stored data** (this is important: it starts the aggregation path).
- **Failure modes:** Same as Update Database.

**Design note:** This introduces a timing risk: **Fetch stored data** can run before the URL loop finishes (because Update Database1 is not wired into the loop-back). Depending on execution timing, the final report may be generated with incomplete data.

---

### Block 6 — Fetch, Aggregate, Final AI Verdict, Slack
**Overview:** Reads all rows from the sheet, aggregates them into a single dataset, asks GPT‑4o to produce a final GO/NO‑GO summary, then posts to Slack.

**Nodes involved:**
- Fetch stored data
- Aggregate
- OpenRouter Chat Model1
- Structured Output Parser1
- Process final data
- Send Report

#### Node: Fetch stored data
- **Type / role:** `Google Sheets` — reads stored results from the sheet.
- **Configuration:** Operation not explicitly set in JSON (defaults to a “read/get” operation in this node type/version). It targets:
  - Document: same spreadsheet
  - Sheet: “DataBase”
- **Inputs:** From **Update Database1**.
- **Outputs:** To **Aggregate**.
- **Failure modes / edge cases:**
  - If the node defaults to “read all rows”, large sheets can be slow/time out.
  - If it defaults to a different operation in your n8n version, results may be empty.

#### Node: Aggregate
- **Type / role:** `Aggregate` — combines multiple items into one structure for final analysis.
- **Configuration:** `aggregateAllItemData` (collect all item JSON into a combined object/array).
- **Inputs:** From **Fetch stored data**.
- **Outputs:** To **Process final data**.
- **Failure modes:** If incoming items are empty, final report will be based on empty set.

#### Node: OpenRouter Chat Model1
- **Type / role:** `LM Chat OpenRouter` — GPT‑4o model for the final summarization agent.
- **Configuration:** Model `openai/gpt-4o`.
- **Connections:** Feeds AI language model input of:
  - **Process final data**
  - **Structured Output Parser1**
- **Failure modes:** Rate limits/quota.

#### Node: Structured Output Parser1
- **Type / role:** `Structured Output Parser` — enforces output JSON `{ "text": "..." }`.
- **Configuration:**
  - `autoFix: true`
  - Example schema is a long “SECURITY DIAGNOSTIC REPORT…”
- **Connections:** Provides `ai_outputParser` to **Process final data**.

#### Node: Process final data
- **Type / role:** `LangChain Agent` — consolidates multiple site reports into final verdict.
- **Key configuration:**
  - Prompt input: `Input : {{ JSON.stringify($json.data, null, 2) }}`
  - System message: “Lead Automation Security Auditor”
  - Required sections in `text`:
    1) FINAL MISSION VERDICT (GO/NO‑GO)  
    2) SCORECARD  
    3) CRITICAL FAILURES  
    4) CONSISTENCY CHECK  
    5) RECOMMENDATION
  - Constraint: “Do not output markdown code blocks.”
  - `hasOutputParser: true`
- **Inputs:** From **Aggregate**.
- **Outputs:** To **Send Report**.
- **Failure modes / edge cases:**
  - If Aggregate outputs a different field than `data`, the stringify expression will be wrong.
  - Very large aggregated data can exceed model context.
  - Inconsistent per-row formatting (freeform markdown) may reduce summarization quality.

#### Node: Send Report
- **Type / role:** `Slack` — posts final summary to a Slack channel.
- **Configuration:**
  - Operation: send message (by providing `text`)
  - Channel: `all-browseract-workflow-test` (ID `C09KLV9DJSX`)
  - Text: `{{ $json.output.text }}`
- **Credentials:** Slack API (“Slack account 2”).
- **Failure modes / edge cases:**
  - Slack token scopes missing (`chat:write`) or channel access issues.
  - Message length limits; very long text may be truncated or rejected.

---

### Block 7 — Documentation / Notes (Sticky Notes)
**Overview:** Provides embedded operational guidance and links.

**Nodes involved:**
- Documentation (sticky note)
- Step 1 Explanation (sticky note)
- Step 2 Explanation (sticky note)
- Step 3 Explanation (sticky note)
- Sticky Note (YouTube embed)

No runtime impact, but contents are referenced in the Summary Table.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Execute manually | n8n-nodes-base.manualTrigger | Manual entry point | — | Clear Database | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| Clear Database | n8n-nodes-base.googleSheets | Clear storage sheet (keep headers) | Execute manually | Define Target URLs | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| Define Target URLs | n8n-nodes-base.set | Define array of detection URLs | Clear Database | Split Out | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| Split Out | n8n-nodes-base.splitOut | Turn URL array into items | Define Target URLs | Loop Over URLs | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| Loop Over URLs | n8n-nodes-base.splitInBatches | Batch loop controller | Split Out; Update Database | Add guarded test step; Extract the agent checking sites | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| Add guarded test step | n8n-nodes-base.set | Define heavy-guarded site URL | Loop Over URLs | Check the site accessibility | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| Check the site accessibility | n8n-nodes-browseract.browserAct | BrowserAct run (accessibility / guarded) | Add guarded test step | Analyze the site results | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| OpenRouter | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (GPT‑4o) for “Analyze the site results” | — (AI connection) | Analyze the site results; Structured Output Parser2 (AI) | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI “Security Analyst” parses the raw text… |
| Structured Output Parser2 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON `{text}` for guarded analysis | — (AI connection) | Analyze the site results (AI parser) | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI “Security Analyst” parses the raw text… |
| Analyze the site results | @n8n/n8n-nodes-langchain.agent | AI validates load + detects blocks | Check the site accessibility | Update Database1 | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI “Security Analyst” parses the raw text… |
| Update Database1 | n8n-nodes-base.googleSheets | Append guarded analysis to sheet | Analyze the site results | Fetch stored data | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| Fetch stored data | n8n-nodes-base.googleSheets | Read sheet rows for aggregation | Update Database1 | Aggregate | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| Aggregate | n8n-nodes-base.aggregate | Combine rows into a single dataset | Fetch stored data | Process final data | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| OpenRouter Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (GPT‑4o) for final summary | — (AI connection) | Process final data; Structured Output Parser1 (AI) | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON `{text}` for final summary | — (AI connection) | Process final data (AI parser) | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| Process final data | @n8n/n8n-nodes-langchain.agent | Final GO/NO‑GO consolidation | Aggregate | Send Report | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| Send Report | n8n-nodes-base.slack | Post final report to Slack | Process final data | — | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| Extract the agent checking sites | n8n-nodes-browseract.browserAct | BrowserAct run per detection URL | Loop Over URLs | Analyze the site results and generate a report | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list of target URLs… |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (GPT‑4o) for per-site forensic report | — (AI connection) | Analyze the site results and generate a report; Structured Output Parser (AI) | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI “Security Analyst” parses the raw text… |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON `{text}` for per-site forensic report | — (AI connection) | Analyze the site results and generate a report (AI parser) | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI “Security Analyst” parses the raw text… |
| Analyze the site results and generate a report | @n8n/n8n-nodes-langchain.agent | AI forensic report per detection site | Extract the agent checking sites | Update Database | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI “Security Analyst” parses the raw text… |
| Update Database | n8n-nodes-base.googleSheets | Append per-site report; loop-back continuation | Analyze the site results and generate a report | Loop Over URLs | ### 📊 Step 3: Aggregation & Reporting — All individual site reports are stored… final AI agent reviews… |
| OpenRouter Chat Model (unused) | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider node present but not connected | — | — |  |
| Structured Output Parser (unused) | @n8n/n8n-nodes-langchain.outputParserStructured | Parser node present but not connected | — | — |  |
| Documentation | n8n-nodes-base.stickyNote | Embedded setup notes & links | — | — | ## ⚡ Workflow Overview & Setup — Summary/Requirements/Links (https://docs.browseract.com) |
| Step 1 Explanation | n8n-nodes-base.stickyNote | Embedded explanation of step 1 | — | — | ### 🕵️ Step 1: Multi-Site Testing — The workflow iterates through a list… |
| Step 2 Explanation | n8n-nodes-base.stickyNote | Embedded explanation of step 2 | — | — | ### 🛡️ Step 2: Forensic Analysis — For each site visited, an AI… |
| Step 3 Explanation | n8n-nodes-base.stickyNote | Embedded explanation of step 3 | — | — | ### 📊 Step 3: Aggregation & Reporting — All individual site reports… |
| Sticky Note | n8n-nodes-base.stickyNote | Embedded video link | — | — | @[youtube](64cKXeY52NQ) |

**Note on “unused” rows:** The workflow JSON includes **OpenRouter Chat Model** and **Structured Output Parser** nodes that are connected, and also separate nodes named **OpenRouter** / **Structured Output Parser2**, plus a pair **OpenRouter Chat Model1** / **Structured Output Parser1**. There is no extra disconnected pair beyond these; however, the naming is confusing. If in your n8n canvas you see duplicates, verify which ones are actually wired into the agents via AI connections.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: “Audit browser fingerprint and IP integrity to Slack reports” (or your preferred name).
- Ensure **Settings → Execution Order** is `v1` (as in the JSON).

2) **Add Trigger**
- Add node: **Manual Trigger**  
  - Name: “Execute manually”

3) **Add Google Sheets: Clear**
- Add node: **Google Sheets**
  - Name: “Clear Database”
  - Credentials: connect a **Google Sheets OAuth2** credential with access to the spreadsheet
  - Operation: **Clear**
  - Document: select your spreadsheet (create one if needed)
  - Sheet: select tab “DataBase”
  - Enable: **Keep First Row** = true
- Connect: **Execute manually → Clear Database**

4) **Add Set: URL list**
- Add node: **Set**
  - Name: “Define Target URLs”
  - Add field `urls` (Type: Array) with value like:
    - `["https://www.browserscan.net/bot-detection", "https://www.ipqualityscore.com/free-ip-lookup-proxy-vpn-test", "https://www.ip-score.com/"]`
- Connect: **Clear Database → Define Target URLs**

5) **Split URL array into items**
- Add node: **Split Out**
  - Name: “Split Out”
  - Field to split out: `urls`
- Connect: **Define Target URLs → Split Out**

6) **Add batching loop**
- Add node: **Split In Batches**
  - Name: “Loop Over URLs”
  - Configure batch size as desired (default is fine for small lists)
  - Options: `Reset` = false
- Connect: **Split Out → Loop Over URLs**

7) **BrowserAct credentials & external workflow**
- In n8n, create a **BrowserAct API** credential.
- In BrowserAct, ensure you have a BrowserAct workflow/template available (the JSON expects):
  - Template/workflow name: “IP, Fingerprint Integrity and Bot Detection Check”
  - BrowserAct workflowId: `76240546993093673` (replace with yours)
  - It should accept an input variable like `Ip_Bot_check_Link` and return an output containing raw results (commonly `output.string`).

8) **Branch A (regular per-URL testing): BrowserAct**
- Add node: **BrowserAct**
  - Name: “Extract the agent checking sites”
  - Type: WORKFLOW
  - workflowId: your BrowserAct workflow id
  - Map input: `Ip_Bot_check_Link` = `{{ $json.urls }}`
  - Incognito: false (optional)
- Connect: **Loop Over URLs (output 2) → Extract the agent checking sites**

9) **Branch A: LLM model + structured parser + agent**
- Add node: **OpenRouter Chat Model**
  - Name: “OpenRouter Chat Model”
  - Credentials: OpenRouter API key
  - Model: `openai/gpt-4o`
- Add node: **Structured Output Parser**
  - Name: “Structured Output Parser”
  - Auto-fix: true
  - Schema example: `{ "text": "# 🛡️ SECURITY DIAGNOSTIC REPORT\n\n..." }`
- Add node: **AI Agent (LangChain Agent)**
  - Name: “Analyze the site results and generate a report”
  - Prompt/input text: `Input : {{ $json.output.string }}`
  - System message: use the “Elite Security & Bot Detection Analyst” system message from the workflow (ensure it demands a single JSON with only `text`)
  - Enable: “Has Output Parser”
- Wire AI connections:
  - **OpenRouter Chat Model → (AI Language Model) → Agent**
  - **Structured Output Parser → (AI Output Parser) → Agent**
- Connect main flow:
  - **Extract the agent checking sites → Agent**

10) **Branch A: Append to Google Sheets + loop-back**
- Add node: **Google Sheets**
  - Name: “Update Database”
  - Operation: **Append**
  - Document: same spreadsheet
  - Sheet: “DataBase”
  - Map column: `Result` = `{{ $json.output.text }}`
  - Ensure the sheet has a header column named `Result`
- Connect: **Agent → Update Database**
- Connect loop-back: **Update Database → Loop Over URLs** (so the next URL is processed)

11) **Branch B (guarded accessibility check): Set + BrowserAct**
- Add node: **Set**
  - Name: “Add guarded test step”
  - Field `Heavy_Guarded_Site` = `https://www.footlocker.co.uk/`
- Add node: **BrowserAct**
  - Name: “Check the site accessibility”
  - Type: WORKFLOW
  - workflowId: same BrowserAct workflow id
  - Important: decide what input to pass:
    - If you want the heavy site, map `Ip_Bot_check_Link` = `{{ $json.Heavy_Guarded_Site }}`
    - (The provided JSON maps `{{ $json.urls }}` which likely won’t exist on this branch.)
- Connect: **Loop Over URLs (output 1) → Add guarded test step → Check the site accessibility**

12) **Branch B: LLM model + structured parser + agent**
- Add node: **OpenRouter Chat Model**
  - Name: “OpenRouter”
  - Model: `openai/gpt-4o`
- Add node: **Structured Output Parser**
  - Name: “Structured Output Parser2”
  - Auto-fix: true
  - Schema example: `{ "text": "..." }`
- Add node: **AI Agent**
  - Name: “Analyze the site results”
  - Prompt: `Input : {{ $json.output.string }}`
  - System message: “Senior Scraping Integrity & Security Analyst…”
  - Has Output Parser: enabled
- Wire AI connections:
  - **OpenRouter → (AI Language Model) → Analyze the site results**
  - **Structured Output Parser2 → (AI Output Parser) → Analyze the site results**
- Connect main flow:
  - **Check the site accessibility → Analyze the site results**

13) **Branch B: Append to Google Sheets**
- Add node: **Google Sheets**
  - Name: “Update Database1”
  - Operation: Append
  - Map: `Result` = `{{ $json.output.text }}`
- Connect: **Analyze the site results → Update Database1**

14) **Fetch all stored results**
- Add node: **Google Sheets**
  - Name: “Fetch stored data”
  - Operation: “Read/Get All” (depending on node UI; configure to read all rows from “DataBase”)
- Connect: **Update Database1 → Fetch stored data**

15) **Aggregate items**
- Add node: **Aggregate**
  - Name: “Aggregate”
  - Mode: **Aggregate All Item Data** (so you get one combined object)
- Connect: **Fetch stored data → Aggregate**

16) **Final AI consolidation**
- Add node: **OpenRouter Chat Model**
  - Name: “OpenRouter Chat Model1”
  - Model: `openai/gpt-4o`
- Add node: **Structured Output Parser**
  - Name: “Structured Output Parser1”
  - Auto-fix: true
  - Schema example: `{ "text": "..." }`
- Add node: **AI Agent**
  - Name: “Process final data”
  - Text: `Input : {{ JSON.stringify($json.data, null, 2) }}`
  - System message: “Lead Automation Security Auditor…” (requires GO/NO‑GO sections; no markdown code blocks)
  - Has Output Parser: enabled
- Wire AI connections:
  - **OpenRouter Chat Model1 → (AI Language Model) → Process final data**
  - **Structured Output Parser1 → (AI Output Parser) → Process final data**
- Connect main flow:
  - **Aggregate → Process final data**

17) **Slack posting**
- Add node: **Slack**
  - Name: “Send Report”
  - Credentials: Slack OAuth/token with `chat:write`
  - Channel: choose your channel (e.g., `all-browseract-workflow-test`)
  - Message text: `{{ $json.output.text }}`
- Connect: **Process final data → Send Report**

18) **(Optional but recommended) Fix the run-completion logic**
- As built in the provided JSON, the “Fetch stored data” path may start before the per-URL loop finishes.
- To make the final Slack report reliable, add a “When loop done” mechanism (e.g., use Split in Batches completion branch, or collect items then fetch once after loop completes).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This automation continuously monitors the integrity of your IP address and browser fingerprint… Requirements: BrowserAct, OpenRouter, Google Sheets, Slack. Mandatory: BrowserAct API (Template: IP, Fingerprint Integrity and Bot Detection Check)” | Sticky note: “Documentation” |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | @[youtube](64cKXeY52NQ) |

Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.