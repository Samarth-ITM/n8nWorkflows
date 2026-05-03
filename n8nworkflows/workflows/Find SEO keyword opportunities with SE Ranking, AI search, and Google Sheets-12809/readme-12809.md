Find SEO keyword opportunities with SE Ranking, AI search, and Google Sheets

https://n8nworkflows.xyz/workflows/find-seo-keyword-opportunities-with-se-ranking--ai-search--and-google-sheets-12809


# Find SEO keyword opportunities with SE Ranking, AI search, and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Identify SEO keyword opportunities by combining SE Ranking competitor discovery + keyword gap analysis + lost keywords + topic expansion (similar/related keywords) + AI search visibility (ChatGPT/Perplexity/Gemini/AI Overview), then consolidate and export the prioritized opportunities into Google Sheets.

**Primary use cases:**
- SEO agencies monitoring competitor movements and finding keyword gaps
- Content teams building clusters and expanding topics
- Marketing teams planning quick wins and growth opportunities

### 1.1 Entry & Configuration
Manual execution initializes a configuration object (domain, brand, country, filters, competitors list, AI engines).

### 1.2 Competitor Discovery & Keyword Gap Scoring
Find top organic competitors from SE Ranking, extract their domains, fetch keyword gaps vs your site, filter by volume/difficulty, and compute an “opportunity_score”.

### 1.3 Topic Expansion Loop (Similar/Related Keywords)
From the top gap keywords, loop through them and request similar + related keywords, then format them as “TOPIC_EXPANSION” opportunities.

### 1.4 Lost Keywords (Quick Wins)
Fetch keywords recently lost (rankings dropped out) and format them into high-priority recovery opportunities.

### 1.5 AI Search Visibility (Leaderboard)
Query SE Ranking AI search leaderboard and extract visibility metrics for your domain across multiple AI engines.

### 1.6 Consolidation, Final Ranking & Export
Merge baseline domain metrics + opportunities + AI visibility, enrich each row with context (domain totals + AI metrics), sort by opportunity score, and append/update rows in Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Entry & Configuration
**Overview:** Starts the workflow manually and defines all global parameters used via expressions in later nodes.

**Nodes involved:**
- When clicking 'Execute workflow'
- Configuration
- Overview (sticky note)

#### Node: **When clicking 'Execute workflow'**
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — workflow entry point.
- **Config choices:** No parameters; user runs workflow manually.
- **Outputs:** Sends one empty item to **Configuration**.
- **Failure modes:** None (except workflow not executed).

#### Node: **Configuration**
- **Type / role:** Set node (`n8n-nodes-base.set`) — creates a single JSON config object.
- **Config choices (interpreted):**
  - `your_domain`: target domain to analyze (e.g., `seranking.com`)
  - `your_brand`: brand name (used in AI leaderboard request)
  - `source`: country/locale (e.g., `us`)
  - `currency`: informational (not used elsewhere in the JSON shown)
  - `competitor_count`: number of competitors to keep from discovery (5)
  - `min_volume` / `max_difficulty`: filters for gaps and expansions
  - `known_competitors`: list of competitor targets + brands (used in AI leaderboard node; currently only index `[2]` is referenced)
  - `ai_engines`: engine list (note: the AI leaderboard node uses a hardcoded engines list instead of this field)
- **Key expressions used elsewhere:** Many nodes reference `$('Configuration').item.json...`
- **Outputs / connections:** Feeds 4 branches in parallel:
  - Get your domain overview
  - Auto-discover top competitors
  - Wait before lost keywords
  - Wait before AI leaderboard
- **Failure modes / edge cases:**
  - Invalid `source` code can cause SE Ranking API errors.
  - If `known_competitors` has fewer than 3 entries, AI leaderboard node referencing `[2]` breaks.
- **Version notes:** Set node v3.4 supports “raw” JSON output mode as used here.

#### Sticky note: **Overview**
- **Role:** Embedded documentation for setup and usage.
- **Content preserved:** Explains audience, output, steps, setup requirements (SE Ranking node v1.3.5+), customization (min_volume/max_difficulty).

---

### Block 2.2 — Baseline Domain Overview
**Overview:** Fetches high-level organic metrics for your domain (keywords count, traffic sum, value sum) for later context in final scoring.

**Nodes involved:**
- Get your domain overview
- Sticky Note (Domain Overview)

#### Node: **Get your domain overview**
- **Type / role:** SE Ranking node (`@seranking/n8n-nodes-seranking.seRanking`) — domain overview API call.
- **Config choices:**
  - `domain` from config: `={{ $('Configuration').item.json.your_domain }}`
  - Operation: default domain overview operation (operation not explicitly set, so relies on node’s default for this resource/version).
- **Credentials:** `seRankingApi` required.
- **Outputs / connections:** To **Merge domain & topics** (input 0).
- **Failure modes / edge cases:**
  - Credential/auth failure (invalid token).
  - Domain not found / no data in SE Ranking project.
  - API rate limits.
  - Output schema changes: later code expects `item.json.organic[0].keywords_count`, `traffic_sum`, `price_sum`.
- **Version-specific:** Requires SE Ranking community node version compatible with this operation (sticky note mentions v1.3.5+).

#### Sticky note: **Domain Overview**
Content: “Get your baseline metrics (keywords, traffic, value)”.

---

### Block 2.3 — Competitor Discovery & Keyword Gap Analysis
**Overview:** Discovers competitors, extracts competitor domains, performs keyword comparison (gaps), filters results using thresholds, and computes an opportunity score per keyword.

**Nodes involved:**
- Auto-discover top competitors
- Extract competitor domains
- Wait before gap analysis
- Get keyword gaps
- Filter & score keyword gaps
- Extract top gap keywords
- Sticky Note1 (Competitor Discovery & Gap Analysis)

#### Node: **Auto-discover top competitors**
- **Type / role:** SE Ranking node — retrieves organic competitors.
- **Config choices:**
  - `operation`: `getCompetitors`
  - `domain`: config `your_domain`
  - `source`: config `source`
- **Outputs / connections:** To **Extract competitor domains**.
- **Failure modes:**
  - API rate limits / timeouts.
  - No competitors returned for small/new sites.

#### Node: **Extract competitor domains**
- **Type / role:** Code node — normalizes competitor list and caps it to `competitor_count`.
- **Logic highlights:**
  - Takes `$input.all()` competitors.
  - Uses `$('Configuration').first().json` for settings.
  - Builds items: `{ rank, competitor_domain, common_keywords, your_domain, source }`
  - **Fallback behavior:** If nothing returned, emits a default competitor `semrush.com`.
- **Outputs / connections:** To **Wait before gap analysis**.
- **Edge cases / failures:**
  - If upstream competitor items don’t have `.json.domain`, output competitor_domain becomes `undefined` (later API call will fail).
  - Fallback competitor may not be relevant; still allows the workflow to proceed.

#### Node: **Wait before gap analysis**
- **Type / role:** Wait node — throttling / spacing requests (2 units; unit depends on n8n Wait node default, typically seconds unless configured).
- **Role:** Reduce risk of SE Ranking rate limit.
- **Outputs / connections:** To **Get keyword gaps**.
- **Failure modes:** Execution delay; if n8n instance restarts, Wait behavior depends on n8n execution persistence settings.

#### Node: **Get keyword gaps**
- **Type / role:** SE Ranking node — keyword comparison between your domain and competitor domain.
- **Config choices:**
  - `operation`: `getKeywordsComparison`
  - `domain`: `={{ $json.your_domain }}`
  - `compareDomain`: `={{ $json.competitor_domain }}`
- **Outputs / connections:** To **Filter & score keyword gaps**.
- **Failure modes:**
  - Invalid competitor domain.
  - API limit / pagination (if large lists and node default page size is limited).

#### Node: **Filter & score keyword gaps**
- **Type / role:** Code node — filters by volume/difficulty, computes opportunity score, tags rows as `KEYWORD_GAP`.
- **Key expressions / variables:**
  - `config = $('Configuration').first().json`
  - `currentCompetitor = $('Extract competitor domains').item.json` (important: depends on correct item pairing during execution)
  - Uses `$('Wait before gap analysis').first().json.source` as `source`
- **Scoring formula:**
  - `volumeScore = volume * 0.4`
  - `difficultyScore = (100 - difficulty) * 0.3`
  - `trafficScore = traffic * 0.3` (or 0)
  - `opportunity_score = round(sum)`
  - `priority`: HIGH if >5000, MEDIUM if >2000 else LOW
- **Outputs / connections:** To **Extract top gap keywords**.
- **Edge cases / failures:**
  - If `k.volume` or `k.difficulty` are missing/non-numeric, filter/scoring may behave unexpectedly.
  - `currentCompetitor` resolution can be tricky when multiple competitor items are processed; if execution context doesn’t align, competitor_domain may be mismatched. (In n8n, referencing another node’s `.item` relies on item linking; if not preserved, use `.first()` or pass competitor_domain through the chain explicitly.)
  - Returns empty array if nothing passes filters; downstream topic expansion will be skipped by design.

#### Node: **Extract top gap keywords**
- **Type / role:** Code node — selects top 10 gap keywords and outputs them as separate items for batching.
- **Logic highlights:**
  - Reads from `$('Filter & score keyword gaps').all()` (not from `$input.all()`).
  - Sorts by `opportunity_score`, keeps top 10.
  - Outputs items: `{ source, keyword }` where `source` is taken from `$input.first().json.source`.
- **Outputs / connections:** To **Loop through keywords**.
- **Edge cases:**
  - If `$input` is empty but gaps exist in referenced node, `$input.first()` would fail; however this node is connected directly after Filter node, so normally `$input` matches.
  - If no gaps, returns `[]` and topic expansion branch effectively stops.

#### Sticky note: **Competitor Discovery & Gap Analysis**
Content: “Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords”.

---

### Block 2.4 — Topic Expansion (Similar + Related Keyword Research Loop)
**Overview:** Loops through selected gap keywords, requests similar and related suggestions from SE Ranking Keyword Research, then formats them into “TOPIC_EXPANSION” opportunity rows.

**Nodes involved:**
- Loop through keywords
- Get similar keywords
- Get related keywords
- Merge similar & related
- Format topic expansion

#### Node: **Loop through keywords**
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — processes keyword items in controlled batches.
- **Config choices:** Defaults (batch size default in n8n is typically 1 unless set).
- **Connections:**
  - Output 0 → **Format topic expansion** (this is unusual ordering; see note below)
  - Output 0 → **Get similar keywords** and **Get related keywords**
  - Input from **Extract top gap keywords**
  - Loop-back input from **Merge similar & related**
- **Important behavior note:** The standard pattern is: SplitInBatches → API calls → Merge → back to SplitInBatches. Here it also connects directly to **Format topic expansion**; however **Format topic expansion** actually pulls data from `$('Get similar keywords').all()` and `$('Get related keywords').all()`, so the direct connection mainly triggers execution order rather than providing data.
- **Failure modes / edge cases:**
  - If batch size too high, may hit API limits.
  - If it receives no items, nothing runs downstream.

#### Node: **Get similar keywords**
- **Type / role:** SE Ranking node — keyword research “similar” suggestions.
- **Config choices:**
  - `resource`: `keywordResearch`
  - `operation`: `getSimilar`
  - `source`: `={{ $json.source }}`
  - `keyword`: `={{ $json.keyword }}`
  - `limit`: 10
- **Outputs:** To **Merge similar & related** (input 0).
- **Failure modes:** keyword not supported / API errors / rate limits.

#### Node: **Get related keywords**
- **Type / role:** SE Ranking node — keyword research “related” suggestions.
- **Config choices:** same as similar, with `operation: getRelated`, limit 10.
- **Outputs:** To **Merge similar & related** (input 1).

#### Node: **Merge similar & related**
- **Type / role:** Merge node (`n8n-nodes-base.merge`) combining two lists.
- **Config choices:**
  - `mode`: combine
  - `combinationMode`: `mergeByPosition`
- **Outputs / connections:** Sends merged output back into **Loop through keywords** (to continue loop).
- **Edge cases:**
  - If one side returns fewer items, merge-by-position can create partial merges; but downstream “Format topic expansion” does not use this merged payload anyway (it reads directly from the two API nodes).

#### Node: **Format topic expansion**
- **Type / role:** Code node — converts nested keyword arrays to flat opportunity rows.
- **Logic highlights:**
  - Pulls: `similar = $('Get similar keywords').all()` and `related = $('Get related keywords').all()`
  - Extracts `item.json.keywords` arrays (expects SE Ranking response shape)
  - Concatenates, filters `volume >= config.min_volume`
  - Scoring: `(volume * 0.5) + ((100 - difficulty) * 0.5)`
  - Output fields: `type: TOPIC_EXPANSION`, `estimated_traffic = volume*0.12`, action “Expand: Add to content cluster”
  - Fallback items when no keywords exist or none pass filters.
- **Outputs / connections:** To **Merge domain & topics** (input 1).
- **Failure modes / edge cases:**
  - If SE Ranking response structure differs (e.g., keywords not under `json.keywords`), it will output “NO_EXPANSION_DATA”.
  - Difficulty may be missing; defaults to 50.
  - Potential duplication: similar + related lists may overlap; no deduplication is performed.

---

### Block 2.5 — Lost Keywords (Recovery Opportunities)
**Overview:** Fetches keywords with “lost” position change and formats them as high-priority items.

**Nodes involved:**
- Wait before lost keywords
- Get your lost keywords
- Format lost keywords
- Sticky Note2 (Lost Keywords)

#### Node: **Wait before lost keywords**
- **Type / role:** Wait node — throttling before SE Ranking keyword call.
- **Config:** amount 2.
- **Outputs:** To **Get your lost keywords**.

#### Node: **Get your lost keywords**
- **Type / role:** SE Ranking node — fetch domain keywords with filters.
- **Config choices:**
  - `operation`: `getKeywords`
  - `domain`, `source` from config
  - Additional fields:
    - `limit: 50`
    - `posChange: lost` (lost rankings)
    - sort: `orderField: volume`, `orderType: desc`
    - `volumeFrom`: config `min_volume`
- **Outputs:** To **Format lost keywords**.
- **Failure modes:** API auth, rate limits, no data.

#### Node: **Format lost keywords**
- **Type / role:** Code node — selects top 30 and formats “LOST_KEYWORD” opportunities.
- **Scoring:**
  - `opportunity_score = round(volume*0.6 + (100-difficulty)*0.4)`
  - `priority` forced to HIGH
  - `action`: quick win if difficulty < 40 else “Recover: Update content”
- **Outputs:** To **Merge lost & AI** (input 0).
- **Edge cases:**
  - `prev_position` may not exist; defaults to `N/A`.
  - `difficulty` may be missing; defaults to 50.

#### Sticky note: **Lost Keywords**
Content: “Find quick wins from keywords you recently lost rankings for”.

---

### Block 2.6 — AI Search Visibility (Leaderboard Extraction)
**Overview:** Calls SE Ranking AI Search leaderboard and extracts your domain’s visibility metrics across AI engines.

**Nodes involved:**
- Wait before AI leaderboard
- Get AI search leaderboard
- Extract your AI visibility
- Sticky Note3 (AI Search Visibility)

#### Node: **Wait before AI leaderboard**
- **Type / role:** Wait node — throttling before AI search call.
- **Config:** amount 2.
- **Outputs:** To **Get AI search leaderboard**.

#### Node: **Get AI search leaderboard**
- **Type / role:** SE Ranking node — AI Search leaderboard query.
- **Config choices (as set):**
  - `resource`: `aiSearch`
  - `operation`: `getLeaderboard`
  - `source`: config `source`
  - `engines`: hardcoded `["chatgpt","ai-overview","perplexity","gemini"]`
  - `primaryBrand`: config `your_brand`
  - `primaryTarget`: config `your_domain`
  - `competitors`: **only one competitor is configured**:
    - brand/domain pulled from `known_competitors[2]` (3rd element)
- **Outputs:** To **Extract your AI visibility**.
- **Failure modes / edge cases:**
  - If `known_competitors[2]` is missing, expression fails.
  - If API expects multiple competitors, the dataset may be too limited.
  - Engine naming must match API (`ai-overview` vs `ai_overview` etc.); this workflow expects `ai-overview`.

#### Node: **Extract your AI visibility**
- **Type / role:** Code node — parses leaderboard result structure and produces a single metrics item.
- **Key assumptions:**
  - Response has `leaderboard.results[your_domain]`
  - For each engine: `link_presence` and `brand_presence`
- **Outputs:** To **Merge lost & AI** (input 1).
- **Edge cases:**
  - If leaderboard structure changes or domain key not present, metrics default to 0.

#### Sticky note: **AI Search Visibility**
Content: “Track presence across ChatGPT, Perplexity, Gemini, AI Overview”.

---

### Block 2.7 — Merging, Final Scoring, and Export
**Overview:** Merges baseline domain metrics + topic expansion outputs + lost keywords + AI metrics, then enriches and sorts all opportunities into a final ranked dataset exported to Google Sheets.

**Nodes involved:**
- Merge domain & topics
- Merge lost & AI
- Merge all data
- Final scoring with AI context
- Export to Google Sheets

#### Node: **Merge domain & topics**
- **Type / role:** Merge node — combines:
  - Input 0: domain overview
  - Input 1: topic expansion items
- **Config:** default merge behavior (no explicit mode shown in parameters).
- **Outputs:** To **Merge all data** (input 0).
- **Edge cases:**
  - Default merge mode can affect item pairing/row counts. If it merges by index, mismatched counts can drop data or produce unexpected merges. Here it is mainly used to pass both datasets downstream for later separation.

#### Node: **Merge lost & AI**
- **Type / role:** Merge node — combines:
  - Input 0: lost keyword opportunities
  - Input 1: AI visibility metrics
- **Outputs:** To **Merge all data** (input 1).
- **Edge cases:** Same as above regarding merge semantics.

#### Node: **Merge all data**
- **Type / role:** Merge node — combines:
  - Input 0: domain+topics stream
  - Input 1: lost+AI stream
- **Outputs:** To **Final scoring with AI context**.

#### Node: **Final scoring with AI context**
- **Type / role:** Code node — validates presence of baseline + AI data, extracts your domain metrics, sorts opportunities, and enriches each row with context fields.
- **Input classification logic:**
  - `domainData = items where item.json.organic && item.json._domain`
  - `aiData = items where item.json.chatgpt_presence !== undefined`
  - `opportunities = items where item.json.keyword && item.json.type`
- **Hard requirements enforced:**
  - Throws error if no `domainData` or no `aiData`.
- **Extracted baseline metrics (expects this structure):**
  - `organic[0].keywords_count`
  - `organic[0].traffic_sum`
  - `organic[0].price_sum`
- **Output:**
  - Sorts opportunities by `opportunity_score` DESC
  - Adds:
    - `rank`, `analysis_date`
    - `your_*` baseline metrics
    - `your_*` AI metrics
    - then spreads the opportunity fields (`...opp`)
- **Failure modes / edge cases:**
  - If domain overview response lacks `_domain` or `organic`, `domainData` filter may not match and will throw “No domain data”.
  - If AI node doesn’t run or returns unexpected schema, throws “No AI data”.
  - If some opportunities don’t have numeric `opportunity_score`, sorting may be inconsistent.

#### Node: **Export to Google Sheets**
- **Type / role:** Google Sheets node — writes final ranked rows.
- **Config choices:**
  - `operation`: `appendOrUpdate`
  - `documentId`: not set in JSON (must be selected)
  - `sheetName`: not set in JSON (must be selected)
- **Credentials:** Google Sheets OAuth2 (or service account) required.
- **Failure modes:**
  - Missing document/sheet selection.
  - Auth errors / insufficient permissions.
  - Column mismatch: appendOrUpdate requires key mapping and headers alignment (not configured here; user must configure the matching columns/keys in node UI).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Embedded workflow description & setup notes | — | — | ## Find competitor keyword opportunities with SE Ranking and AI search… (includes setup steps, requires SE Ranking node v1.3.5+) |
| When clicking 'Execute workflow' | Manual Trigger | Entry point | — | Configuration |  |
| Configuration | Set | Global parameters (domain, filters, competitors, engines) | When clicking 'Execute workflow' | Get your domain overview; Auto-discover top competitors; Wait before lost keywords; Wait before AI leaderboard |  |
| Get your domain overview | SE Ranking | Fetch baseline domain metrics | Configuration | Merge domain & topics | ### Domain Overview<br>Get your baseline metrics (keywords, traffic, value) |
| Auto-discover top competitors | SE Ranking | Fetch organic competitors | Configuration | Extract competitor domains | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Extract competitor domains | Code | Normalize/cap competitor list + fallback | Auto-discover top competitors | Wait before gap analysis | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Wait before gap analysis | Wait | Throttle before gap API calls | Extract competitor domains | Get keyword gaps | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Get keyword gaps | SE Ranking | Keyword comparison vs competitor | Wait before gap analysis | Filter & score keyword gaps | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Filter & score keyword gaps | Code | Filter by volume/difficulty + score | Get keyword gaps | Extract top gap keywords | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Extract top gap keywords | Code | Select top 10 gap keywords for looping | Filter & score keyword gaps | Loop through keywords | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Loop through keywords | Split In Batches | Iterate through top gap keywords | Extract top gap keywords; Merge similar & related | Format topic expansion; Get similar keywords; Get related keywords | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Get similar keywords | SE Ranking | Keyword research (similar) | Loop through keywords | Merge similar & related | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Get related keywords | SE Ranking | Keyword research (related) | Loop through keywords | Merge similar & related | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Merge similar & related | Merge | Join similar+related streams and continue loop | Get similar keywords; Get related keywords | Loop through keywords | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Format topic expansion | Code | Flatten suggestions + score as TOPIC_EXPANSION | Loop through keywords | Merge domain & topics | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Merge domain & topics | Merge | Combine domain overview + topic opportunities | Get your domain overview; Format topic expansion | Merge all data |  |
| Wait before lost keywords | Wait | Throttle before lost keyword fetch | Configuration | Get your lost keywords | ### Lost Keywords<br>Find quick wins from keywords you recently lost rankings for |
| Get your lost keywords | SE Ranking | Fetch lost keywords | Wait before lost keywords | Format lost keywords | ### Lost Keywords<br>Find quick wins from keywords you recently lost rankings for |
| Format lost keywords | Code | Format + score lost keywords | Get your lost keywords | Merge lost & AI | ### Lost Keywords<br>Find quick wins from keywords you recently lost rankings for |
| Wait before AI leaderboard | Wait | Throttle before AI leaderboard fetch | Configuration | Get AI search leaderboard | ### AI Search Visibility<br>Track presence across ChatGPT, Perplexity, Gemini, AI Overview |
| Get AI search leaderboard | SE Ranking | Fetch AI visibility leaderboard | Wait before AI leaderboard | Extract your AI visibility | ### AI Search Visibility<br>Track presence across ChatGPT, Perplexity, Gemini, AI Overview |
| Extract your AI visibility | Code | Extract your domain’s AI metrics | Get AI search leaderboard | Merge lost & AI | ### AI Search Visibility<br>Track presence across ChatGPT, Perplexity, Gemini, AI Overview |
| Merge lost & AI | Merge | Combine lost opportunities + AI metrics | Format lost keywords; Extract your AI visibility | Merge all data |  |
| Merge all data | Merge | Combine all streams for final scoring | Merge domain & topics; Merge lost & AI | Final scoring with AI context |  |
| Final scoring with AI context | Code | Validate, enrich, sort and rank opportunities | Merge all data | Export to Google Sheets |  |
| Export to Google Sheets | Google Sheets | Append/update final dataset | Final scoring with AI context | — |  |
| Sticky Note | Sticky Note | Block label | — | — | ### Domain Overview<br>Get your baseline metrics (keywords, traffic, value) |
| Sticky Note1 | Sticky Note | Block label | — | — | ### Competitor Discovery & Gap Analysis<br>Auto-discover top 5 competitors, analyze keyword gaps, expand topics with related keywords |
| Sticky Note2 | Sticky Note | Block label | — | — | ### Lost Keywords<br>Find quick wins from keywords you recently lost rankings for |
| Sticky Note3 | Sticky Note | Block label | — | — | ### AI Search Visibility<br>Track presence across ChatGPT, Perplexity, Gemini, AI Overview |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Manual Trigger**
   - Node: *Manual Trigger*
   - Name: `When clicking 'Execute workflow'`

3. **Add Configuration (Set node)**
   - Node: *Set*
   - Name: `Configuration`
   - Mode: raw JSON output
   - Paste a config object with keys:
     - `your_domain`, `your_brand`, `source`, `currency`
     - `competitor_count`, `min_volume`, `max_difficulty`
     - `known_competitors` (ensure at least 3 entries if you keep `[2]` usage)
     - `ai_engines` (optional; not used by the AI leaderboard node as currently built)
   - Connect: Manual Trigger → Configuration

4. **SE Ranking credentials**
   - Install **SE Ranking node** (`@seranking/n8n-nodes-seranking`) version **v1.3.5+** as noted in the sticky note.
   - Create credentials: **SE Ranking API** in n8n (token/key per SE Ranking API).
   - Assign these credentials to every SE Ranking node below.

5. **Branch A: Domain overview**
   1) Add SE Ranking node:
      - Name: `Get your domain overview`
      - Set `domain` to expression: `$('Configuration').item.json.your_domain`
   2) Add Merge node:
      - Name: `Merge domain & topics`
   3) Connect:
      - Configuration → Get your domain overview
      - Get your domain overview → Merge domain & topics (input 0)

6. **Branch B: Competitor discovery + gap analysis**
   1) Add SE Ranking node:
      - Name: `Auto-discover top competitors`
      - Operation: `getCompetitors`
      - Domain: `$('Configuration').item.json.your_domain`
      - Source: `$('Configuration').item.json.source`
   2) Add Code node:
      - Name: `Extract competitor domains`
      - Implement logic to:
        - take first `competitor_count`
        - output `competitor_domain`, `your_domain`, `source`
        - include fallback if empty
   3) Add Wait node:
      - Name: `Wait before gap analysis`
      - Amount: 2 (seconds or default unit)
   4) Add SE Ranking node:
      - Name: `Get keyword gaps`
      - Operation: `getKeywordsComparison`
      - Domain: `{{$json.your_domain}}`
      - Compare domain: `{{$json.competitor_domain}}`
   5) Add Code node:
      - Name: `Filter & score keyword gaps`
      - Filter: `volume >= min_volume` and `difficulty <= max_difficulty`
      - Set fields including `type: KEYWORD_GAP` and compute `opportunity_score`
   6) Add Code node:
      - Name: `Extract top gap keywords`
      - Sort by `opportunity_score`, take top 10, output items `{source, keyword}`
   7) Add Split In Batches:
      - Name: `Loop through keywords`
      - Default batch size (or set explicitly to 1)
   8) Add two SE Ranking nodes:
      - `Get similar keywords` (resource `keywordResearch`, operation `getSimilar`, limit 10)
      - `Get related keywords` (resource `keywordResearch`, operation `getRelated`, limit 10)
      - Both use expressions: `source: {{$json.source}}`, `keyword: {{$json.keyword}}`
   9) Add Merge node:
      - Name: `Merge similar & related`
      - Mode: combine
      - Combination mode: merge by position
   10) Add Code node:
      - Name: `Format topic expansion`
      - Read outputs from `Get similar keywords` and `Get related keywords`
      - Flatten nested `keywords[]`
      - Output rows with `type: TOPIC_EXPANSION` and scoring
   11) Connect in order:
      - Configuration → Auto-discover top competitors
      - Auto-discover top competitors → Extract competitor domains → Wait before gap analysis → Get keyword gaps → Filter & score keyword gaps → Extract top gap keywords → Loop through keywords
      - Loop through keywords → Get similar keywords & Get related keywords
      - Get similar keywords → Merge similar & related (input 0)
      - Get related keywords → Merge similar & related (input 1)
      - Merge similar & related → Loop through keywords (to continue batches)
      - Loop through keywords → Format topic expansion (to trigger formatting each cycle)
      - Format topic expansion → Merge domain & topics (input 1)

7. **Branch C: Lost keywords**
   1) Add Wait node: `Wait before lost keywords` (amount 2)
   2) Add SE Ranking node: `Get your lost keywords`
      - Operation: `getKeywords`
      - Domain: config `your_domain`
      - Source: config `source`
      - Additional fields: limit 50, `posChange=lost`, sort by volume desc, `volumeFrom=min_volume`
   3) Add Code node: `Format lost keywords`
      - Keep top 30
      - Output rows `type: LOST_KEYWORD`, compute score/action
   4) Add Merge node: `Merge lost & AI`
   5) Connect:
      - Configuration → Wait before lost keywords → Get your lost keywords → Format lost keywords → Merge lost & AI (input 0)

8. **Branch D: AI leaderboard**
   1) Add Wait node: `Wait before AI leaderboard` (amount 2)
   2) Add SE Ranking node: `Get AI search leaderboard`
      - Resource: `aiSearch`
      - Operation: `getLeaderboard`
      - Source: config `source`
      - Engines: chatgpt, ai-overview, perplexity, gemini
      - Primary brand/target: from config
      - Competitors: set at least one competitor (or map from `known_competitors`)
   3) Add Code node: `Extract your AI visibility`
      - Extract `results[your_domain][engine].link_presence` and `.brand_presence`
      - Output single item with those metrics
   4) Connect:
      - Configuration → Wait before AI leaderboard → Get AI search leaderboard → Extract your AI visibility → Merge lost & AI (input 1)

9. **Final consolidation & export**
   1) Add Merge node: `Merge all data`
      - Connect:
        - Merge domain & topics → Merge all data (input 0)
        - Merge lost & AI → Merge all data (input 1)
   2) Add Code node: `Final scoring with AI context`
      - Validate domain overview + AI metrics exist
      - Extract baseline metrics from domain overview response
      - Collect all `type + keyword` opportunity rows, sort by `opportunity_score`
      - Enrich with `your_*` baseline + AI fields + rank/date
   3) Add Google Sheets node: `Export to Google Sheets`
      - Operation: `appendOrUpdate`
      - Select **Document** and **Sheet**
      - Configure matching columns / key field behavior in the node UI (required for appendOrUpdate)
      - Set up Google credentials (OAuth2 or service account)
   4) Connect:
      - Merge all data → Final scoring with AI context → Export to Google Sheets

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install SE Ranking node v1.3.5+ and add SE Ranking API credentials | Mentioned in the workflow’s “Overview” sticky note |
| Configure `your_domain`, `your_brand`, `source`, `min_volume`, `max_difficulty` in the Configuration node | Mentioned in the workflow’s “Overview” sticky note |
| Google Sheets export is optional but requires document + sheet selection and valid Google credentials | Implied by the Google Sheets node configuration (IDs not set) |