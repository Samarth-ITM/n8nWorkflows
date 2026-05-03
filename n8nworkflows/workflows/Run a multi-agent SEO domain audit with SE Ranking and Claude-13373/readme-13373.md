Run a multi-agent SEO domain audit with SE Ranking and Claude

https://n8nworkflows.xyz/workflows/run-a-multi-agent-seo-domain-audit-with-se-ranking-and-claude-13373


# Run a multi-agent SEO domain audit with SE Ranking and Claude

## 1. Workflow Overview

**Workflow name:** Run a multi-agent SEO domain audit with SE Ranking and Claude  
**Purpose:** Generate a comprehensive SEO strategy report for a given domain by collecting SEO/audit data from SE Ranking, running 4 specialist AI analyses (technical, links, keywords, AI visibility), and synthesizing them via a Strategy Director into a prioritized 90‑day action plan. Outputs are exported to Google Drive (full markdown report) and Google Sheets (metrics row).

**Primary use cases**
- SEO agencies producing quick competitive and technical audits
- Content/marketing teams planning keyword/content roadmaps
- Teams monitoring “AI search visibility” across major AI engines

### 1.1 Input Reception & Normalization
User submits a form (domain, business description, target market, competitors). The workflow normalizes the domain and builds a configuration object used across all downstream nodes.

### 1.2 Data Collection (SE Ranking, parallel)
Using the config, the workflow pulls:
- Domain overview
- Top keywords (top 100 by volume)
- Competitors
- Backlinks summary + anchor texts
- AI Search leaderboard visibility
- Website audit report (created, then polled until finished)

### 1.3 Data Unification & Formatting
All collected outputs are merged and transformed into a single structured JSON payload optimized for LLM prompting (summaries, distributions, top lists, and computed metrics).

### 1.4 Multi-Agent AI Analysis (4 specialists + 1 director)
Four specialist agents analyze the formatted dataset in parallel using Claude Sonnet. Their outputs are compiled and then fed to a Strategy Director agent using Claude Opus to produce an executive strategy and a 90‑day plan.

### 1.5 Report Compilation & Export
A final markdown report is built (including summary tables and collapsible detailed agent outputs). Metrics are appended/updated in Google Sheets, and the full report markdown is saved to Google Drive.

---

## 2. Block-by-Block Analysis

### Block 1 — Documentation / Canvas Notes
**Overview:** Provides human context (audience, requirements, setup, customization). No data flow impact.  
**Nodes involved:** `Overview` (sticky note), `Data Collection Phase` (sticky note), `AI Agents Phase` (sticky note), `Output Phase` (sticky note)

#### Node: Overview
- **Type/role:** Sticky Note (documentation)
- **Configuration choices:** Contains requirements and setup links:
  - SE Ranking community node npm link: https://www.npmjs.com/package/@seranking/n8n-nodes-seranking  
  - SE Ranking API dashboard: https://online.seranking.com/admin.api.dashboard.html
- **Connections:** None
- **Edge cases:** None

#### Node: Data Collection Phase
- **Type/role:** Sticky Note
- **Connections:** None

#### Node: AI Agents Phase
- **Type/role:** Sticky Note
- **Connections:** None

#### Node: Output Phase
- **Type/role:** Sticky Note
- **Connections:** None

---

### Block 2 — Input Reception & Configuration
**Overview:** Accepts user input from a form and transforms it into a normalized configuration JSON (domain cleaned, defaults applied).  
**Nodes involved:** `Domain Input Form`, `Set Configuration`

#### Node: Domain Input Form
- **Type/role:** `formTrigger` (entry point)
- **Key configuration:**
  - Title: “AI SEO Strategy Agent”
  - Fields:
    - **Domain** (required)
    - **Business Description** (optional textarea)
    - **Target Market** (required dropdown): `us, uk, de, fr, es, it, au, ca`
    - **Main Competitors (comma-separated)** (optional)
- **Output:** One item containing the submitted fields as JSON.
- **Edge cases / failures:**
  - User enters a URL path (e.g., `example.com/page`), which is handled later.
  - User includes protocol or `www`, handled in `Set Configuration`.

#### Node: Set Configuration
- **Type/role:** `set` (raw JSON shaping)
- **Key configuration choices:**
  - Produces a single JSON object with:
    - `domain`: cleans input by removing `https://`, `http://`, `www.`, then taking first path segment
    - `business_description`: defaults to `"Not provided"`
    - `source`: target market dropdown value (used by SE Ranking endpoints)
    - `competitors`: raw comma-separated string or `""`
    - `report_date`: current date `YYYY-MM-DD`
    - `ai_engines`: `["chatgpt","perplexity","gemini","ai-overview"]`
- **Key expressions:**
  - Domain cleanup: `{{ $json['Domain'].replace(...).split('/')[0] }}`
  - Date: `{{ new Date().toISOString().split('T')[0] }}`
- **Connections:**
  - **Inputs:** `Domain Input Form`
  - **Outputs:** fans out to all data collection nodes + audit creation + competitor preparation.
- **Edge cases / failures:**
  - If user enters an invalid domain string, SE Ranking nodes may fail with validation/API errors.
  - `competitors` may be empty string, which later causes issues in competitor parsing if not guarded (see Block 3.4).

---

### Block 3 — Data Collection (SE Ranking + audit polling)
**Overview:** Collects all required SEO datasets. Most requests run in parallel. The website audit is asynchronous: it is created, then the workflow waits and polls until finished.  
**Nodes involved:**  
`Get Domain Overview`, `Get Top Keywords`, `Get Competitors`, `Get Backlinks Summary`, `Get Backlinks Anchors`, `Create standard audit`, `Wait for audit (1 min)`, `Get Audit Report`, `If audit still running`, `Prepare competitors for AI Leaderboard`, `Get AI Search Leaderboard`, `Merge All Data`

#### Node: Get Domain Overview
- **Type/role:** SE Ranking community node `@seranking/n8n-nodes-seranking.seRanking` (domain overview)
- **Configuration choices:**
  - Domain: `={{ $json.domain }}`
  - Operation appears default (not explicitly set), implying “domain overview” endpoint.
- **Credentials:** `seRankingApi`
- **Connections:**  
  - **Input:** `Set Configuration`  
  - **Output:** `Merge All Data` (input index 0)
- **Failure modes:**
  - SE Ranking auth/token invalid
  - Rate limiting / API quotas
  - Domain not found or unsupported market data

#### Node: Get Top Keywords
- **Type/role:** SE Ranking node (keywords retrieval)
- **Configuration choices:**
  - Operation: `getKeywords`
  - Domain: from `Set Configuration`
  - Source: from `Set Configuration` (country/market)
  - Additional fields:
    - `limit: 100`
    - `orderField: volume`, `orderType: desc`
- **Connections:** output to `Merge All Data` (index 1)
- **Failure modes:**
  - Same as above; additionally large domains may return fewer/empty results

#### Node: Get Competitors
- **Type/role:** SE Ranking node (competitors by keyword overlap)
- **Configuration choices:**
  - Operation: `getCompetitors`
  - Domain + Source from config
- **Connections:** output to `Merge All Data` (index 2)

#### Node: Get Backlinks Summary
- **Type/role:** SE Ranking node (backlink summary)
- **Configuration choices:**
  - Resource: `backlinks`
  - Target: domain from config
  - Operation appears default for summary.
- **Connections:** output to `Merge All Data` (index 3)

#### Node: Get Backlinks Anchors
- **Type/role:** SE Ranking node (anchor texts)
- **Configuration choices:**
  - Resource: `backlinks`
  - Operation: `getAnchors`
  - Limit: `50`
- **Connections:** output to `Merge All Data` (index 4)

#### Node: Create standard audit
- **Type/role:** SE Ranking node (starts a website audit crawl)
- **Configuration choices:**
  - Resource: `websiteAudit`
  - Operation: `createStandard`
  - Domain from config
- **Connections:** output to `Wait for audit (1 min)`
- **Failure modes:**
  - Audit creation may fail if domain cannot be crawled / project limits
  - API returns audit id; missing id breaks downstream `Get Audit Report`

#### Node: Wait for audit (1 min)
- **Type/role:** `wait` (delay for asynchronous completion)
- **Configuration choices:** Unit: minutes, value implicitly `1`
- **Connections:** to `Get Audit Report`
- **Failure modes:** none typical; increases total runtime

#### Node: Get Audit Report
- **Type/role:** SE Ranking node (fetch audit report by ID)
- **Configuration choices:**
  - Resource: `websiteAudit`
  - Operation: `getReport`
  - `auditId: ={{ $('Create standard audit').item.json.id }}`
- **Connections:** to `If audit still running`
- **Failure modes:**
  - Missing/incorrect `auditId`
  - Report not ready yet (handled by loop)
  - API throttling if polled too frequently

#### Node: If audit still running
- **Type/role:** `if` (polling control)
- **Condition:** `is_finished == false`
- **Connections:**
  - **True branch:** back to `Wait for audit (1 min)` (creates a polling loop)
  - **False branch:** to `Merge All Data` (index 6)
- **Edge cases:**
  - If `is_finished` is missing/null, strict boolean equals may evaluate false and incorrectly treat audit as finished or fail type validation depending on n8n behavior.
  - Infinite loop risk if audit never completes; consider adding a max attempts counter.

#### Node: Prepare competitors for AI Leaderboard
- **Type/role:** `code` (parse competitor list into structured objects)
- **Logic:**
  - Reads config from input.
  - Splits `config.competitors` by comma, trims, maps to `{ target, brand }`.
  - Brand is derived from first subdomain token: `domain.split('.')[0]` with capitalization.
- **Connections:** to `Get AI Search Leaderboard`
- **Edge cases / failures:**
  - If `competitors` is empty string, `.split(',')` yields `[""]` → produces `{target:"", brand:""}`.
  - If fewer than 3 competitors are provided, downstream node references indices `[0]`, `[1]`, `[2]` and can error (see next node).

#### Node: Get AI Search Leaderboard
- **Type/role:** SE Ranking node (AI search visibility leaderboard)
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getLeaderboard`
  - Engines: `chatgpt`, `ai-overview`, `perplexity`, `gemini`
  - Source (market) from config
  - Primary:
    - `primaryBrand`: first token of domain `domain.split('.')[0]`
    - `primaryTarget`: domain
  - Competitors: explicitly maps the first 3 competitorValues:
    - `brand/target` for indices `[0]`, `[1]`, `[2]`
- **Connections:** output to `Merge All Data` (index 5)
- **Failure modes / important edge cases:**
  - If competitorValues has < 3 entries, expressions like `$json.competitorValues[2].brand` can throw expression errors.
  - If competitor targets are blank/invalid, API may reject or return empty results.
  - API availability/rate limits.

#### Node: Merge All Data
- **Type/role:** `merge` (multi-input aggregator)
- **Configuration:** `numberInputs: 7`
- **Expected inputs (by connection index):**
  0. Domain overview
  1. Top keywords
  2. Competitors
  3. Backlinks summary
  4. Backlinks anchors
  5. AI leaderboard
  6. Website audit report (after polling completes)
- **Output:** Single merged stream forwarded to formatting.
- **Edge cases:**
  - If any upstream branch fails or returns zero items, merge behavior can stall or output incomplete data depending on merge mode defaults (not explicitly set here).

---

### Block 4 — Data Formatting for LLM Consumption
**Overview:** Consolidates node outputs into one structured object with computed summaries (keyword distribution, dofollow ratio, CWV labels, section issue aggregation, AI visibility leaderboard normalization).  
**Nodes involved:** `Format Data for AI Agents`

#### Node: Format Data for AI Agents
- **Type/role:** `code` (JavaScript transformation)
- **Key configuration choices:**
  - Uses `$('Node Name').first()/all()` to pull data directly from named nodes (not only from incoming items).
  - Produces one item: `{ json: formattedData }`
- **Key computed fields:**
  - `overview`: organic/paid traffic, keywords, traffic value from `domainOverview`
  - `keywords`: top 20 details + distribution buckets + average position
  - `competitors`: top 10 competitor summary
  - `backlinks`: totals + dofollow ratio + top anchors (first 10)
  - `technical`: audit scores, totals, CWV “Good/Needs Improvement/Poor”, and issues by section (errors/warnings extracted from `sections[].props`)
  - `ai_visibility`: your engine stats + leaderboard table + competitor stats
- **Important note (data mismatch risk):**
  - The `Link Building Agent` prompt references `backlinks.domain_inlink_rank`, but **this formatter does not set `domain_inlink_rank`**. That field will be `undefined` in prompts unless SE Ranking summary includes it and you add it here.
- **Edge cases / failures:**
  - If any upstream node returns no items, `.first()` may throw or return undefined depending on n8n version; many accesses are guarded with optional chaining (`?.`) and fallbacks, but not all.
  - `websiteAudit.sections` structure assumptions: if `section.props` is not an object of issue descriptors with `status/value/name`, filtering may break.
  - `aiLeaderboard.results?.[config.domain]` requires exact key match; if API returns normalized domain forms, you may get zeros.

---

### Block 5 — Specialist Agent Analysis (4 parallel agents)
**Overview:** Runs 4 dedicated LLM “agent” nodes, each receiving the formatted dataset and producing a markdown analysis in its specialty. All use the same Anthropic model connection (“Claude Sonnet”).  
**Nodes involved:** `Claude Sonnet (Specialist Agents)`, `Technical SEO Agent`, `Link Building Agent`, `Keyword Strategy Agent`, `AI Visibility Agent`

#### Node: Claude Sonnet (Specialist Agents)
- **Type/role:** LangChain Chat Model connector (`lmChatAnthropic`)
- **Configuration choices:** Model: `claude-sonnet-4-5-20250929`
- **Connections:** Provides `ai_languageModel` input to all four specialist agents.
- **Failure modes:** invalid Anthropic key, model not available in region/account, rate limits.

#### Node: Technical SEO Agent
- **Type/role:** LangChain Agent node
- **Prompt construction:**
  - Injects: health score, pages, errors/warnings, domain trust, CWV, issues sections JSON.
  - System message requests prioritized categories and actionable fixes, markdown output.
- **Connections:**
  - **Input:** `Format Data for AI Agents`
  - **Model:** `Claude Sonnet (Specialist Agents)`
  - **Output:** `Merge Agent Outputs` (index 1)
- **Edge cases:** Very large `technical.sections` JSON may exceed model context; consider truncation.

#### Node: Link Building Agent
- **Type/role:** LangChain Agent node
- **Prompt construction:**
  - Uses backlink totals, dofollow ratio, **domain_inlink_rank (not provided by formatter)**, top anchors, and competitors slice.
- **Connections:** output to `Merge Agent Outputs` (index 2)
- **Edge cases:**
  - Undefined `domain_inlink_rank` can confuse output; add this metric or remove from prompt.
  - Anchor list empty: prompt still works but less useful.

#### Node: Keyword Strategy Agent
- **Type/role:** LangChain Agent node
- **Prompt construction:** keyword totals, traffic/value, position distribution, top 20 keywords, competitors.
- **Connections:** output to `Merge Agent Outputs` (index 3)
- **Edge cases:** If topKeywords list is empty, distribution will be zeros; agent should still produce generic strategy.

#### Node: AI Visibility Agent
- **Type/role:** LangChain Agent node
- **Prompt construction:** per-engine brand/link presence, leaderboard list, competitor stats, business description.
- **Connections:** output to `Merge Agent Outputs` (index 0)
- **Edge cases:** If AI leaderboard call failed/empty, stats default to zeros; analysis may be generic.

---

### Block 6 — Merge Specialist Outputs + Strategy Director
**Overview:** Collects the 4 specialist markdown outputs into a single object and sends them to a Strategy Director agent (Claude Opus) to create the unified executive strategy and 90‑day plan.  
**Nodes involved:** `Merge Agent Outputs`, `Compile Agent Reports`, `Claude Opus (Strategy Director)`, `Strategy Director Agent`

#### Node: Merge Agent Outputs
- **Type/role:** `merge` (4 inputs)
- **Configuration:** `numberInputs: 4`
- **Inputs:** AI Visibility, Technical, Link Building, Keyword Strategy agents (indices 0–3 as wired)
- **Output:** to `Compile Agent Reports`
- **Edge cases:** If any agent fails, merge may block or omit output depending on merge defaults.

#### Node: Compile Agent Reports
- **Type/role:** `code` (normalize agent outputs)
- **Logic:**
  - Extracts each agent’s output using fallback order: `.json.output || .json.text || ''`
  - Merges `agent_reports` into the original formatted dataset
- **Connections:** to `Strategy Director Agent`
- **Edge cases:** If an agent node returns a different response shape, reports may be empty.

#### Node: Claude Opus (Strategy Director)
- **Type/role:** Anthropic chat model connector
- **Configuration:** Model: `claude-opus-4-6`
- **Connections:** Provides `ai_languageModel` to `Strategy Director Agent`
- **Failure modes:** same as other Anthropic connector; higher cost/latency.

#### Node: Strategy Director Agent
- **Type/role:** LangChain Agent node (final synthesis)
- **Prompt construction highlights:**
  - Forces recommendations/titles to use current year derived from `report_date`.
  - Includes current metrics and AI share of voice (from leaderboard entry where `is_you`).
  - Injects all four specialist reports as raw markdown.
- **Output:** to `Compile Final Report`
- **Edge cases:**
  - If leaderboard entry for `is_you` is missing, share of voice falls back to 0 in prompt.
  - Large combined reports may approach model context limits.

---

### Block 7 — Report Compilation & Exports
**Overview:** Builds a final markdown report plus a “metrics-only” JSON row, then exports to Google Sheets and Google Drive.  
**Nodes involved:** `Compile Final Report`, `Save to Google Sheets`, `Save report to Google Drive`

#### Node: Compile Final Report
- **Type/role:** `code` (report builder)
- **Key actions:**
  - Pulls `formattedData` and `Strategy Director Agent` output.
  - Computes:
    - Total AI brand mentions and link citations (sum across engines)
    - AI share of voice and rank
    - Keyword distribution totals (top3/top10/top20)
    - Backlink “health” label based on dofollow ratio
  - Generates markdown with:
    - Executive summary (director output)
    - Section summaries (technical, backlinks, keywords, AI visibility, competitors)
    - Appendix with `<details>` blocks containing full agent outputs
  - Returns **one item** containing:
    - `file_name`, `file_content` for Drive
    - many scalar metrics for Sheets
- **Connections:** outputs to both export nodes.
- **Edge cases / failures:**
  - Competitor table assumes `formattedData.competitors` is an array; if empty/undefined, `.map` would throw (currently it assumes presence).
  - If Strategy Director output is missing fields, it falls back to placeholders.

#### Node: Save to Google Sheets
- **Type/role:** `googleSheets` (appendOrUpdate metrics row)
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `domain`
  - Mapping mode: “define below” (explicit column mapping)
- **Credentials:** `googleSheetsOAuth2Api`
- **Important configuration issues (likely bugs):**
  - Multiple fields map incorrectly to `traffic_value`:
    - `cwv_mobile` and `cwv_desktop` are mapped to `{{ $json.traffic_value }}` (should be `{{ $json.cwv_mobile }}` / `{{ $json.cwv_desktop }}`)
    - `organic_traffic` mapped to `{{ $json.traffic_value }}` (should be `{{ $json.organic_traffic }}`)
    - `technical_warnings` mapped to `{{ $json.traffic_value }}` (should be `{{ $json.technical_warnings }}`)
- **Failure modes:**
  - DocumentId not set (currently empty); node will fail until a spreadsheet is selected.
  - OAuth permission issues or missing sheet gid/name.

#### Node: Save report to Google Drive
- **Type/role:** `googleDrive` (create file from text)
- **Configuration choices:**
  - Operation: `createFromText`
  - File name/content from `Compile Final Report`
  - Drive: “My Drive”, folder: root
- **Credentials:** `googleDriveOAuth2Api`
- **Failure modes:** OAuth scope issues, insufficient permissions, invalid folder ID.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Workflow documentation & requirements |  |  | ## Run a multi-agent SEO domain audit with SE Ranking and Claude… (includes links to npm + SE Ranking API dashboard) |
| Domain Input Form | Form Trigger | User entry point (domain/market/competitors) |  | Set Configuration |  |
| Set Configuration | Set | Normalize inputs + build config JSON | Domain Input Form | Get Domain Overview; Get Top Keywords; Get Competitors; Get Backlinks Summary; Get Backlinks Anchors; Create standard audit; Prepare competitors for AI Leaderboard |  |
| Data Collection Phase | Sticky Note | Documentation for data collection block |  |  | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get Domain Overview | SE Ranking (community) | Domain organic/paid overview metrics | Set Configuration | Merge All Data | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get Top Keywords | SE Ranking (community) | Fetch top 100 keywords by volume | Set Configuration | Merge All Data | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get Competitors | SE Ranking (community) | Fetch organic competitors | Set Configuration | Merge All Data | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get Backlinks Summary | SE Ranking (community) | Backlink totals/ref domains breakdown | Set Configuration | Merge All Data | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get Backlinks Anchors | SE Ranking (community) | Top anchor texts | Set Configuration | Merge All Data | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Create standard audit | SE Ranking (community) | Start website audit crawl | Set Configuration | Wait for audit (1 min) | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Wait for audit (1 min) | Wait | Delay for audit polling | Create standard audit; If audit still running (true) | Get Audit Report | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get Audit Report | SE Ranking (community) | Retrieve audit report by audit ID | Wait for audit (1 min) | If audit still running | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| If audit still running | IF | Loop until audit completes | Get Audit Report | Wait for audit (1 min) (true); Merge All Data (false) | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Prepare competitors for AI Leaderboard | Code | Parse comma-separated competitors into objects | Set Configuration | Get AI Search Leaderboard | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Get AI Search Leaderboard | SE Ranking (community) | AI visibility leaderboard | Prepare competitors for AI Leaderboard | Merge All Data | ## Data Collection Pulls domain overview, keywords, backlinks, competitors, audit, and AI visibility data from SE Ranking |
| Merge All Data | Merge | Aggregate 7 data sources | Get Domain Overview; Get Top Keywords; Get Competitors; Get Backlinks Summary; Get Backlinks Anchors; Get AI Search Leaderboard; If audit still running (false) | Format Data for AI Agents |  |
| Format Data for AI Agents | Code | Build single structured dataset for LLM prompts | Merge All Data | Technical SEO Agent; Link Building Agent; Keyword Strategy Agent; AI Visibility Agent |  |
| AI Agents Phase | Sticky Note | Documentation for AI analysis block |  |  | ## AI Analysis (5 Agents) Four specialists analyze the data, then a Strategy Director builds the final action plan |
| Claude Sonnet (Specialist Agents) | Anthropic Chat Model | LLM model for specialist agents |  | Technical SEO Agent; Link Building Agent; Keyword Strategy Agent; AI Visibility Agent (as ai_languageModel) | ## AI Analysis (5 Agents) Four specialists analyze the data, then a Strategy Director builds the final action plan |
| AI Visibility Agent | LangChain Agent | AI search visibility analysis | Format Data for AI Agents + Claude Sonnet | Merge Agent Outputs | ## AI Analysis (5 Agents) Four specialists analyze the data, then a Strategy Director builds the final action plan |
| Technical SEO Agent | LangChain Agent | Technical audit analysis | Format Data for AI Agents + Claude Sonnet | Merge Agent Outputs | ## AI Analysis (5 Agents) Four specialists analyze the data, then a Strategy Director builds the final action plan |
| Link Building Agent | LangChain Agent | Backlink + anchor + gap analysis | Format Data for AI Agents + Claude Sonnet | Merge Agent Outputs | ## AI Analysis (5 Agents) Four specialists analyze the data, then a Strategy Director builds the final action plan |
| Keyword Strategy Agent | LangChain Agent | Keyword opportunities & content gaps | Format Data for AI Agents + Claude Sonnet | Merge Agent Outputs | ## AI Analysis (5 Agents) Four specialists analyze the data, then a Strategy Director builds the final action plan |
| Merge Agent Outputs | Merge | Combine 4 specialist outputs | AI Visibility Agent; Technical SEO Agent; Link Building Agent; Keyword Strategy Agent | Compile Agent Reports |  |
| Compile Agent Reports | Code | Normalize agent outputs into `agent_reports` | Merge Agent Outputs | Strategy Director Agent |  |
| Claude Opus (Strategy Director) | Anthropic Chat Model | LLM model for final synthesis |  | Strategy Director Agent (ai_languageModel) |  |
| Strategy Director Agent | LangChain Agent | Executive synthesis + 90-day plan | Compile Agent Reports + Claude Opus | Compile Final Report |  |
| Output Phase | Sticky Note | Documentation for export block |  |  | ## Report & Export Compiles the full report and saves it to Google Drive and Google Sheets |
| Compile Final Report | Code | Build markdown report + metrics payload | Strategy Director Agent | Save to Google Sheets; Save report to Google Drive | ## Report & Export Compiles the full report and saves it to Google Drive and Google Sheets |
| Save to Google Sheets | Google Sheets | Append/update metrics row | Compile Final Report |  | ## Report & Export Compiles the full report and saves it to Google Drive and Google Sheets |
| Save report to Google Drive | Google Drive | Save full markdown report file | Compile Final Report |  | ## Report & Export Compiles the full report and saves it to Google Drive and Google Sheets |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (self-hosted recommended due to community node + runtime).
2. **Add a Form Trigger node** named **“Domain Input Form”**  
   - Title: *AI SEO Strategy Agent*  
   - Description: *Enter a domain to generate a comprehensive SEO strategy report*  
   - Fields:
     - Domain (required)
     - Business Description (textarea, optional)
     - Target Market (dropdown, required): `us, uk, de, fr, es, it, au, ca`
     - Main Competitors (comma-separated) (optional)
3. **Add a Set node** named **“Set Configuration”** (Mode: *Raw JSON Output*) and set JSON to:
   - Build fields: `domain`, `business_description`, `source`, `competitors`, `report_date`, `ai_engines`
   - Ensure `domain` strips protocol + `www.` and removes path segments.
4. **Connect:** `Domain Input Form → Set Configuration`.

### Data collection (parallel)
5. **Install SE Ranking community node** on your n8n instance:  
   https://www.npmjs.com/package/@seranking/n8n-nodes-seranking
6. **Create SE Ranking credentials** (API token) in n8n and select it on all SE Ranking nodes.

7. Add SE Ranking nodes (all connected from **Set Configuration**):
   - **Get Domain Overview** (SE Ranking node)  
     - Domain: `{{$json.domain}}`
   - **Get Top Keywords**  
     - Operation: `getKeywords`  
     - Domain: from `Set Configuration`  
     - Source: from `Set Configuration`  
     - Limit 100, order by volume desc
   - **Get Competitors**  
     - Operation: `getCompetitors`  
     - Domain + Source from config
   - **Get Backlinks Summary**  
     - Resource: `backlinks`, target = domain
   - **Get Backlinks Anchors**  
     - Resource: `backlinks`, Operation: `getAnchors`, limit 50

8. **Website audit polling chain**
   1. Add **Create standard audit** (SE Ranking)  
      - Resource: `websiteAudit`, Operation: `createStandard`, Domain from config
   2. Add **Wait** node “Wait for audit (1 min)” set to 1 minute
   3. Add **Get Audit Report** (SE Ranking)  
      - Resource: `websiteAudit`, Operation: `getReport`  
      - Audit ID: reference the `Create standard audit` node’s returned `id`
   4. Add **IF** node “If audit still running”  
      - Condition: `is_finished` equals `false`
   5. Wire the loop:  
      - `Create standard audit → Wait → Get Audit Report → If`  
      - **True** branch of IF → back to `Wait`  
      - **False** branch continues forward (to merge)

9. **AI visibility leaderboard chain**
   1. Add **Code** node “Prepare competitors for AI Leaderboard”  
      - Parse `competitors` string into `competitorValues: [{target, brand}, …]`
      - (Recommended hardening: ensure at least 3 competitors, or pad with blanks / skip the leaderboard call if insufficient.)
   2. Add SE Ranking node “Get AI Search Leaderboard”  
      - Resource: `aiSearch`, Operation: `getLeaderboard`  
      - Source from config  
      - Engines: chatgpt, ai-overview, perplexity, gemini  
      - Primary target/brand derived from your domain  
      - Competitors: map competitorValues[0..2] to three competitor slots
   3. Connect: `Set Configuration → Prepare competitors… → Get AI Search Leaderboard`.

10. **Merge all data**
   - Add **Merge** node “Merge All Data” with **7 inputs**.
   - Connect each dataset into the appropriate merge input:
     - Overview, keywords, competitors, backlinks summary, anchors, AI leaderboard, audit report (from IF false branch).

### Formatting + agents
11. Add **Code** node “Format Data for AI Agents”
   - Pull data from the named nodes and build a single `formattedData` object.
12. Connect: `Merge All Data → Format Data for AI Agents`.

13. **Add Anthropic model credentials** in n8n (Anthropic API key).

14. Add **Anthropic Chat Model** node “Claude Sonnet (Specialist Agents)”
   - Model: `claude-sonnet-4-5-20250929`

15. Add 4 **LangChain Agent** nodes:
   - “Technical SEO Agent”
   - “Link Building Agent”
   - “Keyword Strategy Agent”
   - “AI Visibility Agent”
   For each:
   - Set prompt to “Define”
   - Provide the corresponding system message and prompt template variables referencing `$json`
   - Connect `Format Data for AI Agents` into each agent
   - Connect `Claude Sonnet` into each agent via **ai_languageModel**

16. Add **Merge** node “Merge Agent Outputs” with **4 inputs**, connect the 4 agents into it.

17. Add **Code** node “Compile Agent Reports” to normalize agent outputs into `agent_reports`, connect `Merge Agent Outputs → Compile Agent Reports`.

18. Add **Anthropic Chat Model** node “Claude Opus (Strategy Director)”
   - Model: `claude-opus-4-6`

19. Add **LangChain Agent** node “Strategy Director Agent”
   - Prompt includes: current-year constraint, metrics, and the 4 specialist reports.
   - Connect `Compile Agent Reports → Strategy Director Agent`
   - Connect `Claude Opus → Strategy Director Agent` via **ai_languageModel**

### Final report + exports
20. Add **Code** node “Compile Final Report”
   - Generate markdown report + computed metrics JSON.
21. Connect: `Strategy Director Agent → Compile Final Report`.

22. (Optional) **Google Drive export**
   - Create Google Drive OAuth2 credentials in n8n.
   - Add node “Save report to Google Drive”
     - Operation: createFromText
     - Name: `{{$json.file_name}}`
     - Content: `{{$json.file_content}}`
     - Choose drive/folder
   - Connect: `Compile Final Report → Save report to Google Drive`.

23. (Optional) **Google Sheets export**
   - Create Google Sheets OAuth2 credentials.
   - Add node “Save to Google Sheets”
     - Operation: appendOrUpdate
     - Select Spreadsheet (Document ID) and Sheet tab
     - Matching column: `domain`
     - Map fields from the metrics JSON
   - Connect: `Compile Final Report → Save to Google Sheets`.
   - Fix the incorrect mappings (recommended):
     - `cwv_mobile ← {{$json.cwv_mobile}}`
     - `cwv_desktop ← {{$json.cwv_desktop}}`
     - `organic_traffic ← {{$json.organic_traffic}}`
     - `technical_warnings ← {{$json.technical_warnings}}`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking community node v1.3.5+ required | https://www.npmjs.com/package/@seranking/n8n-nodes-seranking |
| SE Ranking API token required | https://online.seranking.com/admin.api.dashboard.html |
| Google Drive + Sheets outputs are optional | Configure OAuth2 credentials and pick target Drive folder / Spreadsheet |
| Polling loop can run indefinitely if an audit never finishes | Add a max retry counter or timeout safeguard |
| AI leaderboard competitors hard-require 3 entries in current design | Pad competitor list or conditionally skip node to prevent expression errors |

**Disclaimer (as provided):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.