Track legal risks and litigation threats using Bright Data, OpenRouter and Google Sheets

https://n8nworkflows.xyz/workflows/track-legal-risks-and-litigation-threats-using-bright-data--openrouter-and-google-sheets-13585


# Track legal risks and litigation threats using Bright Data, OpenRouter and Google Sheets

## 1. Workflow Overview

**Title:** Tracking Legal Risks & Litigation Threats with Bright Data & n8n  
**Purpose:** Monitor litigation and regulatory signals for a set of companies by scraping Google Search results via Bright Data, filtering and classifying results with OpenRouter-backed LLM agents, correlating signals by jurisdiction/topic, and writing executive outputs to Google Sheets (alerts, monitoring briefs, and M&A exposure scans).

### 1.1 Input & Scenario Configuration
Loads monitoring parameters (companies, courts, regulators, topics, jurisdiction focus) and routes execution to the “litigation monitoring” scenario.

### 1.2 Data Collection (Search → Scrape → Extract)
Builds a Company × Court matrix, generates Google search URLs, scrapes results with Bright Data, extracts titles/links/snippets, and logs scrape errors to Google Sheets.

### 1.3 Legal Signal Filtering & AI Classification
Normalizes search results, scores keyword relevance, and uses an LLM agent to decide whether each item is a real legal/regulatory event and to classify topic/jurisdiction/court level/risk.

### 1.4 Normalization, Deduplication & Confidence Scoring
Deduplicates events, normalizes them into a consistent schema, computes confidence (0–100), then gates events for downstream correlation and monitoring.

### 1.5 Correlation & Clustering
Clusters events by jurisdiction and topic to reveal concentration patterns and produces a correlated event set for reporting.

### 1.6 Executive Outputs (Alerts, Monitoring Brief, M&A Exposure)
Splits correlated events into individual items, escalates high-risk items to an LLM-generated alert (stored in Sheets), aggregates monitoring events for an LLM-generated brief (stored in Sheets), and produces an LLM-based M&A/partnership exposure assessment (stored in Sheets).

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Scenario Routing
**Overview:** Initializes scenario settings and routes execution into litigation monitoring when enabled.  
**Nodes involved:** Start Legal Scan; Scenario Configuration Loader; Scenario Router – Litigation Monitoring.

#### Node: Start Legal Scan
- **Type/role:** Manual Trigger; workflow entry point.
- **Config:** No parameters; run manually.
- **I/O:** Outputs one empty item → **Scenario Configuration Loader**.
- **Failures:** None (manual start).

#### Node: Scenario Configuration Loader
- **Type/role:** Set; provides all configuration in one item.
- **Config choices (interpreted):**
  - `scenario_types`: `["litigation_monitoring"]`
  - `companies`: `["Google","Amazon","Meta"]`
  - `target_company`: `"OpenAI"` (not used downstream in this JSON)
  - `jurisdictions`: `"United States"` (not used directly in queries here)
  - `courts`: `["U.S. District Court SDNY","U.S. District Court ND California","U.S. Supreme Court"]`
  - `regulators`: `["SEC","FTC","DOJ Antitrust Division","CFPB"]` (declared but not used in current branch)
  - `legal_topics`: `["antitrust","securities","data privacy","consumer protection","labor law"]` (declared but not used in query builder)
  - `batch_mode`: `true` (declared but not used downstream)
- **I/O:** Receives trigger item → outputs configuration item → **Scenario Router – Litigation Monitoring**.
- **Edge cases:** If arrays are empty, later matrix expansion produces no items.

#### Node: Scenario Router – Litigation Monitoring
- **Type/role:** IF; scenario gate.
- **Condition:** `{{ $json.scenario_types.includes("litigation_monitoring") }}`
- **I/O:** True branch → **Company × Court Matrix Expander**. False branch unused (no downstream node shown).
- **Failures:** Expression failure if `scenario_types` is missing or not an array (loose validation is enabled but can still yield unexpected results).

---

### Block 2 — Multi-Source Legal Data Collection (Search + Bright Data + Parsing)
**Overview:** Expands monitoring targets into multiple search requests, scrapes Google results via Bright Data, extracts SERP elements, and logs scraper errors.  
**Nodes involved:** Company × Court Matrix Expander; Search Query & URL Builder; Scrape Legal Data (Bright Data); HTML Extractor – Titles, Links, Snippets; Bright Data Error Formatter; Error Log – Google Sheets.

#### Node: Company × Court Matrix Expander
- **Type/role:** Code; fan-out generator.
- **Logic:** For each `company` in `input.companies` and `court` in `input.courts`, emits a new item with scalar fields:
  - `companies`: single company string
  - `courts`: single court string
  - all other config fields preserved
- **I/O:** Input config item → N items → **Search Query & URL Builder**.
- **Edge cases/failures:**
  - If `companies` or `courts` is not iterable, code throws.
  - Large matrix can explode requests → rate limits (Bright Data/OpenRouter/Sheets).

#### Node: Search Query & URL Builder
- **Type/role:** Code; query construction.
- **Logic:** For each item:
  - `search_query = "${company} ${court} lawsuit"`
  - `search_url = "https://www.google.com/search?q=" + encodeURIComponent(search_query)`
- **I/O:** Items → **Scrape Legal Data (Bright Data)**.
- **Edge cases:** Court/company strings with unusual characters are handled by `encodeURIComponent`.

#### Node: Scrape Legal Data (Bright Data)
- **Type/role:** Bright Data connector; web scraping HTTP fetch through Bright Data zone.
- **Key configuration:**
  - URL: `{{ $json.search_url }}`
  - Zone: `n8n_unlocker`
  - Country: `us`
  - Format: `json`
  - **onError:** `continueErrorOutput` (critical: errors route to the node’s error output)
  - `retryOnFail: true`
- **I/O:**
  - **Main output (success)** → **HTML Extractor – Titles, Links, Snippets**
  - **Error output** → **Bright Data Error Formatter**
- **Likely integration requirements:**
  - Bright Data credential configured (`BrightData account`).
  - Zone must exist in Bright Data.
- **Failure modes:**
  - Auth/zone misconfiguration (401/403)
  - Rate limit / quota exceeded
  - Target page challenges or blocked content
  - Response not containing expected `body` field for HTML parsing

#### Node: HTML Extractor – Titles, Links, Snippets
- **Type/role:** HTML node; CSS extraction from HTML content.
- **Config:**
  - Operation: extract HTML content
  - Source property: `body`
  - Extract:
    - `titles`: selector `h3` (array)
    - `link`: selector `a:has(h3)` attribute `href` (array)
    - `snippet`: selector `div.VwiC3b` (array)
- **I/O:** Bright Data successful items → **Search Result Normalizer**.
- **Edge cases:**
  - Google SERP markup changes (class/selectors may break → empty arrays).
  - If `body` is missing/not HTML, extraction yields empty outputs.

#### Node: Bright Data Error Formatter
- **Type/role:** Set; normalizes scrape errors into a consistent schema.
- **Fields produced:**
  - `errorSource`: `"Bright Data Scraper"`
  - `errorMessage`: `{{$json.error?.message || $json.message || 'Unknown Bright Data Error'}}`
  - `errorCode`: `{{$json.statusCode || $json.code || 'N/A'}}`
- **I/O:** Bright Data error output → **Error Log – Google Sheets**.
- **Edge cases:** Some errors may not contain `error/message/statusCode` keys; defaults cover this.

#### Node: Error Log – Google Sheets
- **Type/role:** Google Sheets append; error logging.
- **Config:**
  - Spreadsheet: `1BnK0JLPzm5NK82cOb0zIcNvgNe2UyrPOrPuaxwsD4vg`
  - Sheet/tab: `BD log error`
  - Operation: Append
  - Mapped columns:
    - `status` ← `errorSource`
    - `error_code` ← `errorCode`
    - `error_message` ← `errorMessage`
- **Credentials:** Google Sheets OAuth2.
- **Failure modes:**
  - OAuth expired / missing permissions
  - Spreadsheet/tab not found or headers mismatch
  - Google API rate limits (noted in sticky note)

---

### Block 3 — Legal Signal Filtering & AI Classification
**Overview:** Converts extracted arrays into individual results, scores legal relevance, then uses an OpenRouter-backed LLM agent with structured output parsing to classify true legal events.  
**Nodes involved:** Search Result Normalizer; Legal Signal Keyword Scorer; AI Legal Case Classifier; Legal Case Classifier (OpenRouter model); Legal Classification Output Parser.

#### Node: Search Result Normalizer
- **Type/role:** Code; converts arrays into per-result items.
- **Logic:** For each input item, iterates up to max length of `titles`, `links`, `snippets`, emits one item per index where link exists:
  - `title`, `link`, `snippet`
  - `source: "Google"`
  - `extracted_at: ISO timestamp`
- **I/O:** HTML Extractor output → **Legal Signal Keyword Scorer**.
- **Edge cases:**
  - Arrays of mismatched length handled; missing title/snippet becomes null.
  - Skips empty links.

#### Node: Legal Signal Keyword Scorer
- **Type/role:** Code; heuristic relevance scoring and top-k selection.
- **Logic:**
  - Scans `title + snippet` for legal keywords; each match adds +2.
  - Link heuristics: `.pdf` +3, contains `court` +3, contains `gov` +3.
  - Sort descending by `legal_score`.
  - Returns only top 5 items.
- **I/O:** Normalized results → **AI Legal Case Classifier**.
- **Edge cases:**
  - If `title`/`snippet` null, concatenation yields `"null ..."`. Consider guarding in future.
  - `item.json.link.includes(...)` will throw if link is null; current code skips items without link earlier, so typically safe.

#### Node: Legal Case Classifier (OpenRouter LM)
- **Type/role:** OpenRouter chat model provider for LangChain agent.
- **Config:** Uses `OpenRouter account` credential; model options not specified (defaults).
- **I/O:** Connected via `ai_languageModel` to **AI Legal Case Classifier**.
- **Failure modes:** Bad API key, model unavailable, rate limits, long latency/timeouts.

#### Node: AI Legal Case Classifier
- **Type/role:** LangChain Agent; prompts LLM to classify each result.
- **Prompt input (`text`):**
  - Title, Link, Snippet from current item.
- **System message (key requirements):**
  - Identify if actual legal case/regulatory action/court filing.
  - Classify legal topic, jurisdiction, court level.
  - Risk level LOW/MEDIUM/HIGH.
  - `primary_source = true` for official court/gov.
  - “Return structured JSON only.”
- **Output parsing:** `hasOutputParser: true` with structured schema node connected.
- **I/O:** Items → **Duplicate Legal Event Filter** (after parsing).
- **Edge cases:**
  - LLM may return invalid JSON or omit required fields → parser failure.
  - Hallucinated citations/dates if snippet lacks them; mitigate with stricter prompting or post-validation.

#### Node: Legal Classification Output Parser
- **Type/role:** Structured Output Parser; enforces JSON schema for classifier output.
- **Schema highlights:**
  - Required: `is_legal_case`, `legal_topic`, `jurisdiction`, `court_level`, `risk_level`, `primary_source`
  - Enums for `type`, `court_level`, `risk_level`
- **I/O:** Feeds parsed structure into **AI Legal Case Classifier** via `ai_outputParser`.
- **Failure modes:** Any schema mismatch; you may want error handling if productionized.

---

### Block 4 — Deduplication, Normalization & Confidence Gate
**Overview:** Removes duplicates, normalizes to an audit-friendly event record, computes confidence, and gates for high-value processing.  
**Nodes involved:** Duplicate Legal Event Filter; Legal Event Normalizer & Confidence Scorer; High-Risk Legal Event Gate.

#### Node: Duplicate Legal Event Filter
- **Type/role:** Remove Duplicates; de-dup.
- **Config:** Compare selected fields: `output.title, output.source_url`
  - Assumes parsed LLM output is in `item.json.output`.
- **I/O:** From **AI Legal Case Classifier** → **Legal Event Normalizer & Confidence Scorer**.
- **Edge cases:**
  - If LLM output lacks `title` or `source_url`, duplicates may not be removed reliably.
  - Different URLs for same case (mirrors/news coverage) will pass through.

#### Node: Legal Event Normalizer & Confidence Scorer
- **Type/role:** Code; normalization + scoring.
- **Key logic:**
  - Drops anything where `raw.is_legal_case !== true`.
  - Normalizes fields into `normalized_legal_event`:
    - `event_id`: lowercase `source_url`
    - `filing_date`: parsed date → `YYYY-MM-DD` or null
    - `risk_level`: retains LLM output (defaults to `"UNKNOWN"` if missing)
    - timestamps included
  - Confidence score starts at 50 and adds:
    - +20 primary source
    - +15 Supreme Court
    - +10 HIGH risk
    - +5 citation present
    - +5 https URL
  - Produces `confidence_score` (max 100) and `confidence_band` (LOW/MEDIUM/HIGH thresholds 60/80)
- **I/O:** Outputs objects with:
  - `normalized_legal_event`
  - `confidence_score`
  - `confidence_band`
  → **High-Risk Legal Event Gate**
- **Failure modes:** If upstream item shape changes (e.g., `output` missing), code may behave unexpectedly; it guards by skipping when `raw` missing.

#### Node: High-Risk Legal Event Gate
- **Type/role:** IF; decides which events proceed to correlation.
- **Condition (OR):**
  - `normalized_legal_event.risk_level == "HIGH"` OR `confidence_score > 70`
- **I/O:**
  - True → **Legal Correlation & Clustering Engine**
  - False → **Monitoring Branch Placeholder** (currently no-op)
- **Edge cases:** If `risk_level` is `"UNKNOWN"` but confidence high, it still passes.

---

### Block 5 — Correlation & Clustering
**Overview:** Builds jurisdiction clusters, topic clusters, and a simplified correlated events list for downstream reporting and analysis.  
**Nodes involved:** Legal Correlation & Clustering Engine.

#### Node: Legal Correlation & Clustering Engine
- **Type/role:** Code; aggregation/clustering across incoming items.
- **Logic:**
  - Normalizes keys to lowercase trimmed strings.
  - Builds:
    - `jurisdiction_clusters`: map keyed by normalized jurisdiction
      - total events, high risk count, events list
    - `topic_clusters`: array keyed by normalized topic
      - total events, jurisdictions list, events list
    - `correlated_events`: simplified array (event_id/title/jurisdiction/topic/risk/confidence)
- **I/O:** Outputs a single item containing all three structures → two branches:
  - **Litigation Events Extractor**
  - **AI M&A Legal Exposure Analyzer**
- **Edge cases:**
  - Large `events` arrays can grow quickly; consider limiting stored event detail.
  - Jurisdiction/topic strings inconsistent (LLM variability) can fragment clusters; consider canonicalization.

---

### Block 6 — High-Risk Escalation Alerts
**Overview:** Splits correlated events, filters high-risk+high-confidence items, generates executive alerts with an LLM, parses structured output, and stores alerts in Google Sheets.  
**Nodes involved:** Litigation Events Extractor; Litigation Event Splitter; High-Risk Escalation Filter; AI High-Risk Litigation Alert Generator; High-Risk Alert Generator (OpenRouter model); High-Risk Alert Output Parser; High-Risk Alerts – Google Sheets.

#### Node: Litigation Events Extractor
- **Type/role:** Set; prepares array for splitting.
- **Config:** `litigation_events = {{ $json.correlated_events }}`
- **I/O:** From correlation engine → **Litigation Event Splitter**.
- **Edge cases:** If `correlated_events` missing, splitter will fail or produce 0 items.

#### Node: Litigation Event Splitter
- **Type/role:** Split Out; turns `litigation_events[]` into one item per event.
- **I/O:** → **High-Risk Escalation Filter**
- **Edge cases:** Ensure field name matches exactly.

#### Node: High-Risk Escalation Filter
- **Type/role:** IF; escalation criteria.
- **Condition (AND):**
  - `risk_level == "HIGH"`
  - `confidence_score >= 70`
- **I/O (True branch):**
  - → **AI High-Risk Litigation Alert Generator**
  - → **Monitoring Event Aggregator** (same item is also used for monitoring stream)
- **False branch:** not connected.
- **Edge cases:** If `confidence_score` is missing (it is present in `correlated_events`), filter might behave unexpectedly.

#### Node: High-Risk Alert Generator (OpenRouter LM)
- **Type/role:** OpenRouter model provider.
- **I/O:** `ai_languageModel` → **AI High-Risk Litigation Alert Generator**.
- **Failure modes:** API key, rate limits, model errors.

#### Node: AI High-Risk Litigation Alert Generator
- **Type/role:** LangChain Agent; generates exec alert.
- **Prompt input:** `Event: {{ JSON.stringify($json) }}`
- **System message:** “Summarize this litigation escalation for a legal executive. Focus on exposure, jurisdiction risk and business impact.”
- **Output parsing:** `hasOutputParser: true` with **High-Risk Alert Output Parser**.
- **I/O:** → **High-Risk Alerts – Google Sheets**
- **Edge cases:** If event lacks detail, the LLM may invent specifics; consider instructing “use only provided data”.

#### Node: High-Risk Alert Output Parser
- **Type/role:** Structured Output Parser; enforces alert schema.
- **Schema highlights:**
  - Required: `alert_title`, `case_name`, `jurisdiction`, `risk_level`, `exposure`, `business_impact`, `recommended_action`
  - Nested objects: `exposure`, `jurisdiction_risk`, `business_impact`
- **I/O:** Feeds parsed schema to the agent via `ai_outputParser`.
- **Failure modes:** Schema mismatch; no explicit error handling.

#### Node: High-Risk Alerts – Google Sheets
- **Type/role:** Google Sheets append; stores alert rows.
- **Sheet/tab:** `High Risk Alerts`
- **Columns mapped (examples):**
  - `case_name = {{ $json.output.case_name }}`
  - `market_impact = {{ $json.output.business_impact.market_impact }}`
  - `financial_risk = {{ $json.output.exposure.financial_risk }}`
  - etc.
- **Edge cases:**
  - Any missing nested field yields blank cell (expression resolves to null/empty).
  - Sheet headers must match the configured mapping.

---

### Block 7 — Monitoring Brief Generation
**Overview:** Aggregates (selected) events into one list, generates an executive monitoring brief using an LLM, parses structured output, and stores it in Google Sheets.  
**Nodes involved:** Monitoring Event Aggregator; AI Litigation Monitoring Summary Generator; Monitoring Summary Generator (OpenRouter model); Monitoring Summary Output Parser; Monitoring Summary – Google Sheets.

#### Node: Monitoring Event Aggregator
- **Type/role:** Aggregate; collects incoming items into one.
- **Config:** Aggregate all item data → `monitoring_events`.
- **I/O:** From **High-Risk Escalation Filter** (true branch) → **AI Litigation Monitoring Summary Generator**.
- **Important behavior:** As currently wired, only escalated high-risk events are aggregated. If you intended to summarize *all* gated events, you’d need to feed it from earlier (e.g., High-Risk Legal Event Gate true branch or correlation output).
- **Failure modes:** Large aggregated payload can exceed LLM context.

#### Node: Monitoring Summary Generator (OpenRouter LM)
- **Type/role:** OpenRouter model provider.
- **I/O:** `ai_languageModel` → **AI Litigation Monitoring Summary Generator**.

#### Node: AI Litigation Monitoring Summary Generator
- **Type/role:** LangChain Agent; creates monitoring brief.
- **Prompt input:** JSON stringified `monitoring_events` plus guidance bullets.
- **System message:** Legal monitoring analyst; concise executive-ready.
- **Output parsing:** `hasOutputParser: true` with **Monitoring Summary Output Parser**.
- **I/O:** → **Monitoring Summary – Google Sheets**
- **Edge cases:** Events array may be empty if no escalations; consider adding a guard/branch for “no events”.

#### Node: Monitoring Summary Output Parser
- **Type/role:** Structured Output Parser; enforces brief schema.
- **Schema highlights:**
  - Required: `brief_title`, `key_developments[]`, `jurisdiction_concentration{...}`, `escalation_signals[]`, `watchlist_recommendation`, `overall_monitoring_risk_level`
- **Failure modes:** LLM output not matching schema.

#### Node: Monitoring Summary – Google Sheets
- **Type/role:** Google Sheets append.
- **Sheet/tab:** `Monitoring summary`
- **Mappings:** Writes arrays into cells (Google Sheets will typically store as comma-joined string or JSON-like string depending on n8n settings).
- **Failure modes:** OAuth, headers mismatch, rate limits.

---

### Block 8 — M&A / Partnership Legal Exposure Assessment
**Overview:** Uses clustered correlation output to generate a transaction-oriented legal exposure assessment and stores it in Google Sheets.  
**Nodes involved:** AI M&A Legal Exposure Analyzer; M&A Exposure Analyzer (OpenRouter model); M&A Exposure Output Parser; M&A Exposure – Google Sheets.

#### Node: M&A Exposure Analyzer (OpenRouter LM)
- **Type/role:** OpenRouter model provider.
- **I/O:** `ai_languageModel` → **AI M&A Legal Exposure Analyzer**.

#### Node: AI M&A Legal Exposure Analyzer
- **Type/role:** LangChain Agent; M&A exposure analysis.
- **Prompt input:** JSON of `correlated_events`, `jurisdiction_clusters`, `topic_clusters` plus explicit focus areas and posture recommendation.
- **System message:** “senior M&A legal risk advisor… executive-ready.”
- **Output parsing:** `hasOutputParser: true` with **M&A Exposure Output Parser**.
- **I/O:** → **M&A Exposure – Google Sheets**
- **Edge cases:** If there are no events, LLM may still generate a posture; consider handling empty input explicitly.

#### Node: M&A Exposure Output Parser
- **Type/role:** Structured Output Parser; enforces report schema.
- **Schema highlights:**
  - Required: `report_type`, `overall_transaction_risk`, `risk_drivers[]`, `recommended_transaction_posture`
  - Optional arrays: `high_risk_cases[]`, `due_diligence_red_flags[]`
- **Failure modes:** Schema mismatch.

#### Node: M&A Exposure – Google Sheets
- **Type/role:** Google Sheets append.
- **Sheet/tab:** `M&A / Partnership Legal Exposure Scan`
- **Notable mapping detail:** Uses only the first element of `high_risk_cases`:
  - `event_id = {{ $json.output.high_risk_cases[0].event_id }}`
  - If `high_risk_cases` is empty, these expressions will evaluate to null/blank (and may throw in stricter environments).
- **Recommendation:** Add guards like `{{$json.output.high_risk_cases?.[0]?.event_id}}` if your n8n version supports optional chaining in expressions.

---

### Block 9 — Placeholder / Unused Branch
**Overview:** Holds a “monitoring branch” output that currently does nothing.  
**Nodes involved:** Monitoring Branch Placeholder.

#### Node: Monitoring Branch Placeholder
- **Type/role:** NoOp; placeholder for future logic.
- **I/O:** Receives false branch from High-Risk Legal Event Gate.
- **Use case:** Add handling for medium/low risk events, long-term watchlists, or storage.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Legal Scan | Manual Trigger | Entry point | — | Scenario Configuration Loader | ##  Legal Risk & Litigation Intelligence Orchestration Engine; This workflow monitors companies across courts, regulators, and jurisdictions to detect, classify, and correlate legal risk signals. … What you need: Bright Data, OpenRouter, Google Sheets. |
| Scenario Configuration Loader | Set | Load scenario parameters | Start Legal Scan | Scenario Router – Litigation Monitoring | Customize Here; Edit this node to set: companies, jurisdictions, courts, regulators, legal_topics |
| Scenario Router – Litigation Monitoring | IF | Scenario gate | Scenario Configuration Loader | Company × Court Matrix Expander | Customize Here; Edit this node to set: companies, jurisdictions, courts, regulators, legal_topics |
| Company × Court Matrix Expander | Code | Expand company×court combinations | Scenario Router – Litigation Monitoring | Search Query & URL Builder | ## Multi-Source Legal Data Collection; Expands companies × courts matrix… |
| Search Query & URL Builder | Code | Build query and Google URL | Company × Court Matrix Expander | Scrape Legal Data (Bright Data) | ## Multi-Source Legal Data Collection; Expands companies × courts matrix… |
| Scrape Legal Data (Bright Data) | Bright Data | Scrape Google SERP | Search Query & URL Builder | HTML Extractor – Titles, Links, Snippets; Bright Data Error Formatter | ## Multi-Source Legal Data Collection; Expands companies × courts matrix… |
| HTML Extractor – Titles, Links, Snippets | HTML | Extract SERP fields | Scrape Legal Data (Bright Data) | Search Result Normalizer | ## Multi-Source Legal Data Collection; Expands companies × courts matrix… |
| Bright Data Error Formatter | Set | Normalize scrape errors | Scrape Legal Data (Bright Data) error output | Error Log – Google Sheets | ## Multi-Source Legal Data Collection; … Logs scraper errors automatically |
| Error Log – Google Sheets | Google Sheets | Append scrape errors | Bright Data Error Formatter | — | ## Multi-Source Legal Data Collection; … Logs scraper errors automatically |
| Search Result Normalizer | Code | Turn arrays into per-result items | HTML Extractor – Titles, Links, Snippets | Legal Signal Keyword Scorer | ## Legal Signal Filtering & AI Classification; Scores keyword-based legal relevance… |
| Legal Signal Keyword Scorer | Code | Heuristic legal relevance scoring | Search Result Normalizer | AI Legal Case Classifier | ## Legal Signal Filtering & AI Classification; Scores keyword-based legal relevance… |
| Legal Case Classifier | OpenRouter Chat Model | LLM provider for classifier agent | — | AI Legal Case Classifier (ai_languageModel) | ## Legal Signal Filtering & AI Classification; … |
| Legal Classification Output Parser | Structured Output Parser | Enforce classifier JSON schema | — | AI Legal Case Classifier (ai_outputParser) | ## Legal Signal Filtering & AI Classification; … |
| AI Legal Case Classifier | LangChain Agent | Classify legal events | Legal Signal Keyword Scorer; (LM + parser via AI ports) | Duplicate Legal Event Filter | ## Legal Signal Filtering & AI Classification; … |
| Duplicate Legal Event Filter | Remove Duplicates | De-dup by title+URL | AI Legal Case Classifier | Legal Event Normalizer & Confidence Scorer | ## Normalization & Confidence Scoring; Standardizes… Removes duplicate events |
| Legal Event Normalizer & Confidence Scorer | Code | Normalize + compute confidence | Duplicate Legal Event Filter | High-Risk Legal Event Gate | ## Normalization & Confidence Scoring; Standardizes… Calculates confidence score |
| High-Risk Legal Event Gate | IF | Gate for correlation (risk/confidence) | Legal Event Normalizer & Confidence Scorer | Legal Correlation & Clustering Engine; Monitoring Branch Placeholder | ## Normalization & Confidence Scoring; … Filters high-risk escalations |
| Monitoring Branch Placeholder | NoOp | Placeholder for non-gated events | High-Risk Legal Event Gate (false) | — |  |
| Legal Correlation & Clustering Engine | Code | Cluster by jurisdiction/topic | High-Risk Legal Event Gate (true) | Litigation Events Extractor; AI M&A Legal Exposure Analyzer | ## Correlation & Risk Clustering; Clusters by jurisdiction… |
| Litigation Events Extractor | Set | Prepare events array | Legal Correlation & Clustering Engine | Litigation Event Splitter | ## Correlation & Risk Clustering; Clusters by jurisdiction… |
| Litigation Event Splitter | Split Out | Split events into items | Litigation Events Extractor | High-Risk Escalation Filter | ## Correlation & Risk Clustering; Clusters by jurisdiction… |
| High-Risk Escalation Filter | IF | Escalate high-risk events | Litigation Event Splitter | AI High-Risk Litigation Alert Generator; Monitoring Event Aggregator | ## Executive Intelligence Outputs; Generates structured reports for High-Risk Alerts and Monitoring Briefs |
| High-Risk Alert Generator | OpenRouter Chat Model | LLM provider for alert agent | — | AI High-Risk Litigation Alert Generator (ai_languageModel) | ## Executive Intelligence Outputs; … |
| High-Risk Alert Output Parser | Structured Output Parser | Enforce alert JSON schema | — | AI High-Risk Litigation Alert Generator (ai_outputParser) | ## Executive Intelligence Outputs; … |
| AI High-Risk Litigation Alert Generator | LangChain Agent | Generate executive escalation alert | High-Risk Escalation Filter; (LM + parser via AI ports) | High-Risk Alerts – Google Sheets | ## Executive Intelligence Outputs; … |
| High-Risk Alerts – Google Sheets | Google Sheets | Store alerts | AI High-Risk Litigation Alert Generator | — | ## Executive Intelligence Outputs; … |
| Monitoring Event Aggregator | Aggregate | Aggregate items into monitoring list | High-Risk Escalation Filter | AI Litigation Monitoring Summary Generator | ## Executive Intelligence Outputs; … |
| Monitoring Summary Generator | OpenRouter Chat Model | LLM provider for monitoring brief | — | AI Litigation Monitoring Summary Generator (ai_languageModel) | ## Executive Intelligence Outputs; … |
| Monitoring Summary Output Parser | Structured Output Parser | Enforce brief JSON schema | — | AI Litigation Monitoring Summary Generator (ai_outputParser) | ## Executive Intelligence Outputs; … |
| AI Litigation Monitoring Summary Generator | LangChain Agent | Generate monitoring brief | Monitoring Event Aggregator; (LM + parser via AI ports) | Monitoring Summary – Google Sheets | ## Executive Intelligence Outputs; … |
| Monitoring Summary – Google Sheets | Google Sheets | Store monitoring brief | AI Litigation Monitoring Summary Generator | — | ## Executive Intelligence Outputs; … |
| M&A Exposure Analyzer | OpenRouter Chat Model | LLM provider for M&A analysis | — | AI M&A Legal Exposure Analyzer (ai_languageModel) | ## Correlation & Risk Clustering; … Surfaces concentration red flags |
| M&A Exposure Output Parser | Structured Output Parser | Enforce M&A report schema | — | AI M&A Legal Exposure Analyzer (ai_outputParser) | ## Correlation & Risk Clustering; … |
| AI M&A Legal Exposure Analyzer | LangChain Agent | Generate transaction exposure assessment | Legal Correlation & Clustering Engine; (LM + parser via AI ports) | M&A Exposure – Google Sheets | ## Correlation & Risk Clustering; … |
| M&A Exposure – Google Sheets | Google Sheets | Store M&A assessment | AI M&A Legal Exposure Analyzer | — | ## Correlation & Risk Clustering; … |
| Sticky Note | Sticky Note | Workflow description | — | — | ##  Legal Risk & Litigation Intelligence Orchestration Engine… |
| Sticky Note1 | Sticky Note | Data collection block note | — | — | ## Multi-Source Legal Data Collection… |
| Sticky Note2 | Sticky Note | Classification block note | — | — | ## Legal Signal Filtering & AI Classification… |
| Sticky Note3 | Sticky Note | Normalization block note | — | — | ## Normalization & Confidence Scoring… |
| Sticky Note4 | Sticky Note | Correlation block note | — | — | ## Correlation & Risk Clustering… |
| Sticky Note5 | Sticky Note | Executive outputs note | — | — | ## Executive Intelligence Outputs… |
| Sticky Note6 | Sticky Note | Credential setup note | — | — | ## Setup Instructions… |
| Sticky Note7 | Sticky Note | Config customization note | — | — | Customize Here… |
| Sticky Note8 | Sticky Note | Google Sheets tab/headers note | — | — | ## Google Sheets Setup… |
| Sticky Note9 | Sticky Note | Rate limiting note | — | — | ## Rate Limiting Advisory… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named “Tracking Legal Risks & Litigation Threats with Bright Data & n8n”.

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `Start Legal Scan`

3) **Add configuration loader**
   - Node: **Set**
   - Name: `Scenario Configuration Loader`
   - Add fields:
     - `scenario_types` (Array): `["litigation_monitoring"]`
     - `companies` (Array): `["Google","Amazon","Meta"]`
     - `target_company` (String): `OpenAI`
     - `jurisdictions` (String): `United States`
     - `courts` (Array): `["U.S. District Court SDNY","U.S. District Court ND California","U.S. Supreme Court"]`
     - `regulators` (Array): `["SEC","FTC","DOJ Antitrust Division","CFPB"]`
     - `legal_topics` (Array): `["antitrust","securities","data privacy","consumer protection","labor law"]`
     - `batch_mode` (Boolean): `true`
   - Connect: **Start Legal Scan → Scenario Configuration Loader**

4) **Add scenario router**
   - Node: **IF**
   - Name: `Scenario Router – Litigation Monitoring`
   - Condition (Boolean true):
     - Left value: `={{ $json.scenario_types.includes("litigation_monitoring") }}`
   - Connect: **Scenario Configuration Loader → Scenario Router – Litigation Monitoring** (true path used)

5) **Add matrix expander**
   - Node: **Code**
   - Name: `Company × Court Matrix Expander`
   - Paste logic to expand `companies × courts` (as in workflow behavior).
   - Connect: **Scenario Router (true) → Company × Court Matrix Expander**

6) **Add search builder**
   - Node: **Code**
   - Name: `Search Query & URL Builder`
   - Create:
     - `search_query = "${company} ${court} lawsuit"`
     - `search_url = https://www.google.com/search?q=...`
   - Connect: **Company × Court Matrix Expander → Search Query & URL Builder**

7) **Add Bright Data scrape**
   - Node: **Bright Data** (package `@brightdata/n8n-nodes-brightdata`)
   - Name: `Scrape Legal Data (Bright Data)`
   - Set:
     - URL: `={{ $json.search_url }}`
     - Zone: your Bright Data zone (e.g., `n8n_unlocker`)
     - Country: `us`
     - Format: `json`
   - Turn on:
     - **Retry on fail**
     - **On Error:** “Continue (Error Output)”
   - Configure credential: **BrightData account** (API token).
   - Connect: **Search Query & URL Builder → Scrape Legal Data (Bright Data)**

8) **Add HTML extraction**
   - Node: **HTML**
   - Name: `HTML Extractor – Titles, Links, Snippets`
   - Operation: Extract HTML Content
   - Property containing HTML: `body`
   - Extraction rules:
     - `titles`: selector `h3`, return array
     - `link`: selector `a:has(h3)`, attribute `href`, return array
     - `snippet`: selector `div.VwiC3b`, return array
   - Connect: **Scrape Legal Data (Bright Data) main → HTML Extractor**

9) **Add Bright Data error logging path**
   - Node: **Set** named `Bright Data Error Formatter` with:
     - `errorSource` = `Bright Data Scraper`
     - `errorMessage` = `={{$json.error?.message || $json.message || 'Unknown Bright Data Error'}}`
     - `errorCode` = `={{$json.statusCode || $json.code || 'N/A'}}`
   - Node: **Google Sheets** named `Error Log – Google Sheets`
     - Operation: Append
     - Document: your spreadsheet
     - Sheet/tab: `BD log error`
     - Map columns: `status`, `error_code`, `error_message`
     - Credential: **Google Sheets OAuth2**
   - Connect: **Scrape Legal Data (Bright Data) error → Bright Data Error Formatter → Error Log – Google Sheets**

10) **Normalize SERP results**
   - Node: **Code** named `Search Result Normalizer`
   - Emit one item per result (title/link/snippet), include `source` and `extracted_at`.
   - Connect: **HTML Extractor → Search Result Normalizer**

11) **Score legal keyword relevance**
   - Node: **Code** named `Legal Signal Keyword Scorer`
   - Implement keyword scoring and keep top 5.
   - Connect: **Search Result Normalizer → Legal Signal Keyword Scorer**

12) **Add OpenRouter LLM model (classifier)**
   - Node: **OpenRouter Chat Model** (LangChain connector) named `Legal Case Classifier`
   - Credential: **OpenRouter account** (API key)
   - Choose model in node options if desired (not specified in provided workflow).

13) **Add structured output parser (classifier)**
   - Node: **Structured Output Parser** named `Legal Classification Output Parser`
   - Schema: boolean `is_legal_case`, enums for `type/court_level/risk_level`, etc. (match your desired schema).
   
14) **Add LangChain agent (classifier)**
   - Node: **AI Agent** named `AI Legal Case Classifier`
   - Prompt includes Title/Link/Snippet from input item.
   - System message instructs “Return structured JSON only” with required fields.
   - Attach:
     - `Legal Case Classifier` to agent’s **AI Language Model** port
     - `Legal Classification Output Parser` to agent’s **AI Output Parser** port
   - Connect: **Legal Signal Keyword Scorer → AI Legal Case Classifier**

15) **Deduplicate**
   - Node: **Remove Duplicates** named `Duplicate Legal Event Filter`
   - Compare: selected fields `output.title` and `output.source_url`
   - Connect: **AI Legal Case Classifier → Duplicate Legal Event Filter**

16) **Normalize + confidence scoring**
   - Node: **Code** named `Legal Event Normalizer & Confidence Scorer`
   - Skip non-legal cases; output:
     - `normalized_legal_event`
     - `confidence_score`
     - `confidence_band`
   - Connect: **Duplicate Legal Event Filter → Legal Event Normalizer & Confidence Scorer**

17) **Gate high-value events**
   - Node: **IF** named `High-Risk Legal Event Gate`
   - OR conditions:
     - `={{ $json.normalized_legal_event.risk_level }}` equals `HIGH`
     - `={{ $json.confidence_score }}` > `70`
   - Add **NoOp** node named `Monitoring Branch Placeholder` to false output (optional).
   - Connect:
     - **Legal Event Normalizer & Confidence Scorer → High-Risk Legal Event Gate**
     - False → **Monitoring Branch Placeholder**

18) **Correlation & clustering**
   - Node: **Code** named `Legal Correlation & Clustering Engine`
   - Produce one item with `correlated_events`, `jurisdiction_clusters`, `topic_clusters`.
   - Connect: **High-Risk Legal Event Gate (true) → Legal Correlation & Clustering Engine**

19) **Prepare and split events for alerting**
   - Node: **Set** `Litigation Events Extractor`: `litigation_events = {{$json.correlated_events}}`
   - Node: **Split Out** `Litigation Event Splitter` splitting `litigation_events`
   - Node: **IF** `High-Risk Escalation Filter`:
     - `risk_level == HIGH` AND `confidence_score >= 70`
   - Connect:
     - **Legal Correlation & Clustering Engine → Litigation Events Extractor → Litigation Event Splitter → High-Risk Escalation Filter**

20) **High-risk alert LLM + Sheets**
   - Add OpenRouter model node `High-Risk Alert Generator`
   - Add parser `High-Risk Alert Output Parser` (alert schema)
   - Add agent `AI High-Risk Litigation Alert Generator`:
     - Input: `Event: {{JSON.stringify($json)}}`
     - System message: exec escalation summary focus
     - Attach model + parser via AI ports
   - Add Google Sheets node `High-Risk Alerts – Google Sheets`:
     - Append to tab `High Risk Alerts`
     - Map fields from `$json.output...`
   - Connect:
     - **High-Risk Escalation Filter (true) → AI High-Risk Litigation Alert Generator → High-Risk Alerts – Google Sheets**

21) **Monitoring brief LLM + Sheets**
   - Node: **Aggregate** `Monitoring Event Aggregator` → destination `monitoring_events`
   - Add OpenRouter model `Monitoring Summary Generator`
   - Add parser `Monitoring Summary Output Parser`
   - Add agent `AI Litigation Monitoring Summary Generator`:
     - Input includes `{{JSON.stringify($json.monitoring_events)}}`
     - Attach model + parser
   - Add Google Sheets `Monitoring Summary – Google Sheets` to tab `Monitoring summary`
   - Connect:
     - **High-Risk Escalation Filter (true) → Monitoring Event Aggregator → AI Litigation Monitoring Summary Generator → Monitoring Summary – Google Sheets**

22) **M&A exposure assessment LLM + Sheets**
   - Add OpenRouter model `M&A Exposure Analyzer`
   - Add parser `M&A Exposure Output Parser`
   - Add agent `AI M&A Legal Exposure Analyzer`:
     - Input includes correlated events + clusters from correlation engine item
     - Attach model + parser
   - Add Google Sheets `M&A Exposure – Google Sheets` to tab `M&A / Partnership Legal Exposure Scan`
   - Connect:
     - **Legal Correlation & Clustering Engine → AI M&A Legal Exposure Analyzer → M&A Exposure – Google Sheets**

23) **Create Google Sheet tabs and headers**
   - Create 4 tabs with headers exactly as specified in Sticky Note “Google Sheets Setup”:
     - `Monitoring summary`
     - `High Risk Alerts`
     - `M&A / Partnership Legal Exposure Scan`
     - `BD log error`

24) **Credential configuration**
   - **Bright Data:** add API token; ensure zone exists.
   - **OpenRouter:** add API key; optionally select a model.
   - **Google Sheets OAuth2:** connect Google account; ensure spreadsheet access.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer |
| Bright Data, OpenRouter, and Google Sheets credentials must be configured before running. | Sticky note “Setup Instructions” |
| Google Sheets must contain 4 tabs with specific headers (Monitoring summary, High Risk Alerts, M&A / Partnership Legal Exposure Scan, BD log error). | Sticky note “Google Sheets Setup” |
| Rate limiting risk: Bright Data API, OpenRouter LLM API, Google Sheets API (100 requests / 100s / user). Consider Wait nodes or batching. | Sticky note “Rate Limiting Advisory” |
| Workflow intent: scrape → filter/score → AI classification → correlation → executive outputs (alerts + briefs). | Sticky note “Legal Risk & Litigation Intelligence Orchestration Engine” |