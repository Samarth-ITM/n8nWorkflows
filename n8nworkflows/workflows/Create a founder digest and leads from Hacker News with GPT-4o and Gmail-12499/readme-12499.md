Create a founder digest and leads from Hacker News with GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/create-a-founder-digest-and-leads-from-hacker-news-with-gpt-4o-and-gmail-12499


# Create a founder digest and leads from Hacker News with GPT-4o and Gmail

## 1. Workflow Overview

**Purpose:** This workflow runs daily to discover high-signal Hacker News threads (via Google SERP/SerpAPI), structure and filter them, then uses Azure OpenAI (GPT-4o-mini) to produce two outputs in parallel:
1) **A daily “Founder Digest”** emailed via Gmail (trend + why it matters + actions).  
2) **Founder/buying-intent insights (“leads/signals”)** extracted with an AI agent (optionally enriched via an MCP social tool) and appended to Google Sheets.

**Target use cases:**
- Founder/operator daily intelligence on HN (Show HN, Launch HN, AI, SaaS, startup signals).
- Lightweight lead discovery and “market signal” logging for follow-up.

### 1.1 Trigger & Discovery (SERP)
Daily schedule → query Google results for HN discussion pages.

### 1.2 Parsing & Filtering (Discussion Records)
Convert SERP response into normalized discussion items → filter out non-thread/noise pages.

### 1.3 Aggregation (Single dataset for AI)
Aggregate all valid threads into a single object for downstream AI prompts.

### 1.4 Output A: Founder Digest Email (AI + Gmail)
AI agent generates a formatted digest → send email.

### 1.5 Output B: Lead/Signal Extraction + Storage (AI + MCP + Sheets)
AI agent extracts high-signal insights (with MCP tool available) → normalize text → append to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Trigger & Search
**Overview:** Runs on a schedule and fetches relevant Hacker News discussions using SerpAPI (Google SERP).  
**Nodes involved:**  
- Trigger Daily Hacker News Scan  
- Search Hacker News Discussions via Google SERP

#### Node: Trigger Daily Hacker News Scan
- **Type / Role:** Schedule Trigger — time-based entrypoint.
- **Configuration (interpreted):** Runs on an interval schedule (the JSON shows an interval rule placeholder; in the UI this should be set to “every day” at a chosen time).
- **Connections:** Outputs to **Search Hacker News Discussions via Google SERP**.
- **Failure/edge cases:**
  - Misconfigured schedule (e.g., interval not set as intended) → workflow not firing.
  - Instance timezone differences can shift expected delivery time.

#### Node: Search Hacker News Discussions via Google SERP
- **Type / Role:** SerpAPI node — executes a Google query.
- **Configuration (interpreted):**
  - Query: `site:news.ycombinator.com ("Show HN" OR "Launch HN" OR "AI" OR "startup" OR "SaaS")`
  - Returns **5** results (`num: 5`).
- **Input:** Trigger.
- **Output:** Raw SerpAPI response to **Extract SERP Results into Discussion Records**.
- **Credentials:** Requires SerpAPI API key/credentials in n8n.
- **Failure/edge cases:**
  - SerpAPI auth errors / quota exhaustion.
  - Google SERP variability: results can include non-thread pages, duplicates, or irrelevant pages.
  - Small `num=5` can miss important threads on busy days.

**Sticky note (applies to nodes in this block):**  
“## 📡 Trigger & Search … Scheduled daily scan … Searches for Show HN, Launch HN, AI, startup, and SaaS …”

---

### Block 1.2 — Processing: Extract and Filter Discussion Records
**Overview:** Transforms the SERP response into clean per-thread items, then filters out known noise titles and ensures links are actual discussion threads.  
**Nodes involved:**  
- Extract SERP Results into Discussion Records  
- Filter Valid Hacker News Discussion Threads

#### Node: Extract SERP Results into Discussion Records
- **Type / Role:** Code node — maps SerpAPI’s `organic_results` into a simplified schema.
- **Configuration (interpreted):**
  - Reads first input item: `const data = $input.first().json;`
  - Uses `data.organic_results || []`
  - Outputs items with fields: `title`, `summary`(snippet), `link`, `date`, `source`
- **Input:** SerpAPI response.
- **Output:** List of structured items to **Filter Valid Hacker News Discussion Threads**.
- **Failure/edge cases:**
  - SerpAPI response shape changes (missing `organic_results`) → outputs empty list.
  - Missing fields: node defaults to empty strings and “Recent”.
  - If SerpAPI returns fewer than 5 results, downstream aggregation may be sparse.

#### Node: Filter Valid Hacker News Discussion Threads
- **Type / Role:** Code node — removes irrelevant/noise pages and keeps only real HN threads.
- **Configuration (interpreted):**
  - Blocks specific titles:
    - “Show | Hacker News”
    - “Show HN Guidelines”
    - “Launch HNs”
    - “Launch HN Instructions”
  - Keeps only links containing: `news.ycombinator.com/item`
  - Re-outputs normalized fields (title/summary/link/date/source)
- **Input:** Structured SERP items.
- **Output:** Filtered discussion items to **Aggregate Hacker News Discussions for Analysis**.
- **Failure/edge cases:**
  - Legitimate threads might be excluded if titles match blocked titles unexpectedly.
  - Threads could be missed if URL format differs (mobile URLs, tracking parameters, alternate HN domains).
  - If everything is filtered out → downstream AI gets an empty dataset.

**Sticky note (applies to nodes in this block):**  
“## 🔍 Data Processing … Extracts … filters out noise pages … keeping only real discussion threads …”

---

### Block 1.3 — Aggregation (Prepare dataset for AI)
**Overview:** Combines multiple filtered thread items into one aggregated dataset so AI nodes can reason over the full set for the day.  
**Nodes involved:**  
- Aggregate Hacker News Discussions for Analysis

#### Node: Aggregate Hacker News Discussions for Analysis
- **Type / Role:** Aggregate node — aggregates all incoming item JSON into one item.
- **Configuration (interpreted):**
  - Mode: “Aggregate All Item Data” (collects all items into a single array, typically under `data` in output).
- **Input:** Filtered thread items.
- **Output connections:** Splits to two parallel branches:
  - **Generate Founder Digest from Hacker News Discussions (AI)**
  - **Extract High-Signal Founder & Buying-Intent Insights (AI)**
- **Failure/edge cases:**
  - Empty input → aggregated `data` may be `[]`; AI prompts will still run unless you add an IF guard.
  - Large inputs (if `num` increased significantly) can enlarge prompt size and hit model context limits.

**Sticky note (applies to this block):**  
“## 📦 Discussion Aggregation … Combines all valid … into a single dataset …”

---

### Block 1.4 — Founder Digest Generation & Email Delivery
**Overview:** Uses an AI agent (GPT-4o-mini on Azure OpenAI) to generate a strict-format digest and emails it via Gmail.  
**Nodes involved:**  
- LLM Reasoning Engine for Founder Digest  
- Generate Founder Digest from Hacker News Discussions (AI)  
- Email Founder Daily Hacker News Digest

#### Node: LLM Reasoning Engine for Founder Digest
- **Type / Role:** Azure OpenAI Chat Model (LangChain) — provides the language model for the agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
- **Connections:** Connected to the agent via `ai_languageModel`.
- **Credentials:** Azure OpenAI credentials required (endpoint, API key, deployment/model mapping depending on n8n setup).
- **Failure/edge cases:**
  - Azure auth/endpoint misconfiguration.
  - Model/deployment name mismatch.
  - Rate limits/timeouts during scheduled runs.

#### Node: Generate Founder Digest from Hacker News Discussions (AI)
- **Type / Role:** LangChain Agent — generates the digest text using provided system + user prompt.
- **Configuration (interpreted):**
  - Prompt injects aggregated data: `{{ JSON.stringify($json.data, null, 2) }}`
  - Enforces exact format:
    - “🔥 Hacker News Daily Digest”
    - “📌 Key Trend”
    - “🧠 Why It Matters”
    - “🚀 Founder Actions”
  - System message: “startup and technology trend analyst… avoid fluff… concise and actionable”
- **Inputs:**
  - Main input: aggregated dataset (expects `$json.data`).
  - AI language model input: from **LLM Reasoning Engine for Founder Digest**.
- **Output:** Typically agent output is accessible as `{{$json.output}}` for the next node (as used by Gmail).
- **Failure/edge cases:**
  - If `$json.data` is undefined (aggregation changed or empty) → prompt may degrade or error.
  - Output may deviate from “format EXACTLY” in edge cases; consider adding post-validation.

#### Node: Email Founder Daily Hacker News Digest
- **Type / Role:** Gmail node — sends the digest email.
- **Configuration (interpreted):**
  - To: `user@example.com` (placeholder; must be replaced)
  - Subject: `🔥 Hacker News Daily Digest`
  - Body: `={{ $json.output }}`
- **Input:** AI agent output.
- **Credentials:** Gmail OAuth2.
- **Failure/edge cases:**
  - OAuth2 token expiry/refresh issues.
  - Gmail sending limits, “from” restrictions, or blocked content rules.
  - If AI output is empty → sends blank email unless guarded.

**Sticky note (applies to nodes in this block):**  
“## 📧 Founder Digest … daily email digest … Uses AI to synthesize discussions …”

---

### Block 1.5 — Lead Discovery / Signal Extraction + Enrichment + Storage
**Overview:** Uses an AI agent to extract buying-intent/founder signals from the aggregated HN dataset. The agent has an MCP tool available for external enrichment. Output is normalized and appended to Google Sheets.  
**Nodes involved:**  
- LLM Engine for Lead Qualification & Signal Interpretation  
- External Social Context & Enrichment Tool (MCP Client)  
- Extract High-Signal Founder & Buying-Intent Insights (AI)  
- Normalize AI Insight Output for Storage  
- Append Founder Insights to Hacker News Intelligence Sheet

#### Node: LLM Engine for Lead Qualification & Signal Interpretation
- **Type / Role:** Azure OpenAI Chat Model (LangChain) — provides the model for the lead/signal agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
- **Connections:** Feeds the agent via `ai_languageModel`.
- **Credentials:** Azure OpenAI.
- **Failure/edge cases:** Same as other Azure model node (auth, rate limits, deployment mismatch).

#### Node: External Social Context & Enrichment Tool (MCP Client)
- **Type / Role:** MCP Client Tool (LangChain tool) — optional enrichment tool the agent can call.
- **Configuration (interpreted):**
  - Endpoint: `https://mcp.xpoz.ai/mcp`
  - Auth: Bearer token auth (`bearerAuth`)
- **Connections:** Exposed to the agent via `ai_tool`.
- **Credentials:** Requires MCP bearer token/credential in n8n.
- **Failure/edge cases:**
  - Endpoint downtime / TLS issues.
  - Auth token invalid/expired.
  - Tool results may be non-deterministic; agent may overuse tool calls unless constrained.

#### Node: Extract High-Signal Founder & Buying-Intent Insights (AI)
- **Type / Role:** LangChain Agent — extracts structured “founder intelligence” from the aggregated dataset.
- **Configuration (interpreted):**
  - Prompt instructs:
    1) Key Signal  
    2) Problem Statement  
    3) Why This Matters  
    4) Actionable Insight  
  - Constraints: “Use ONLY the provided data. Do NOT invent facts.”
  - System message emphasizes high-signal insights and ignoring generic content.
  - Max iterations: 30 (agent may loop/tool-call up to this limit).
  - Injects `{{ JSON.stringify($json.data, null, 2) }}`
- **Inputs:**
  - Main: aggregated dataset from Aggregate node.
  - AI language model: **LLM Engine for Lead Qualification & Signal Interpretation**.
  - Tool: **External Social Context & Enrichment Tool (MCP Client)**.
- **Output:** Passed to normalization node; expects `item.json.output` to exist.
- **Failure/edge cases:**
  - Empty `$json.data` → weak/empty insights.
  - “Do not invent facts” can conflict with enrichment tool usage unless the agent clearly attributes tool output vs. HN data.
  - Long tool/agent loops can hit execution time limits.

#### Node: Normalize AI Insight Output for Storage
- **Type / Role:** Code node — cleans AI output formatting for easier Sheet storage.
- **Configuration (interpreted):**
  - Reads `item.json.output`
  - Removes numbered bold headings via regex: `/\d+\.\s*\*\*.*?\*\*\n?/g`
  - Removes bullet markers `- `
  - Collapses extra line breaks, trims whitespace
  - Outputs `{ relaxed_output: text }`
- **Input:** AI agent output.
- **Output:** Clean text into **Append Founder Insights to Hacker News Intelligence Sheet**.
- **Failure/edge cases:**
  - If the agent output format changes, regex may remove unintended parts or not remove headings.
  - If `output` is missing → stores empty string.

#### Node: Append Founder Insights to Hacker News Intelligence Sheet
- **Type / Role:** Google Sheets node — appends a row with the normalized insight.
- **Configuration (interpreted):**
  - Operation: Append
  - Document: Spreadsheet ID `YourSpreadsheetId` (placeholder)
  - Sheet/tab name: `hacker_news`
  - Column mapping: writes `output = {{$json.relaxed_output}}`
  - Matching columns includes `output` (not typically needed for append; but configured)
- **Input:** Normalized `{ relaxed_output }`.
- **Credentials:** Google Sheets OAuth2.
- **Failure/edge cases:**
  - Spreadsheet/tab not found or permissions missing.
  - Column schema mismatch (if the sheet doesn’t have an `output` column).
  - Google API quota/rate limits.

**Sticky note (applies to nodes in this block):**  
“## 🎯 Lead Discovery … identify buying-intent signals … enrich … log to Google Sheets …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | Documentation / canvas overview | — | — | “## 🚀 Automate Founder Digest & Lead Discovery… Setup steps… Connect SerpAPI… Azure OpenAI… Gmail… Google Sheets… MCP…” |
| Section: Trigger | Sticky Note | Documentation for trigger/search area | — | — | “## 📡 Trigger & Search… Scheduled daily scan… SERP API…” |
| Section: Processing | Sticky Note | Documentation for parsing/filtering | — | — | “## 🔍 Data Processing… filters out noise pages…” |
| Section: Email Digest | Sticky Note | Documentation for digest branch | — | — | “## 📧 Founder Digest… daily email digest…” |
| Section: Lead Discovery | Sticky Note | Documentation for lead branch | — | — | “## 🎯 Lead Discovery… enrich… log to Google Sheets…” |
| Security Note | Sticky Note | Credential requirements note | — | — | “## 🔐 Required Credentials… SerpAPI… Azure OpenAI… Gmail… Google Sheets… Xpoz MCP…” |
| Trigger Daily Hacker News Scan | Schedule Trigger | Entry point (daily run) | — | Search Hacker News Discussions via Google SERP | “## 📡 Trigger & Search… Scheduled daily scan…” |
| Search Hacker News Discussions via Google SERP | SerpAPI | Google SERP discovery for HN | Trigger Daily Hacker News Scan | Extract SERP Results into Discussion Records | “## 📡 Trigger & Search… Scheduled daily scan…” |
| Extract SERP Results into Discussion Records | Code | Parse SERP response to discussion items | Search Hacker News Discussions via Google SERP | Filter Valid Hacker News Discussion Threads | “## 🔍 Data Processing… filters out noise pages…” |
| Filter Valid Hacker News Discussion Threads | Code | Keep only real HN `/item?id=` threads | Extract SERP Results into Discussion Records | Aggregate Hacker News Discussions for Analysis | “## 🔍 Data Processing… filters out noise pages…” |
| Sticky Note | Sticky Note | Notes on aggregation | — | — | “## 📦 Discussion Aggregation… Combines all valid… into a single dataset…” |
| Aggregate Hacker News Discussions for Analysis | Aggregate | Combine all threads into `$json.data` | Filter Valid Hacker News Discussion Threads | Generate Founder Digest from Hacker News Discussions (AI); Extract High-Signal Founder & Buying-Intent Insights (AI) | “## 📦 Discussion Aggregation… Combines all valid… into a single dataset…” |
| LLM Reasoning Engine for Founder Digest | Azure OpenAI Chat Model (LangChain) | LLM provider for digest agent | — (model connection) | Generate Founder Digest from Hacker News Discussions (AI) | “## 📧 Founder Digest… daily email digest…” |
| Generate Founder Digest from Hacker News Discussions (AI) | LangChain Agent | Create formatted founder digest | Aggregate Hacker News Discussions for Analysis (+ model) | Email Founder Daily Hacker News Digest | “## 📧 Founder Digest… daily email digest…” |
| Email Founder Daily Hacker News Digest | Gmail | Email the digest | Generate Founder Digest from Hacker News Discussions (AI) | — | “## 📧 Founder Digest… daily email digest…” |
| LLM Engine for Lead Qualification & Signal Interpretation | Azure OpenAI Chat Model (LangChain) | LLM provider for lead agent | — (model connection) | Extract High-Signal Founder & Buying-Intent Insights (AI) | “## 🎯 Lead Discovery… enrich… log to Google Sheets…” |
| External Social Context & Enrichment Tool (MCP Client) | MCP Client Tool (LangChain) | Optional social enrichment tool | — (tool connection) | Extract High-Signal Founder & Buying-Intent Insights (AI) | “## 🎯 Lead Discovery… enrich… log to Google Sheets…” |
| Extract High-Signal Founder & Buying-Intent Insights (AI) | LangChain Agent | Extract buying-intent/founder signals | Aggregate Hacker News Discussions for Analysis (+ model + tool) | Normalize AI Insight Output for Storage | “## 🎯 Lead Discovery… enrich… log to Google Sheets…” |
| Normalize AI Insight Output for Storage | Code | Clean agent output for Sheets | Extract High-Signal Founder & Buying-Intent Insights (AI) | Append Founder Insights to Hacker News Intelligence Sheet | “## 🎯 Lead Discovery… enrich… log to Google Sheets…” |
| Append Founder Insights to Hacker News Intelligence Sheet | Google Sheets | Append insights to spreadsheet | Normalize AI Insight Output for Storage | — | “## 🎯 Lead Discovery… enrich… log to Google Sheets…” |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Automate Founder Digest & Lead Discovery from Hacker News with GPT-4o* (or your preferred name).
- Ensure execution order is default (v1 is fine).

2) **Add Schedule Trigger**
- Add node: **Schedule Trigger**
- Configure to run **daily** at your desired time (match your timezone).
- Rename: **Trigger Daily Hacker News Scan**

3) **Add SerpAPI search node**
- Add node: **SerpAPI** (SerpAPI integration)
- Rename: **Search Hacker News Discussions via Google SERP**
- Credentials: connect your **SerpAPI** credential.
- Set query `q` to:  
  `site:news.ycombinator.com ("Show HN" OR "Launch HN" OR "AI" OR "startup" OR "SaaS")`
- Set number of results (`num`) to **5** (or adjust).
- Connect: **Schedule Trigger → SerpAPI**

4) **Add Code node to extract results**
- Add node: **Code**
- Rename: **Extract SERP Results into Discussion Records**
- Paste logic that:
  - Reads `$input.first().json.organic_results || []`
  - Maps to `{ title, summary: snippet, link, date, source }`
  - Returns one n8n item per result
- Connect: **SerpAPI → Extract Code**

5) **Add Code node to filter valid HN discussion threads**
- Add node: **Code**
- Rename: **Filter Valid Hacker News Discussion Threads**
- Implement filtering:
  - Remove noise titles (guidelines / Show page)
  - Keep only URLs containing `news.ycombinator.com/item`
- Connect: **Extract Code → Filter Code**

6) **Add Aggregate node**
- Add node: **Aggregate**
- Rename: **Aggregate Hacker News Discussions for Analysis**
- Set aggregation mode to **Aggregate All Item Data** (single output item containing an array of all items, typically under `data`).
- Connect: **Filter Code → Aggregate**

7) **Set up Azure OpenAI Chat Model (digest)**
- Add node: **Azure OpenAI Chat Model** (LangChain)
- Rename: **LLM Reasoning Engine for Founder Digest**
- Choose model/deployment: **gpt-4o-mini**
- Configure Azure OpenAI credentials (endpoint, API key, deployment/model mapping as required by your n8n version).

8) **Add LangChain Agent for founder digest**
- Add node: **AI Agent** (LangChain Agent)
- Rename: **Generate Founder Digest from Hacker News Discussions (AI)**
- Prompt type: “Define”
- System message: trend analyst, concise, actionable.
- User prompt:
  - Include strict output format headings
  - Include `{{ JSON.stringify($json.data, null, 2) }}` as the input dataset
- Connect main: **Aggregate → Digest Agent**
- Connect AI language model: **LLM Reasoning Engine for Founder Digest → Digest Agent** (language model connection)

9) **Add Gmail send node**
- Add node: **Gmail** → operation **Send**
- Rename: **Email Founder Daily Hacker News Digest**
- Credentials: Gmail OAuth2 (connect account).
- To: your email address (replace `user@example.com`)
- Subject: `🔥 Hacker News Daily Digest`
- Message/body: set expression to `{{ $json.output }}`
- Connect: **Digest Agent → Gmail**

10) **Set up Azure OpenAI Chat Model (lead/signal)**
- Add node: **Azure OpenAI Chat Model** (LangChain)
- Rename: **LLM Engine for Lead Qualification & Signal Interpretation**
- Model/deployment: **gpt-4o-mini**
- Same Azure credentials (or separate, if desired).

11) **Add MCP Client Tool node**
- Add node: **MCP Client Tool** (LangChain)
- Rename: **External Social Context & Enrichment Tool (MCP Client)**
- Endpoint URL: `https://mcp.xpoz.ai/mcp`
- Auth: **Bearer**
- Credentials: store bearer token securely in n8n credentials.

12) **Add LangChain Agent for lead/signal extraction**
- Add node: **AI Agent** (LangChain Agent)
- Rename: **Extract High-Signal Founder & Buying-Intent Insights (AI)**
- Prompt type: “Define”
- Set max iterations to **30**
- System message: founder intelligence analyst; high-signal only; no fluff.
- User prompt:
  - Ask for: Key Signal, Problem Statement, Why This Matters, Actionable Insight
  - Add constraint “use only provided data; do not invent facts”
  - Inject dataset via `{{ JSON.stringify($json.data, null, 2) }}`
- Connect main: **Aggregate → Lead Agent**
- Connect AI language model: **LLM Engine… → Lead Agent**
- Connect tool: **MCP Client Tool → Lead Agent** (tool connection)

13) **Add Code node to normalize output for storage**
- Add node: **Code**
- Rename: **Normalize AI Insight Output for Storage**
- Logic:
  - Read `item.json.output`
  - Strip list markers / headings
  - Output `{ relaxed_output: text }`
- Connect: **Lead Agent → Normalize Code**

14) **Add Google Sheets append node**
- Add node: **Google Sheets**
- Rename: **Append Founder Insights to Hacker News Intelligence Sheet**
- Credentials: Google Sheets OAuth2.
- Operation: **Append**
- Spreadsheet: select your target spreadsheet (replace `YourSpreadsheetId`)
- Sheet/tab: `hacker_news` (create it if missing)
- Ensure a column named `output` exists, then map:
  - `output` = `{{ $json.relaxed_output }}`
- Connect: **Normalize Code → Google Sheets**

15) **(Optional) Add guardrails**
- Add an **IF** node after filtering or after aggregation:
  - If `data` is empty, skip AI + email/sheets to avoid noisy empty outputs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Required credentials: SerpAPI, Azure OpenAI, Gmail OAuth2, Google Sheets OAuth2, Xpoz MCP (bearer). Use OAuth2 where possible and never share personal API keys. | Security note (in-workflow sticky note) |
| This workflow turns Hacker News into a daily intelligence engine by combining SERP-based discovery, AI reasoning, and structured storage. | Main overview sticky note |
| Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | User-provided disclaimer/context |