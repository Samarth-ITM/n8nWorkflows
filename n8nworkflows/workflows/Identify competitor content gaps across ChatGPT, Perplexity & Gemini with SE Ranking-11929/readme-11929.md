Identify competitor content gaps across ChatGPT, Perplexity & Gemini with SE Ranking

https://n8nworkflows.xyz/workflows/identify-competitor-content-gaps-across-chatgpt--perplexity---gemini-with-se-ranking-11929


# Identify competitor content gaps across ChatGPT, Perplexity & Gemini with SE Ranking

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Identify competitor content gaps across ChatGPT, Perplexity & Gemini with SE Ranking  
**Workflow name (in JSON):** Identify competitor content gaps in AI search with SE Ranking

**Purpose:**  
This workflow compares your domain against a competitor to identify **AI search visibility gaps** across **ChatGPT, Perplexity, and Gemini**, then enriches those gaps with **keyword research metrics** and **backlink authority context**, producing a prioritized opportunity list exported to **Google Sheets**.

**Target use cases:**
- Marketing/SEO teams tracking AI visibility vs competitors
- Content strategists planning topics for AI citations and traditional SEO wins
- Competitive intelligence reporting (AI engines + SEO metrics)

### 1.1 Logical blocks
1. **Input & Configuration**: Manual run + define your domain, competitor, region, scope.
2. **AI Visibility Retrieval (Your domain + Competitor)**: Pull AI search summaries for ChatGPT/Perplexity/Gemini with rate limiting.
3. **AI Gap Computation**: Merge results and compute gaps + an AI opportunity score per engine.
4. **Competitor Research (Prompts, Keywords, Backlinks)**: Retrieve competitor prompts, SEO keywords, and backlink authority; extract top terms.
5. **Keyword List Unification & Keyword Metrics**: Combine prompts + keywords, de-duplicate/clean/filter, then fetch full keyword research metrics.
6. **Final Scoring & Export**: Create a unified opportunity report (AI gaps + keyword gaps), rank/prioritize, export to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Configuration
**Overview:** Starts the workflow manually and sets the core parameters used by all downstream SE Ranking calls (domains, region database, scope).

**Nodes involved:**
- Manual Trigger
- Configuration

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point.
- **Configuration choices:** No parameters.
- **Outputs:** Sends a single empty item to **Configuration**.
- **Edge cases:** None (only runs when manually executed).

#### Node: Configuration
- **Type / role:** `n8n-nodes-base.set` — defines runtime constants.
- **Configuration choices (interpreted):**
  - Sets:
    - `your_domain`: `seranking.com`
    - `competitor_domain`: `semrush.com`
    - `source`: `us` (SE Ranking regional database)
    - `scope`: `base_domain` (domain analysis scope)
- **Key variables referenced elsewhere:**
  - `$json.your_domain`, `$json.competitor_domain`, `$json.source`, `$json.scope`
  - Many nodes use cross-node access: `$('Configuration').item.json...` or `$('Configuration').first().json...`
- **Connections:**
  - Input: Manual Trigger
  - Output: **Your Domain - ChatGPT**
- **Edge cases / failure modes:**
  - If `source` is invalid for the SE Ranking API, AI/keyword calls can fail (validation/API errors).
  - If domains are malformed (e.g., include protocol), SE Ranking responses may be empty or error.

---

### Block 2 — AI Visibility Retrieval (Your domain + Competitor) with Rate Limiting
**Overview:** Pulls AI search visibility summaries for both domains across three engines. Wait nodes throttle calls to reduce API rate-limit risks.

**Nodes involved:**
- Your Domain - ChatGPT
- Wait (Rate Limit)
- Your Domain - Perplexity
- Wait (Rate Limit)1
- Your Domain - Gemini
- Wait (Rate Limit)2
- Competitor - ChatGPT
- Wait (Rate Limit)3
- Competitor - Perplexity
- Wait (Rate Limit)4
- Competitor - Gemini
- Merge AI Visibility Data

#### Node: Your Domain - ChatGPT
- **Type / role:** `@seranking/n8n-nodes-seranking.seRanking` — SE Ranking community node, AI Search resource.
- **Configuration choices:**
  - `resource`: `aiSearch`
  - `domain`: from `Configuration.your_domain`
  - `scope`: from `Configuration.scope` (via `$json.scope`, since Configuration directly connects here)
  - `source`: from `Configuration.source`
  - Engine default implies **ChatGPT** (since no `engine` is set in this node).
- **Connections:**
  - Input: Configuration
  - Output: Wait (Rate Limit)
- **Edge cases:**
  - SE Ranking auth/token invalid → 401/403.
  - API quota/rate limit → 429.
  - Empty/missing `summary` in response; later code uses `?.summary || {}` to soften this.

#### Node: Wait (Rate Limit)
- **Type / role:** `n8n-nodes-base.wait` — throttling.
- **Configuration choices:** waits `3` (seconds by default).
- **Connections:** Your Domain - ChatGPT → Wait → Your Domain - Perplexity
- **Edge cases:** Increases runtime; if n8n execution timeouts are strict, long chains may fail.

#### Node: Your Domain - Perplexity
- **Type / role:** SE Ranking node — AI Search resource for Perplexity engine.
- **Configuration choices:**
  - `resource`: `aiSearch`
  - `engine`: `perplexity`
  - Uses `$('Configuration').item.json...` expressions for scope/domain/source.
- **Connections:** Wait (Rate Limit) → Your Domain - Perplexity → Wait (Rate Limit)1
- **Edge cases:** Same as other SE Ranking calls.

#### Node: Wait (Rate Limit)1
- **Type / role:** Wait throttling
- **Configuration:** `amount: 3`
- **Connections:** Your Domain - Perplexity → Wait → Your Domain - Gemini

#### Node: Your Domain - Gemini
- **Type / role:** SE Ranking node — AI Search resource for Gemini engine.
- **Configuration:** `engine: gemini`, plus domain/scope/source from Configuration.
- **Connections:** Your Domain - Gemini → Wait (Rate Limit)2

#### Node: Wait (Rate Limit)2
- **Type / role:** Wait throttling
- **Configuration:** `amount: 3`
- **Connections:** Wait → Competitor - ChatGPT

#### Node: Competitor - ChatGPT
- **Type / role:** SE Ranking node — AI Search resource for competitor (ChatGPT default).
- **Configuration:** domain = competitor_domain; scope/source from Configuration.
- **Connections:** Wait (Rate Limit)2 → Competitor - ChatGPT → Wait (Rate Limit)3

#### Node: Wait (Rate Limit)3
- **Type / role:** Wait throttling
- **Configuration:** `amount: 3`
- **Connections:** Wait → Competitor - Perplexity

#### Node: Competitor - Perplexity
- **Type / role:** SE Ranking node — AI Search resource for Perplexity engine.
- **Configuration:** `engine: perplexity`, competitor domain.
- **Connections:**
  - Output 1 → Merge AI Visibility Data (input index 0)
  - Output 2 → Wait (Rate Limit)4
- **Edge cases:** Dual outputs increase dependency complexity; if Merge expects positional pairing but one branch is delayed/fails, merge may behave unexpectedly.

#### Node: Wait (Rate Limit)4
- **Type / role:** Wait throttling
- **Configuration:** `amount: 3`
- **Connections:** Wait → Competitor - Gemini

#### Node: Competitor - Gemini
- **Type / role:** SE Ranking node — AI Search resource for Gemini engine.
- **Connections:** Competitor - Gemini → Merge AI Visibility Data (input index 1)

#### Node: Merge AI Visibility Data
- **Type / role:** `n8n-nodes-base.merge` — combines two inputs.
- **Configuration choices:**
  - `mode: combine`
  - `combinationMode: mergeByPosition`
- **Inputs:** From Competitor - Perplexity (index 0) and Competitor - Gemini (index 1).  
  Note: although it’s named “Merge AI Visibility Data”, it does **not** merge all six calls; the subsequent code node directly reads the other nodes by name.
- **Output:** Calculate AI Gaps
- **Edge cases:**
  - `mergeByPosition` assumes aligned item ordering; if one input produces different item counts, output can be misaligned.
  - Because later logic uses `$('...').first()` from named nodes, this merge is effectively a “synchronization” step; if either input is empty due to API error, execution may still proceed but later scoring may be skewed.

---

### Block 3 — AI Gap Computation
**Overview:** Computes per-engine gaps (link presence, average position, AI opportunity traffic) and assigns an opportunity score and priority.

**Nodes involved:**
- Calculate AI Gaps
- Competitor Backlink Authority (triggered from this block as well)
- Wait (Rate Limit)6 (used to pace prompt fetching)

#### Node: Calculate AI Gaps
- **Type / role:** `n8n-nodes-base.code` — transforms SE Ranking summaries into gap records.
- **Key logic (interpreted):**
  - Reads config via `$('Configuration').first().json`
  - Reads outputs directly from:
    - `Your Domain - ChatGPT/Perplexity/Gemini`
    - `Competitor - ChatGPT/Perplexity/Gemini`
  - Extracts from each response: `summary.link_presence.current`, `summary.average_position.current`, `summary.ai_opportunity_traffic.current`
  - Gap calculations:
    - `link_presence_gap = your - competitor` (negative means competitor better)
    - `avgPosGap = compAvgPos - yourAvgPos` (intended so negative means competitor better, but see edge case below)
    - `traffic_gap = your - competitor` (negative means competitor better)
  - Opportunity score: weighted magnitude of negative gaps
  - Priority thresholds: `>100 HIGH`, `>50 MEDIUM`, else `LOW`
  - Flags:
    - `is_competitor_winning: link_presence_gap < 0 || avgPosGap < 0`
- **Connections:**
  - Input: Merge AI Visibility Data
  - Outputs:
    - Competitor Backlink Authority
    - Wait (Rate Limit)6 → Competitor Top Prompts
- **Edge cases / potential logic bug:**
  - **Average position direction:** In SEO, *lower average position is better*. The code comment says “flip”, but uses `avgPosGap = compAvgPos - yourAvgPos`.  
    Example: your=10, competitor=5 (competitor better). `comp - your = -5` → competitor winning (negative) matches flag logic.  
    Example: your=5, competitor=10 (you better). `comp - your = +5` → not winning.  
    So the sign is consistent with “negative means competitor better”. The inline comment “Lower is better, so flip” is misleading, but the implementation works for “competitor better => negative”.
  - `.toFixed(2)` on `yourAvgPos`/`compAvgPos` assumes numbers; defaults to `0` so safe.
  - If SE Ranking returns strings/null, math may coerce unexpectedly.

#### Node: Wait (Rate Limit)6
- **Type / role:** Wait throttling
- **Configuration:** empty `parameters` (so defaults apply; in n8n Wait node, that may mean “wait indefinitely for resume” depending on mode/version). Here it’s **configured as a normal Wait node v1.1**, but **without `amount`** it may not delay as intended.
- **Connections:** Calculate AI Gaps → Wait (Rate Limit)6 → Competitor Top Prompts
- **Edge cases:**
  - If this defaults to a “resume via webhook” style wait, the workflow may pause unexpectedly.
  - Consider setting an explicit `amount` like other wait nodes.

---

### Block 4 — Competitor Research (Prompts, Keywords, Backlinks)
**Overview:** Pulls competitor prompts used in AI search, competitor SEO keywords, and backlink authority, then extracts top terms for downstream keyword research.

**Nodes involved:**
- Competitor Top Prompts
- Extract Top Prompts
- Wait (Rate Limit)5
- Competitor Top Keywords
- Extract Top Keywords
- Competitor Backlink Authority

#### Node: Competitor Top Prompts
- **Type / role:** SE Ranking node — AI Search prompts extraction.
- **Configuration choices:**
  - `resource: aiSearch`
  - `operation: getPromptsByTarget`
  - domain/scope/source from Configuration
  - additional fields:
    - `sort: volume`, `sortOrder: desc`, `limit: 50`
- **Connections:**
  - From Wait (Rate Limit)6
  - Outputs:
    - Extract Top Prompts
    - Wait (Rate Limit)5 → Competitor Top Keywords
- **Edge cases:**
  - Response shape assumed: `prompts` array and `total` count; if API changes, Extract Top Prompts may output empty keywords.

#### Node: Extract Top Prompts
- **Type / role:** Code node — formats top prompts into a keyword list.
- **Key logic:**
  - Uses `$input.first().json.prompts || []`
  - Filters for prompt presence, slices to top 20 prompts, returns:
    - `keywords` (array of prompt strings)
    - `count`
    - `original_total`
- **Connections:** Competitor Top Prompts → Extract Top Prompts → Merge All Intelligence1 (input 0)
- **Edge cases:**
  - Assumes prompts already sorted by volume by API; it does not re-sort (comment suggests sort but code does not sort).
  - Prompts may contain punctuation beyond the later regex validation; they may be removed in Combine Keywords.

#### Node: Wait (Rate Limit)5
- **Type / role:** Wait throttling
- **Configuration:** `amount: 3`
- **Connections:** Competitor Top Prompts → Wait → Competitor Top Keywords

#### Node: Competitor Top Keywords
- **Type / role:** SE Ranking node — retrieves competitor organic keywords.
- **Configuration choices:**
  - `operation: getKeywords`
  - `domain`: competitor domain
  - `source`: config source
  - additional fields:
    - `limit: 100`
    - sort by `traffic` descending
    - keyword position filter: `positionFrom: 1`, `positionTo: 20`
    - volume filter: `volumeFrom: 500`
- **Connections:** Wait (Rate Limit)5 → Competitor Top Keywords → Extract Top Keywords
- **Edge cases:** Large domains may still hit API limits or return fewer than expected.

#### Node: Extract Top Keywords
- **Type / role:** Code node — selects a smaller list of high-value competitor keywords.
- **Key logic:**
  - Reads all items: `$input.all()`
  - Filters: `volume > 500` and `position <= 10`
  - Takes top 30 and returns `keywords` array + `count`
- **Connections:** Competitor Top Keywords → Extract Top Keywords → Merge All Intelligence1 (input 1)
- **Edge cases:**
  - Assumes each item has `volume`, `position`, `keyword`.
  - If SE Ranking returns nested fields, mapping may fail silently (resulting in empty list).

#### Node: Competitor Backlink Authority
- **Type / role:** SE Ranking node — backlink authority context.
- **Configuration choices:**
  - `resource: backlinks`
  - `target`: competitor domain
  - `historical: true`
- **Connections:**
  - From Calculate AI Gaps
  - Output: Merge All Intelligence (input 1)
- **Edge cases:**
  - Response assumed to contain `backlinks_info.backlinks/refdomains/domain_inlink_rank`. Missing fields default to 0 later.

---

### Block 5 — Keyword List Unification & Keyword Metrics
**Overview:** Combines prompt-derived keywords and competitor SEO keywords, cleans them (including an explicit “inappropriate terms” filter), limits the list, then fetches full keyword metrics from SE Ranking.

**Nodes involved:**
- Merge All Intelligence1
- Combine Keywords
- Gap Keywords - Full Metrics

#### Node: Merge All Intelligence1
- **Type / role:** Merge node — combines extracted prompts + extracted keywords.
- **Configuration choices:** Defaults (JSON shows empty `parameters`), but node typeVersion `2.1`. In n8n, default merge behavior is typically **append** or **combine** depending on node defaults; here it is used as a 2-input aggregator into the next Code node.
- **Connections:**
  - Input 0: Extract Top Prompts
  - Input 1: Extract Top Keywords
  - Output: Combine Keywords
- **Edge cases:**
  - If defaults are not “combine”, the downstream code still uses `$input.all()` so it will work as long as it receives both items somewhere in the input stream.

#### Node: Combine Keywords
- **Type / role:** Code node — unifies keyword lists and enforces hygiene constraints.
- **Key logic:**
  - Collects all `item.json.keywords` arrays
  - De-duplicates case-insensitively: `kw.toLowerCase().trim()`
  - Filters out a hardcoded list of inappropriate/adult terms
  - Validates keywords:
    - non-empty, `<200` chars
    - regex: `^[a-zA-Z0-9\\s\\-\\?\\.]+$` (only alphanumeric, spaces, hyphens, question marks, periods)
  - Outputs:
    - `keywords`: first 50 valid keywords
    - counts: `total_count`, `filtered_out`, `prompts_count`, `competitor_keywords_count`
- **Connections:** Combine Keywords → Gap Keywords - Full Metrics
- **Edge cases:**
  - **Regex is strict**: will drop keywords containing commas, apostrophes, non-Latin characters, accents (é), slashes, ampersands, etc. This can remove many legitimate queries depending on locale.
  - If `allInputs[0]` / `[1]` ordering changes, `prompts_count` and `competitor_keywords_count` metadata can be wrong (not fatal).

#### Node: Gap Keywords - Full Metrics
- **Type / role:** SE Ranking node — keyword research enrichment.
- **Configuration choices:**
  - `resource: keywordResearch`
  - `source`: from Configuration
  - `keywords`: **hardcoded to `seo`** (expression is `=seo`)
  - additional fields:
    - `cols`: `keyword`, `volume`, `cpc`, `competition`, `difficulty`
    - sort by `volume desc`
- **Connections:** Gap Keywords - Full Metrics → Merge All Intelligence (input 0)
- **Critical issue (likely misconfiguration):**
  - The node should probably use the output from **Combine Keywords** (i.e., the cleaned list), but currently it always requests metrics for the single keyword `"seo"`.
  - Expected fix in n8n expression field: set `keywords` to something like:
    - `={{ $json.keywords }}` (if node accepts array), or
    - `={{ $json.keywords.join(',') }}` (if it expects comma-separated string),
    depending on the community node’s input format.
- **Edge cases:**
  - Keyword research endpoints often have limits (batch sizes, rate limits). The workflow already limits to 50 keywords.

---

### Block 6 — Final Scoring & Export
**Overview:** Combines AI gap items, keyword metrics items, and backlink context into a single ranked opportunity report and writes it to Google Sheets.

**Nodes involved:**
- Merge All Intelligence
- Final Opportunity Scoring
- Export to Google Sheets

#### Node: Merge All Intelligence
- **Type / role:** Merge node — pairs keyword metrics with backlink authority context.
- **Configuration choices:**
  - `mode: combine`
  - `combinationMode: mergeByPosition`
- **Connections:**
  - Input 0: Gap Keywords - Full Metrics
  - Input 1: Competitor Backlink Authority
  - Output: Final Opportunity Scoring
- **Edge cases:**
  - `mergeByPosition` can misalign if keyword metrics returns multiple items and backlinks returns a single item. In practice, the next code node does **not rely on merged fields**; it reads other nodes directly by name (`$('Gap Keywords - Full Metrics').all()` and `$('Competitor Backlink Authority').first()`), so this merge mostly acts as a sequencing step.

#### Node: Final Opportunity Scoring
- **Type / role:** Code node — generates final prioritized list.
- **Key logic (interpreted):**
  - Reads:
    - `aiGaps` from `Calculate AI Gaps`
    - `keywordMetrics` from `Gap Keywords - Full Metrics`
    - `backlinkData` from `Competitor Backlink Authority`
    - `config` from `Configuration`
  - Builds `compAuthority` from `backlinks_info`
  - Creates opportunities of two types:
    - `AI_VISIBILITY_GAP`: only when `is_competitor_winning` is true; action depends on link_presence_gap
    - `KEYWORD_GAP`: only when `volume > 1000` and `difficulty < 70`
      - keyword opportunity score = `volume*0.5 + (100-difficulty)*0.3 + cpc*10`
      - priority thresholds: `>5000 HIGH`, `>2000 MEDIUM`, else `LOW`
  - Sorts by `opportunity_score`, returns top 50 with metadata.
- **Connections:** Merge All Intelligence → Final Opportunity Scoring → Export to Google Sheets
- **Edge cases:**
  - If Gap Keywords - Full Metrics returns items without numeric `volume/difficulty/cpc`, scoring can produce `NaN` and break sorting.
  - Because Gap Keywords - Full Metrics is currently hardcoded to `seo`, the output may not reflect competitor gaps until corrected.

#### Node: Export to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — persistence/export.
- **Configuration choices:**
  - `operation`: `appendOrUpdate`
  - `documentId`: Google Sheet URL (provided)
  - `sheetName`: `Sheet1` (gid=0)
  - `mappingMode`: auto-map input data
- **Credentials:** `googleSheetsOAuth2Api`
- **Edge cases / failure modes:**
  - OAuth misconfiguration → auth errors.
  - If “appendOrUpdate” requires a key column but none is configured, behavior may default to append only or fail (depends on node/version).
  - Auto-mapping can create inconsistent columns if fields vary between opportunity types (AI vs Keyword). Consider normalizing columns.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n Sticky Note | Workflow description & requirements |  |  | ## 🔍 Identify competitor content gaps in AI search with SE Ranking \n**Who is this for:** Marketing teams, content strategists, SEO teams. \n**Requirements:** Self-hosted n8n, SE Ranking community node: https://www.npmjs.com/package/@seranking/n8n-nodes-seranking, SE Ranking API token: https://online.seranking.com/admin.api.dashboard.html, Google Sheets (optional). \n**Setup:** install node, add creds, update domains, connect Sheets. |
| Sticky Note1 | n8n Sticky Note | Step label |  |  | ### Step 1: Configuration — Set domain, competitor, region. Change `source` (us, uk, de, fr, es, it, etc.). |
| Sticky Note2 | n8n Sticky Note | Step label |  |  | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Sticky Note3 | n8n Sticky Note | Step label |  |  | ### Step 3: Calculate AI Gaps — Compare visibility metrics and calculate opportunity scores. |
| Sticky Note4 | n8n Sticky Note | Step label |  |  | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Sticky Note5 | n8n Sticky Note | Step label |  |  | ### Step 5: Final Scoring & Export — Prioritize opportunities and export to Google Sheets. |
| Manual Trigger | Manual Trigger | Entry point | — | Configuration |  |
| Configuration | Set | Defines domains/source/scope | Manual Trigger | Your Domain - ChatGPT | ### Step 1: Configuration — Set domain, competitor, region. Change `source` (us, uk, de, fr, es, it, etc.). |
| Your Domain - ChatGPT | SE Ranking (community) | AI search summary (ChatGPT default) for your domain | Configuration | Wait (Rate Limit) | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Wait (Rate Limit) | Wait | Throttle API requests | Your Domain - ChatGPT | Your Domain - Perplexity | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Your Domain - Perplexity | SE Ranking (community) | AI search summary (Perplexity) for your domain | Wait (Rate Limit) | Wait (Rate Limit)1 | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Wait (Rate Limit)1 | Wait | Throttle API requests | Your Domain - Perplexity | Your Domain - Gemini | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Your Domain - Gemini | SE Ranking (community) | AI search summary (Gemini) for your domain | Wait (Rate Limit)1 | Wait (Rate Limit)2 | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Wait (Rate Limit)2 | Wait | Throttle API requests | Your Domain - Gemini | Competitor - ChatGPT | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Competitor - ChatGPT | SE Ranking (community) | AI search summary (ChatGPT default) for competitor | Wait (Rate Limit)2 | Wait (Rate Limit)3 | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Wait (Rate Limit)3 | Wait | Throttle API requests | Competitor - ChatGPT | Competitor - Perplexity | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Competitor - Perplexity | SE Ranking (community) | AI search summary (Perplexity) for competitor | Wait (Rate Limit)3 | Merge AI Visibility Data; Wait (Rate Limit)4 | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Wait (Rate Limit)4 | Wait | Throttle API requests | Competitor - Perplexity | Competitor - Gemini | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Competitor - Gemini | SE Ranking (community) | AI search summary (Gemini) for competitor | Wait (Rate Limit)4 | Merge AI Visibility Data | ### Step 2: AI Visibility Data — Fetch AI search metrics across ChatGPT, Perplexity, Gemini. |
| Merge AI Visibility Data | Merge | Combine/synchronize AI visibility branch | Competitor - Perplexity; Competitor - Gemini | Calculate AI Gaps | ### Step 3: Calculate AI Gaps — Compare visibility metrics and calculate opportunity scores. |
| Calculate AI Gaps | Code | Compute gaps & opportunity score per AI engine | Merge AI Visibility Data | Competitor Backlink Authority; Wait (Rate Limit)6 | ### Step 3: Calculate AI Gaps — Compare visibility metrics and calculate opportunity scores. |
| Competitor Backlink Authority | SE Ranking (community) | Backlink authority context | Calculate AI Gaps | Merge All Intelligence | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Wait (Rate Limit)6 | Wait | Throttle before prompt fetch (but amount not set) | Calculate AI Gaps | Competitor Top Prompts | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Competitor Top Prompts | SE Ranking (community) | Fetch competitor AI prompts by target | Wait (Rate Limit)6 | Extract Top Prompts; Wait (Rate Limit)5 | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Extract Top Prompts | Code | Select top prompts and format as keywords array | Competitor Top Prompts | Merge All Intelligence1 | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Wait (Rate Limit)5 | Wait | Throttle API requests | Competitor Top Prompts | Competitor Top Keywords | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Competitor Top Keywords | SE Ranking (community) | Fetch competitor ranking keywords | Wait (Rate Limit)5 | Extract Top Keywords | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Extract Top Keywords | Code | Select high-value competitor keywords | Competitor Top Keywords | Merge All Intelligence1 | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Merge All Intelligence1 | Merge | Combine prompts + keywords lists | Extract Top Prompts; Extract Top Keywords | Combine Keywords | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Combine Keywords | Code | De-duplicate/clean/filter keywords and limit to 50 | Merge All Intelligence1 | Gap Keywords - Full Metrics | ### Step 4: Keyword & Backlink Research — Extract competitor's top prompts, keywords, backlink authority. |
| Gap Keywords - Full Metrics | SE Ranking (community) | Enrich keyword list with metrics (volume/cpc/difficulty) | Combine Keywords | Merge All Intelligence | ### Step 5: Final Scoring & Export — Prioritize opportunities and export to Google Sheets. |
| Merge All Intelligence | Merge | Combine/synchronize keyword metrics + backlinks | Gap Keywords - Full Metrics; Competitor Backlink Authority | Final Opportunity Scoring | ### Step 5: Final Scoring & Export — Prioritize opportunities and export to Google Sheets. |
| Final Opportunity Scoring | Code | Build final ranked opportunity report | Merge All Intelligence | Export to Google Sheets | ### Step 5: Final Scoring & Export — Prioritize opportunities and export to Google Sheets. |
| Export to Google Sheets | Google Sheets | Append/update rows in Sheet1 | Final Opportunity Scoring | — | ### Step 5: Final Scoring & Export — Prioritize opportunities and export to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Identify competitor content gaps in AI search with SE Ranking*.

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - No configuration needed.

3. **Add node: Configuration**
   - Node type: **Set**
   - Add fields:
     - `your_domain` (string) e.g. `seranking.com`
     - `competitor_domain` (string) e.g. `semrush.com`
     - `source` (string) e.g. `us`
     - `scope` (string) e.g. `base_domain`
   - Connect: **Manual Trigger → Configuration**

4. **Install and configure SE Ranking community node**
   - Install package on your self-hosted n8n:
     - https://www.npmjs.com/package/@seranking/n8n-nodes-seranking
   - Create SE Ranking API token:
     - https://online.seranking.com/admin.api.dashboard.html
   - In n8n **Credentials**, add **SE Ranking API** credential and paste the token.

5. **Add node: Your Domain - ChatGPT**
   - Node type: **SE Ranking** (community)
   - Resource: **aiSearch**
   - Domain: `={{ $json.your_domain }}`
   - Scope: `={{ $json.scope }}`
   - Source: `={{ $json.source }}`
   - Engine: leave default (ChatGPT)
   - Credentials: select your SE Ranking credential
   - Connect: **Configuration → Your Domain - ChatGPT**

6. **Add throttling + remaining “Your domain” AI nodes**
   - Add **Wait** node (“Wait (Rate Limit)”), set **Amount = 3 seconds**
     - Connect: Your Domain - ChatGPT → Wait (Rate Limit)
   - Add **SE Ranking** node (“Your Domain - Perplexity”)
     - Resource: aiSearch, Engine: `perplexity`
     - Domain: `={{ $('Configuration').item.json.your_domain }}`
     - Scope: `={{ $('Configuration').item.json.scope }}`
     - Source: `={{ $('Configuration').item.json.source }}`
     - Connect: Wait (Rate Limit) → Your Domain - Perplexity
   - Add **Wait** node (“Wait (Rate Limit)1”), Amount = 3
     - Connect: Your Domain - Perplexity → Wait (Rate Limit)1
   - Add **SE Ranking** node (“Your Domain - Gemini”)
     - Resource: aiSearch, Engine: `gemini`
     - Domain/scope/source from Configuration (same pattern)
     - Connect: Wait (Rate Limit)1 → Your Domain - Gemini

7. **Add competitor AI nodes with waits**
   - Add **Wait** node (“Wait (Rate Limit)2”), Amount = 3
     - Connect: Your Domain - Gemini → Wait (Rate Limit)2
   - Add **SE Ranking** node (“Competitor - ChatGPT”)
     - Resource: aiSearch (ChatGPT default)
     - Domain: `={{ $('Configuration').item.json.competitor_domain }}`
     - Scope/source from Configuration
     - Connect: Wait (Rate Limit)2 → Competitor - ChatGPT
   - Add **Wait** node (“Wait (Rate Limit)3”), Amount = 3
     - Connect: Competitor - ChatGPT → Wait (Rate Limit)3
   - Add **SE Ranking** node (“Competitor - Perplexity”)
     - Resource: aiSearch, Engine: `perplexity`
     - Domain: competitor_domain, plus scope/source
     - Connect: Wait (Rate Limit)3 → Competitor - Perplexity
   - Add **Wait** node (“Wait (Rate Limit)4”), Amount = 3
     - Connect: Competitor - Perplexity → Wait (Rate Limit)4
   - Add **SE Ranking** node (“Competitor - Gemini”)
     - Resource: aiSearch, Engine: `gemini`
     - Domain: competitor_domain, plus scope/source
     - Connect: Wait (Rate Limit)4 → Competitor - Gemini

8. **Add node: Merge AI Visibility Data**
   - Node type: **Merge**
   - Mode: **Combine**
   - Combination mode: **Merge by position**
   - Connect:
     - Competitor - Perplexity → Merge AI Visibility Data (Input 1)
     - Competitor - Gemini → Merge AI Visibility Data (Input 2)

9. **Add node: Calculate AI Gaps (Code)**
   - Node type: **Code**
   - Paste the gap calculation logic (adapt from workflow) that reads:
     - `$('Your Domain - ChatGPT/Perplexity/Gemini').first().json`
     - `$('Competitor - ChatGPT/Perplexity/Gemini').first().json`
   - Connect: Merge AI Visibility Data → Calculate AI Gaps

10. **Add node: Competitor Backlink Authority**
    - Node type: **SE Ranking**
    - Resource: **backlinks**
    - Target: `={{ $('Configuration').item.json.competitor_domain }}`
    - Additional field: `historical = true`
    - Connect: Calculate AI Gaps → Competitor Backlink Authority

11. **Add node: Wait (Rate Limit)6**
    - Node type: **Wait**
    - Set **Amount = 3 seconds** (recommended; the provided workflow leaves it blank)
    - Connect: Calculate AI Gaps → Wait (Rate Limit)6

12. **Add node: Competitor Top Prompts**
    - Node type: **SE Ranking**
    - Resource: **aiSearch**
    - Operation: **getPromptsByTarget**
    - Domain/scope/source: from Configuration
    - Additional fields: sort=volume, sortOrder=desc, limit=50
    - Connect: Wait (Rate Limit)6 → Competitor Top Prompts

13. **Add node: Extract Top Prompts (Code)**
    - Node type: **Code**
    - Extract `prompts[]` into `keywords[]` (top 20)
    - Connect: Competitor Top Prompts → Extract Top Prompts

14. **Add wait and competitor keyword retrieval**
    - Add **Wait (Rate Limit)5** Amount = 3
      - Connect: Competitor Top Prompts → Wait (Rate Limit)5
    - Add **Competitor Top Keywords** (SE Ranking)
      - Operation: `getKeywords`
      - Domain: competitor_domain
      - Source: config source
      - Additional filters: limit 100, orderField traffic desc, position 1–20, volumeFrom 500
      - Connect: Wait (Rate Limit)5 → Competitor Top Keywords
    - Add **Extract Top Keywords** (Code)
      - Filter volume>500, position<=10, top 30, output `keywords[]`
      - Connect: Competitor Top Keywords → Extract Top Keywords

15. **Add node: Merge All Intelligence1**
    - Node type: **Merge**
    - Connect:
      - Extract Top Prompts → Merge All Intelligence1 (Input 1)
      - Extract Top Keywords → Merge All Intelligence1 (Input 2)

16. **Add node: Combine Keywords (Code)**
    - Node type: **Code**
    - Combine all `keywords[]`, de-duplicate, filter inappropriate terms, validate with regex, output first 50.
    - Connect: Merge All Intelligence1 → Combine Keywords

17. **Add node: Gap Keywords - Full Metrics**
    - Node type: **SE Ranking**
    - Resource: **keywordResearch**
    - Source: `={{ $('Configuration').first().json.source }}`
    - Additional fields:
      - cols: keyword, volume, cpc, competition, difficulty
      - sort by volume desc
    - **Important:** Set `keywords` to the output of Combine Keywords (not a hardcoded literal). Use one of:
      - `={{ $json.keywords }}` (if node accepts arrays), or
      - `={{ $json.keywords.join(',') }}` (if node expects a string list).
    - Connect: Combine Keywords → Gap Keywords - Full Metrics

18. **Add node: Merge All Intelligence**
    - Node type: **Merge**
    - Mode: Combine, Merge by position
    - Connect:
      - Gap Keywords - Full Metrics → Merge All Intelligence (Input 1)
      - Competitor Backlink Authority → Merge All Intelligence (Input 2)

19. **Add node: Final Opportunity Scoring (Code)**
    - Node type: **Code**
    - Build opportunities from:
      - `$('Calculate AI Gaps').all()`
      - `$('Gap Keywords - Full Metrics').all()`
      - `$('Competitor Backlink Authority').first().json`
    - Sort by score, output top 50 rows.
    - Connect: Merge All Intelligence → Final Opportunity Scoring

20. **Add node: Export to Google Sheets**
    - Node type: **Google Sheets**
    - Credentials: **Google Sheets OAuth2**
    - Operation: **appendOrUpdate**
    - Select Spreadsheet + Sheet (e.g., Sheet1)
    - Mapping: **Auto-map input data**
    - Connect: Final Opportunity Scoring → Export to Google Sheets

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking community node package | https://www.npmjs.com/package/@seranking/n8n-nodes-seranking |
| SE Ranking API token creation | https://online.seranking.com/admin.api.dashboard.html |
| Workflow includes explicit keyword hygiene filtering | Combine Keywords node removes an “inappropriate terms” list and enforces a strict regex; adjust for non-English locales or punctuation. |
| Potential misconfiguration to fix | “Gap Keywords - Full Metrics” uses keywords set to `seo` instead of using Combine Keywords output. |
| Google Sheet referenced in node configuration | https://docs.google.com/spreadsheets/d/1ouYDOeYe3tuNCEglNyWSh4e1gktN3ExpcbLl46xQ7Xc/edit#gid=0 |

