Track AI search winners and uncover topic gaps with SE Ranking and Google Sheets

https://n8nworkflows.xyz/workflows/track-ai-search-winners-and-uncover-topic-gaps-with-se-ranking-and-google-sheets-14257


# Track AI search winners and uncover topic gaps with SE Ranking and Google Sheets

# 1. Workflow Overview

This workflow collects AI search visibility and SEO gap data for one primary domain and up to two competitors, then exports the results into Google Sheets. It is designed for SEO teams, content strategists, and marketing managers who want to compare AI search presence across major engines and uncover missing organic and AI-topic opportunities.

It is triggered by an n8n form, uses the SE Ranking community node for all SEO and AI-search data retrieval, applies JavaScript formatting and filtering in Code nodes, and appends each dataset to separate Google Sheets tabs.

## 1.1 Input Reception and Normalization

The workflow starts with a public form where the user provides their domain, brand, seed topic, competitors, and target market. A Set node then normalizes the submission into a consistent configuration object used throughout the workflow.

## 1.2 AI Search Leaderboard

This block retrieves AI search leaderboard data across multiple AI engines and formats it into a row-based structure suitable for spreadsheet export.

## 1.3 Organic Keyword Gap Analysis

This branch compares the user’s domain against two competitors using SE Ranking keyword comparison data, merges both competitor datasets, filters and deduplicates keywords, and exports the strongest missing opportunities.

## 1.4 Question Keyword Discovery

This block retrieves question-style keywords related to the seed topic, filtered for informational intent and a minimum search volume.

## 1.5 Your Existing AI Topic Footprint

This branch fetches prompts where the user’s domain already appears in AI search results and formats them for reporting.

## 1.6 Competitor AI Topic Gap Detection

This branch fetches AI prompts associated with both competitor brands, merges them with the user’s AI-topic footprint, filters for SEO-relevant prompts, and identifies prompts where competitors appear but the user does not.

## 1.7 Data Export and Form Sharing

Each branch appends results to Google Sheets. The workflow is intended to be activated so the form trigger exposes a production URL that can be shared or embedded.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Normalization

**Overview:**  
This block captures user input from a hosted form and converts it into a normalized internal configuration object. It also handles fallback logic so the second competitor defaults to the first if the user leaves it blank.

**Nodes Involved:**  
- Domain Input Form
- Configuration

### Domain Input Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`; entry point that exposes a hosted form and triggers the workflow on submission.
- **Configuration choices:**
  - Form title: `AI Search Gap Analysis`
  - Description explains the purpose clearly
  - Required fields:
    - Your Domain
    - Your Brand Name
    - Seed Topic
    - Competitor 1 Domain
    - Competitor 1 Brand
    - Target Market
  - Optional fields:
    - Competitor 2 Domain
    - Competitor 2 Brand
  - Target Market is a dropdown with: `us, uk, de, fr, es, it, au, ca, pl`
- **Key expressions or variables used:** None in this node; it emits submitted form fields as JSON.
- **Input and output connections:**  
  - Input: none, this is the workflow trigger  
  - Output: Configuration
- **Version-specific requirements:** `typeVersion 2.2`; requires n8n support for Form Trigger nodes.
- **Edge cases or potential failure types:**
  - Workflow inactive: no production form URL available
  - Invalid user values such as malformed domains
  - Empty optional competitor 2 fields
- **Sub-workflow reference:** None

### Configuration
- **Type and technical role:** `n8n-nodes-base.set`; standardizes form data into canonical variables used by all downstream nodes.
- **Configuration choices:**
  - Creates fields:
    - `your_domain`
    - `your_brand`
    - `competitor_domain_1`
    - `competitor_brand_1`
    - `competitor_domain_2`
    - `competitor_brand_2`
    - `seed_topic`
    - `source`
    - `scope`
    - `volume_min`
    - `results_limit`
  - Defaults:
    - `scope = base_domain`
    - `volume_min = 500`
    - `results_limit = 200`
  - Fallback logic:
    - If competitor 2 is blank, competitor 2 domain defaults to competitor 1 domain
    - If competitor 2 brand is blank, competitor 2 brand defaults to competitor 1 brand
- **Key expressions or variables used:**
  - `{{ $json['Your Domain'] }}`
  - `{{ $json['Competitor 2 Domain'] || $json['Competitor 1 Domain'] }}`
  - `{{ $json['Target Market'] }}`
- **Input and output connections:**  
  - Input: Domain Input Form  
  - Outputs:
    - Get AI leaderboard
    - Wait before keyword gap (3s)
    - Wait before questions (6s)
    - Wait before your AI topics (9s)
    - Wait before competitor topics (12s)
    - Wait before competitor topics (10s)
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases or potential failure types:**
  - Form field label changes would break expressions
  - Numeric values are stored as strings; downstream code must cast where needed
  - Duplicate competitor fallback may produce duplicate results in comparison branches
- **Sub-workflow reference:** None

---

## 2.2 AI Search Leaderboard

**Overview:**  
This block retrieves AI search leaderboard data for the primary domain and two competitors across several AI engines, then converts the response into flat rows for export.

**Nodes Involved:**  
- Get AI leaderboard
- Format leaderboard
- Export to Sheets: AI Leaderboard

### Get AI leaderboard
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; calls SE Ranking AI Search leaderboard endpoint.
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getLeaderboard`
  - Source database from `{{ $json.source }}`
  - Engines:
    - `ai-overview`
    - `chatgpt`
    - `perplexity`
    - `gemini`
  - Primary brand/domain from the normalized config
  - Two competitor brand/domain pairs from the normalized config
- **Key expressions or variables used:**
  - `{{ $json.source }}`
  - `{{ $json.your_brand }}`
  - `{{ $json.your_domain }}`
  - competitor expressions from config
- **Input and output connections:**  
  - Input: Configuration  
  - Output: Format leaderboard
- **Version-specific requirements:** `typeVersion 1`; requires the SE Ranking community node package to be installed.
- **Edge cases or potential failure types:**
  - Missing or invalid SE Ranking API credentials
  - Unsupported source database
  - API rate limits or quota issues
  - If competitor 2 is blank and falls back to competitor 1, leaderboard may compare duplicate competitors
  - Engine coverage mismatch: code expects `ai-mode` fields too, but this node only explicitly requests four engines
- **Sub-workflow reference:** None

### Format leaderboard
- **Type and technical role:** `n8n-nodes-base.code`; flattens leaderboard and per-engine metrics into spreadsheet-friendly rows.
- **Configuration choices:**
  - Reads:
    - `data.leaderboard`
    - `data.results`
  - Outputs one row per ranked domain
  - Calculates:
    - `share_of_voice_pct` as formatted percent string
    - total and per-engine brand/link presence
    - `is_your_domain`
    - current date
- **Key expressions or variables used:**
  - Uses `$input.first().json`
  - Optional chaining-like safe access through `results[item.domain]?.chatgpt?...`
- **Input and output connections:**  
  - Input: Get AI leaderboard  
  - Output: Export to Sheets: AI Leaderboard
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - Missing `leaderboard` or `results` arrays/objects produces empty output
  - `ai_mode_*` fields may stay zero if API response does not include `ai-mode`
  - If `share_of_voice` is null, it defaults to `0`
- **Sub-workflow reference:** None

### Export to Sheets: AI Leaderboard
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends leaderboard rows to a Google Sheets tab.
- **Configuration choices:**
  - Operation: `append`
  - Mapping: auto-map input data
  - Sheet selected by `gid=0`
  - Document ID expected from spreadsheet URL
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Format leaderboard  
  - Output: none
- **Version-specific requirements:** `typeVersion 4.4`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Empty `documentId` in current workflow JSON means export will fail until configured
  - Wrong sheet/tab selection
  - OAuth authorization errors
  - Column header mismatch if the target sheet is not prepared
- **Sub-workflow reference:** None

---

## 2.3 Organic Keyword Gap Analysis

**Overview:**  
This block retrieves missing keyword opportunities from two competitor comparisons, spaces the API requests with waits, merges both result sets, filters for Latin-script keywords and minimum volume, deduplicates them, and exports the top items.

**Nodes Involved:**  
- Wait before keyword gap (3s)
- Get keyword gaps 1st Competitor
- Wait before keyword gap (3s)1
- Get keyword gaps 2nd Competitor
- Merge keyword gaps
- Format keyword gaps
- Export to Sheets: Organic Gaps

### Wait before keyword gap (3s)
- **Type and technical role:** `n8n-nodes-base.wait`; delays execution before keyword gap requests.
- **Configuration choices:** waits `3` units; in this context intended as 3 seconds.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Configuration  
  - Outputs:
    - Get keyword gaps 1st Competitor
    - Wait before keyword gap (3s)1
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Wait nodes increase total execution time
  - If workflow timeout policies are strict, chained waits may matter
- **Sub-workflow reference:** None

### Get keyword gaps 1st Competitor
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; compares competitor 1 domain vs the user’s domain.
- **Configuration choices:**
  - Operation: `getKeywordsComparison`
  - `diff = 1` to retrieve differences/gaps
  - `domain = competitor_domain_1`
  - `compareDomain = your_domain`
  - Source from config
  - Additional fields:
    - limit from `results_limit`
    - order by `volume desc`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_domain_1 }}`
  - `{{ $('Configuration').item.json.your_domain }}`
  - `{{ $('Configuration').item.json.source }}`
- **Input and output connections:**  
  - Input: Wait before keyword gap (3s)  
  - Output: Merge keyword gaps
- **Version-specific requirements:** SE Ranking community node required.
- **Edge cases or potential failure types:**
  - Invalid domains
  - Source database lacking comparison data
  - API throttling
- **Sub-workflow reference:** None

### Wait before keyword gap (3s)1
- **Type and technical role:** `n8n-nodes-base.wait`; inserts another delay before the second competitor request.
- **Configuration choices:** waits `3`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Wait before keyword gap (3s)  
  - Output: Get keyword gaps 2nd Competitor
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:** same as previous Wait node
- **Sub-workflow reference:** None

### Get keyword gaps 2nd Competitor
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; compares competitor 2 domain vs the user’s domain.
- **Configuration choices:** same as competitor 1 comparison, using `competitor_domain_2`.
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_domain_2 }}`
  - `{{ $('Configuration').item.json.your_domain }}`
- **Input and output connections:**  
  - Input: Wait before keyword gap (3s)1  
  - Output: Merge keyword gaps
- **Version-specific requirements:** SE Ranking community node required.
- **Edge cases or potential failure types:**
  - If competitor 2 falls back to competitor 1, duplicate result sets may be produced
- **Sub-workflow reference:** None

### Merge keyword gaps
- **Type and technical role:** `n8n-nodes-base.merge`; combines the two competitor result streams.
- **Configuration choices:** default merge behavior with two inputs; effectively collects both datasets for downstream processing.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Inputs:
    - Get keyword gaps 1st Competitor
    - Get keyword gaps 2nd Competitor
  - Output: Format keyword gaps
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Different response shapes between calls
  - If one branch fails, merged downstream data may be incomplete
- **Sub-workflow reference:** None

### Format keyword gaps
- **Type and technical role:** `n8n-nodes-base.code`; filters, deduplicates, and formats keyword gap rows.
- **Configuration choices:**
  - Reads all merged items
  - Uses `Configuration` for `volume_min` and `your_domain`
  - Keeps only Latin/English-looking keywords via regex
  - Filters by minimum volume
  - Deduplicates by lowercase trimmed keyword
  - Maps key output fields:
    - keyword
    - volume
    - difficulty
    - cpc
    - competitor_position
    - competitor_url
    - your_position
    - your_domain
    - date
  - Limits output to 40 rows
- **Key expressions or variables used:**
  - `$('Configuration').first().json`
  - Regex: `/^[a-zA-Z0-9\\s\\-_'\".,?!&%()/\\\\+:;]+$/`
- **Input and output connections:**  
  - Input: Merge keyword gaps  
  - Output: Export to Sheets: Organic Gaps
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - Regex excludes many valid non-English Latin-language keywords with accents
  - If volume is a string or absent, numeric filtering may behave unexpectedly unless SE Ranking returns numbers
  - Deduplication removes duplicates without preserving which competitor supplied them
- **Sub-workflow reference:** None

### Export to Sheets: Organic Gaps
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends formatted keyword gaps to a sheet.
- **Configuration choices:** append with automapping, target tab via `gid=0`, empty spreadsheet URL placeholder.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Format keyword gaps  
  - Output: none
- **Version-specific requirements:** Google Sheets OAuth2 needed.
- **Edge cases or potential failure types:**
  - Export fails until spreadsheet URL/document ID is configured
  - Same `gid=0` as other exports may cause all datasets to append into the same tab unless changed manually
- **Sub-workflow reference:** None

---

## 2.4 Question Keyword Discovery

**Overview:**  
This block fetches question-based keywords around the submitted seed topic, restricted to informational intent and minimum volume, then formats and exports them.

**Nodes Involved:**  
- Wait before questions (6s)
- Get question keywords
- Format questions
- Export to Sheets: Questions

### Wait before questions (6s)
- **Type and technical role:** `n8n-nodes-base.wait`; staggers the request timing.
- **Configuration choices:** waits `6`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Configuration  
  - Output: Get question keywords
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:** longer execution duration
- **Sub-workflow reference:** None

### Get question keywords
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; gets question-style keyword suggestions.
- **Configuration choices:**
  - Resource: `keywordResearch`
  - Operation: `getQuestions`
  - Seed keyword from `seed_topic`
  - Source from config
  - Additional fields:
    - `limit = 30`
    - `intents = ['I']` for informational intent
    - `volumeFrom = volume_min`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.seed_topic }}`
  - `{{ $('Configuration').item.json.source }}`
  - `{{ $('Configuration').item.json.volume_min }}`
- **Input and output connections:**  
  - Input: Wait before questions (6s)  
  - Output: Format questions
- **Version-specific requirements:** SE Ranking community node required.
- **Edge cases or potential failure types:**
  - Seed topic too niche may return few or zero questions
  - Unsupported market/source
  - API limits
- **Sub-workflow reference:** None

### Format questions
- **Type and technical role:** `n8n-nodes-base.code`; flattens the returned keyword list into rows.
- **Configuration choices:**
  - Reads `data.keywords`
  - Keeps rows with `kw.keyword`
  - Outputs:
    - question
    - volume
    - difficulty
    - cpc
    - serp_features
    - intent
    - seed_topic
    - date
  - Limits to 30 rows
- **Key expressions or variables used:**
  - `$('Configuration').first().json`
  - joins arrays like `(kw.serp_features || []).join(', ')`
- **Input and output connections:**  
  - Input: Get question keywords  
  - Output: Export to Sheets: Questions
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - If API changes from `keywords` to another shape, output becomes empty
  - Arrays may be empty and join to blank strings
- **Sub-workflow reference:** None

### Export to Sheets: Questions
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends discovered question keywords to Google Sheets.
- **Configuration choices:** append, auto-map input data, tab via `gid=0`, document URL not yet configured.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Format questions  
  - Output: none
- **Version-specific requirements:** Google Sheets OAuth2 needed.
- **Edge cases or potential failure types:** same Google Sheets configuration risks as other export nodes
- **Sub-workflow reference:** None

---

## 2.5 Your Existing AI Topic Footprint

**Overview:**  
This branch identifies prompts where the user’s domain already appears in AI search results, currently using the AI Overview engine, and exports that footprint.

**Nodes Involved:**  
- Wait before your AI topics (9s)
- Get your AI topics
- Format your AI topics
- Export to Sheets: Your AI Topics

### Wait before your AI topics (9s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:** waits `9`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Configuration  
  - Output: Get your AI topics
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:** longer workflow duration
- **Sub-workflow reference:** None

### Get your AI topics
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; retrieves prompts by target/domain.
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByTarget`
  - Engine: `ai-overview`
  - Scope from config: `base_domain`
  - Domain: `your_domain`
  - Source from config
  - Additional fields:
    - sort by volume descending
    - limit from config
    - `volumeFrom = 500`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.scope }}`
  - `{{ $('Configuration').item.json.your_domain }}`
  - `{{ $('Configuration').item.json.source }}`
- **Input and output connections:**  
  - Input: Wait before your AI topics (9s)  
  - Output: Format your AI topics
- **Version-specific requirements:** SE Ranking community node required.
- **Edge cases or potential failure types:**
  - Uses hardcoded `volumeFrom: 500` instead of `volume_min` variable
  - AI search data availability may be limited by market/domain
- **Sub-workflow reference:** None

### Format your AI topics
- **Type and technical role:** `n8n-nodes-base.code`; flattens AI prompts into reportable rows.
- **Configuration choices:**
  - Reads `data.prompts`
  - Outputs:
    - prompt
    - volume
    - appearance_type
    - snippet_preview` (first 200 chars of answer text)
    - your_domain
    - date
- **Key expressions or variables used:**
  - `$('Configuration').first().json`
  - safe snippet extraction with try/catch
- **Input and output connections:**  
  - Input: Get your AI topics  
  - Outputs:
    - Export to Sheets: Your AI Topics
    - Merge your topics and competitor topics
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - Missing `prompts` array yields empty result
  - Answer text may be absent; handled safely
- **Sub-workflow reference:** None

### Export to Sheets: Your AI Topics
- **Type and technical role:** `n8n-nodes-base.googleSheets`
- **Configuration choices:** append, automap, `gid=0`, empty document URL placeholder
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Format your AI topics  
  - Output: none
- **Version-specific requirements:** Google Sheets OAuth2 required
- **Edge cases or potential failure types:** same export configuration risks as above
- **Sub-workflow reference:** None

---

## 2.6 Competitor AI Topic Gap Detection

**Overview:**  
This block collects AI prompts for both competitor brands, merges them with the user’s own AI-topic prompt list, and identifies SEO-related prompts where a competitor appears but the user does not.

**Nodes Involved:**  
- Wait before competitor topics (12s)
- Get 1st competitor AI topics
- Wait before competitor topics (10s)
- Get 2nd competitor AI topics
- Merge your topics and competitor topics
- Find topics you're missing
- Export to Sheets: Competitor AI Topics

### Wait before competitor topics (12s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:** waits `12`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Configuration  
  - Output: Get 1st competitor AI topics
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:** execution delay
- **Sub-workflow reference:** None

### Get 1st competitor AI topics
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; retrieves prompts associated with competitor brand 1.
- **Configuration choices:**
  - Resource: `aiSearch`
  - Operation: `getPromptsByBrand`
  - Engine: `ai-overview`
  - Brand name from config
  - Source from config
  - Additional fields:
    - sort by volume desc
    - limit from config
    - volumeFrom from config
    - `multiKeywordIncluded = seo`
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_brand_1 }}`
  - `{{ $('Configuration').item.json.volume_min }}`
- **Input and output connections:**  
  - Input: Wait before competitor topics (12s)  
  - Output: Merge your topics and competitor topics
- **Version-specific requirements:** SE Ranking community node required.
- **Edge cases or potential failure types:**
  - Brand matching may be noisy if competitor brand is ambiguous
  - `multiKeywordIncluded=seo` narrows retrieval to SEO-related prompts but does not guarantee complete topical precision
- **Sub-workflow reference:** None

### Wait before competitor topics (10s)
- **Type and technical role:** `n8n-nodes-base.wait`
- **Configuration choices:** despite the name, configured amount is also `12`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Configuration  
  - Output: Get 2nd competitor AI topics
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Node name and configured duration differ, which can confuse maintenance
- **Sub-workflow reference:** None

### Get 2nd competitor AI topics
- **Type and technical role:** `@seranking/n8n-nodes-seranking.seRanking`; retrieves prompts associated with competitor brand 2.
- **Configuration choices:** same as competitor 1, using `competitor_brand_2`.
- **Key expressions or variables used:**
  - `{{ $('Configuration').item.json.competitor_brand_2 }}`
- **Input and output connections:**  
  - Input: Wait before competitor topics (10s)  
  - Output: Merge your topics and competitor topics
- **Version-specific requirements:** SE Ranking community node required.
- **Edge cases or potential failure types:**
  - If competitor 2 falls back to competitor 1, gap analysis may duplicate the same competitor source
- **Sub-workflow reference:** None

### Merge your topics and competitor topics
- **Type and technical role:** `n8n-nodes-base.merge`; waits for and combines three inputs:
  1. formatted user prompt rows
  2. raw competitor 1 prompt response
  3. raw competitor 2 prompt response
- **Configuration choices:** `numberInputs = 3`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Inputs:
    - Format your AI topics
    - Get 1st competitor AI topics
    - Get 2nd competitor AI topics
  - Output: Find topics you're missing
- **Version-specific requirements:** `typeVersion 3.2`
- **Edge cases or potential failure types:**
  - Mixed data shapes are intentional here, but future maintainers must preserve that assumption
  - If one competitor call fails, downstream code may misidentify competitor order
- **Sub-workflow reference:** None

### Find topics you're missing
- **Type and technical role:** `n8n-nodes-base.code`; compares user prompts against competitor prompts and returns unique SEO-related gaps.
- **Configuration choices:**
  - Uses a hardcoded SEO term list:
    - seo, keyword, ranking, backlink, serp, traffic, search engine, rank track, domain, audit, content, google, organic, link building, on-page, off-page, site speed, crawl, index
  - Defines `isSeoRelated()` by substring match
  - Treats rows with `your_domain` as the user’s formatted prompt set
  - Treats items with `prompts` arrays as competitor raw API responses
  - Tags competitor prompts by input order:
    - first competitor raw input => competitor_brand_1
    - second competitor raw input => competitor_brand_2
  - Deduplicates by normalized prompt text
  - Excludes prompts already present in the user’s prompt set
  - Outputs:
    - prompt
    - volume
    - their_appearance_type
    - snippet_preview
    - competitor
    - gap_note
    - date
  - Limits output to 300 rows
- **Key expressions or variables used:**
  - `$('Configuration').first().json`
  - `allInputs.filter(i => i.json.your_domain !== undefined)`
  - `allInputs.filter(i => i.json.prompts !== undefined)`
- **Input and output connections:**  
  - Input: Merge your topics and competitor topics  
  - Output: Export to Sheets: Competitor AI Topics
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - SEO filter is heuristic and may exclude relevant prompts or include marginal ones
  - Competitor tagging depends on merge input order remaining stable
  - If your prompts are formatted differently from competitor raw prompts, exact string matching may miss semantically similar prompts
  - Duplicate competitor fallback can skew results
- **Sub-workflow reference:** None

### Export to Sheets: Competitor AI Topics
- **Type and technical role:** `n8n-nodes-base.googleSheets`
- **Configuration choices:** append, auto-map, `gid=0`, spreadsheet URL placeholder empty
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input: Find topics you're missing  
  - Output: none
- **Version-specific requirements:** Google Sheets OAuth2 required
- **Edge cases or potential failure types:** same export configuration risks as other Sheets nodes
- **Sub-workflow reference:** None

---

## 2.7 Notes and Visual Documentation Nodes

**Overview:**  
These nodes are documentation-only sticky notes inside the canvas. They do not affect execution but provide setup, purpose, sharing, and block-level explanations.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note3
- Sticky Note4

### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** contains full workflow description, requirements, setup, and customization notes.
- **Key expressions or variables used:** None
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none; ignored at runtime
- **Sub-workflow reference:** None

### Sticky Note1
- **Type and technical role:** sticky note for the leaderboard section.
- **Configuration choices:** describes AI Search Leaderboard purpose.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

### Sticky Note3
- **Type and technical role:** sticky note for AI footprint comparison section.
- **Configuration choices:** explains merge and SEO filtering logic for competitor vs own prompts.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

### Sticky Note4
- **Type and technical role:** sticky note for form sharing instructions.
- **Configuration choices:** explains activation, production URL retrieval, and iframe embedding.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Visual documentation for workflow purpose, setup, requirements, and customization |  |  | Track who wins AI search and find the topics you're missing with SE Ranking; Who is this for; What this workflow does; What you'll get; How it works; Requirements; Setup; Customization; [Get one here](https://online.seranking.com/admin.api.dashboard.html); [SE Ranking community node](https://www.npmjs.com/package/@seranking/n8n-nodes-seranking) |
| Sticky Note4 | stickyNote | Visual documentation for sharing or embedding the form |  |  | 📋 How to share or embed this form; Activate the workflow; copy the Production URL; share directly; embed with iframe |
| Sticky Note1 | stickyNote | Visual documentation for AI leaderboard block |  |  | 🏆 AI Search Leaderboard — Compares you and up to 2 competitors across ChatGPT, Perplexity, Gemini, AI Overviews, and AI Mode — returns share of voice and presence scores per engine. |
| Sticky Note3 | stickyNote | Visual documentation for AI footprint comparison block |  |  | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Domain Input Form | formTrigger | Public workflow entry form for collecting domain, competitors, topic, and market |  | Configuration | 📋 How to share or embed this form; Activate the workflow; copy the Production URL; share directly; embed with iframe |
| Configuration | set | Normalizes form fields into reusable workflow variables | Domain Input Form | Get AI leaderboard; Wait before keyword gap (3s); Wait before questions (6s); Wait before your AI topics (9s); Wait before competitor topics (12s); Wait before competitor topics (10s) | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Get AI leaderboard | @seranking/n8n-nodes-seranking.seRanking | Retrieves AI search leaderboard across selected engines | Configuration | Format leaderboard | 🏆 AI Search Leaderboard — Compares you and up to 2 competitors across ChatGPT, Perplexity, Gemini, AI Overviews, and AI Mode — returns share of voice and presence scores per engine. |
| Format leaderboard | code | Flattens leaderboard data into export rows | Get AI leaderboard | Export to Sheets: AI Leaderboard | 🏆 AI Search Leaderboard — Compares you and up to 2 competitors across ChatGPT, Perplexity, Gemini, AI Overviews, and AI Mode — returns share of voice and presence scores per engine. |
| Export to Sheets: AI Leaderboard | googleSheets | Appends leaderboard rows to Google Sheets | Format leaderboard |  | 🏆 AI Search Leaderboard — Compares you and up to 2 competitors across ChatGPT, Perplexity, Gemini, AI Overviews, and AI Mode — returns share of voice and presence scores per engine. |
| Wait before keyword gap (3s) | wait | Delays first keyword-gap branch to reduce API pressure | Configuration | Get keyword gaps 1st Competitor; Wait before keyword gap (3s)1 |  |
| Get keyword gaps 1st Competitor | @seranking/n8n-nodes-seranking.seRanking | Retrieves keyword gap data for competitor 1 vs your domain | Wait before keyword gap (3s) | Merge keyword gaps |  |
| Wait before keyword gap (3s)1 | wait | Delays second keyword-gap request | Wait before keyword gap (3s) | Get keyword gaps 2nd Competitor |  |
| Get keyword gaps 2nd Competitor | @seranking/n8n-nodes-seranking.seRanking | Retrieves keyword gap data for competitor 2 vs your domain | Wait before keyword gap (3s)1 | Merge keyword gaps |  |
| Merge keyword gaps | merge | Combines both competitor keyword-gap datasets | Get keyword gaps 1st Competitor; Get keyword gaps 2nd Competitor | Format keyword gaps |  |
| Format keyword gaps | code | Filters, deduplicates, and formats keyword opportunities | Merge keyword gaps | Export to Sheets: Organic Gaps |  |
| Export to Sheets: Organic Gaps | googleSheets | Appends organic keyword gaps to Google Sheets | Format keyword gaps |  |  |
| Wait before questions (6s) | wait | Delays question-keyword request | Configuration | Get question keywords |  |
| Get question keywords | @seranking/n8n-nodes-seranking.seRanking | Retrieves informational question keywords for the seed topic | Wait before questions (6s) | Format questions |  |
| Format questions | code | Formats question-keyword results for export | Get question keywords | Export to Sheets: Questions |  |
| Export to Sheets: Questions | googleSheets | Appends question keywords to Google Sheets | Format questions |  |  |
| Wait before your AI topics (9s) | wait | Delays your AI-topics retrieval | Configuration | Get your AI topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Get your AI topics | @seranking/n8n-nodes-seranking.seRanking | Retrieves prompts where your domain appears in AI search | Wait before your AI topics (9s) | Format your AI topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Format your AI topics | code | Formats your AI prompt footprint into rows | Get your AI topics | Export to Sheets: Your AI Topics; Merge your topics and competitor topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Export to Sheets: Your AI Topics | googleSheets | Appends your AI topic footprint to Google Sheets | Format your AI topics |  | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Wait before competitor topics (12s) | wait | Delays competitor 1 AI-topic retrieval | Configuration | Get 1st competitor AI topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Wait before competitor topics (10s) | wait | Delays competitor 2 AI-topic retrieval | Configuration | Get 2nd competitor AI topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Get 1st competitor AI topics | @seranking/n8n-nodes-seranking.seRanking | Retrieves AI prompts associated with competitor brand 1 | Wait before competitor topics (12s) | Merge your topics and competitor topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Get 2nd competitor AI topics | @seranking/n8n-nodes-seranking.seRanking | Retrieves AI prompts associated with competitor brand 2 | Wait before competitor topics (10s) | Merge your topics and competitor topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Merge your topics and competitor topics | merge | Combines your formatted prompts with both competitor raw prompt datasets | Format your AI topics; Get 1st competitor AI topics; Get 2nd competitor AI topics | Find topics you're missing | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Find topics you're missing | code | Finds SEO-relevant AI prompts where competitors appear and you do not | Merge your topics and competitor topics | Export to Sheets: Competitor AI Topics | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |
| Export to Sheets: Competitor AI Topics | googleSheets | Appends competitor-only AI topic gaps to Google Sheets | Find topics you're missing |  | 🤖 AI Footprint Comparison — Your AI topics and both competitors' topics merge before the gap code runs. Only SEO-context prompts are kept — filters out unrelated brand mentions. |

---

# 4. Reproducing the Workflow from Scratch

1. **Install prerequisites**
   1. Install the SE Ranking community node package in your n8n instance: `@seranking/n8n-nodes-seranking`.
   2. Make sure your n8n instance supports:
      - Form Trigger nodes
      - Google Sheets node
      - Code node
      - Merge node with multiple inputs
   3. Obtain:
      - SE Ranking API token
      - Google Sheets OAuth2 credentials
      - A Google Spreadsheet with separate tabs recommended for each dataset

2. **Create the workflow**
   1. Create a new workflow.
   2. Name it something like: `Track who wins AI search and find the topics you're missing with SE Ranking`.

3. **Create the trigger form**
   1. Add a **Form Trigger** node named `Domain Input Form`.
   2. Set:
      - Form title: `AI Search Gap Analysis`
      - Description: `Enter your domain and up to 2 competitors to see who's winning AI search and find the topics you're missing.`
   3. Add these fields in order:
      - Text: `Your Domain`, required
      - Text: `Your Brand Name`, required
      - Text: `Seed Topic`, required
      - Text: `Competitor 1 Domain`, required
      - Text: `Competitor 1 Brand`, required
      - Text: `Competitor 2 Domain`, optional
      - Text: `Competitor 2 Brand`, optional
      - Dropdown: `Target Market`, required, with options:
        `us, uk, de, fr, es, it, au, ca, pl`

4. **Create the configuration node**
   1. Add a **Set** node named `Configuration`.
   2. Connect `Domain Input Form -> Configuration`.
   3. Create these fields:
      - `your_domain` = `{{ $json['Your Domain'] }}`
      - `your_brand` = `{{ $json['Your Brand Name'] }}`
      - `competitor_domain_1` = `{{ $json['Competitor 1 Domain'] }}`
      - `competitor_brand_1` = `{{ $json['Competitor 1 Brand'] }}`
      - `competitor_domain_2` = `{{ $json['Competitor 2 Domain'] || $json['Competitor 1 Domain'] }}`
      - `competitor_brand_2` = `{{ $json['Competitor 2 Brand'] || $json['Competitor 1 Brand'] }}`
      - `seed_topic` = `{{ $json['Seed Topic'] }}`
      - `source` = `{{ $json['Target Market'] }}`
      - `scope` = `base_domain`
      - `volume_min` = `500`
      - `results_limit` = `200`

5. **Create the AI leaderboard branch**
   1. Add an **SE Ranking** node named `Get AI leaderboard`.
   2. Connect `Configuration -> Get AI leaderboard`.
   3. Configure:
      - Resource: `aiSearch`
      - Operation: `getLeaderboard`
      - Source: `{{ $json.source }}`
      - Engines: `ai-overview`, `chatgpt`, `perplexity`, `gemini`
      - Primary brand: `{{ $json.your_brand }}`
      - Primary target: `{{ $json.your_domain }}`
      - Competitor 1:
        - brand: `{{ $json.competitor_brand_1 }}`
        - target: `{{ $json.competitor_domain_1 }}`
      - Competitor 2:
        - brand: `{{ $json.competitor_brand_2 }}`
        - target: `{{ $json.competitor_domain_2 }}`
   4. Attach SE Ranking credentials.

6. **Format the leaderboard**
   1. Add a **Code** node named `Format leaderboard`.
   2. Connect `Get AI leaderboard -> Format leaderboard`.
   3. Paste logic equivalent to:
      - read `leaderboard` and `results`
      - output one row per domain
      - include:
        `rank, domain, share_of_voice_pct, brand_presence_total, link_presence_total, is_your_domain, chatgpt_brand, chatgpt_links, perplexity_brand, perplexity_links, gemini_brand, gemini_links, ai_overview_brand, ai_overview_links, ai_mode_brand, ai_mode_links, date`
      - format share of voice as a percentage string
      - default missing values to zero

7. **Add the leaderboard export**
   1. Add a **Google Sheets** node named `Export to Sheets: AI Leaderboard`.
   2. Connect `Format leaderboard -> Export to Sheets: AI Leaderboard`.
   3. Configure:
      - Operation: `Append`
      - Mapping mode: auto-map
      - Spreadsheet: your target spreadsheet URL
      - Sheet/tab: choose the leaderboard tab
   4. Attach Google Sheets OAuth2 credentials.

8. **Create the keyword-gap branch delays**
   1. Add a **Wait** node named `Wait before keyword gap (3s)`.
   2. Connect `Configuration -> Wait before keyword gap (3s)`.
   3. Set delay amount to `3`.
   4. Add another **Wait** node named `Wait before keyword gap (3s)1`.
   5. Connect `Wait before keyword gap (3s) -> Wait before keyword gap (3s)1`.
   6. Also connect `Wait before keyword gap (3s) -> Get keyword gaps 1st Competitor` later.

9. **Create first competitor keyword-gap request**
   1. Add an **SE Ranking** node named `Get keyword gaps 1st Competitor`.
   2. Connect `Wait before keyword gap (3s) -> Get keyword gaps 1st Competitor`.
   3. Configure:
      - Operation: `getKeywordsComparison`
      - Domain: `{{ $('Configuration').item.json.competitor_domain_1 }}`
      - Compare domain: `{{ $('Configuration').item.json.your_domain }}`
      - Source: `{{ $('Configuration').item.json.source }}`
      - Diff: `1`
      - Additional fields:
        - limit: `{{ $('Configuration').item.json.results_limit }}`
        - orderField: `volume`
        - orderType: `desc`

10. **Create second competitor keyword-gap request**
    1. Add an **SE Ranking** node named `Get keyword gaps 2nd Competitor`.
    2. Connect `Wait before keyword gap (3s)1 -> Get keyword gaps 2nd Competitor`.
    3. Use the same configuration as above, but set:
       - Domain: `{{ $('Configuration').item.json.competitor_domain_2 }}`

11. **Merge keyword-gap data**
    1. Add a **Merge** node named `Merge keyword gaps`.
    2. Connect:
       - `Get keyword gaps 1st Competitor -> Merge keyword gaps` input 1
       - `Get keyword gaps 2nd Competitor -> Merge keyword gaps` input 2

12. **Format keyword gaps**
    1. Add a **Code** node named `Format keyword gaps`.
    2. Connect `Merge keyword gaps -> Format keyword gaps`.
    3. Implement logic to:
       - read all incoming items
       - load config from `Configuration`
       - parse `volume_min`
       - keep only keywords matching a Latin-script regex
       - keep only rows with volume >= `volume_min`
       - deduplicate by normalized keyword text
       - map fields:
         `keyword, volume, difficulty, cpc, competitor_position, competitor_url, your_position, your_domain, date`
       - keep only the first 40 rows

13. **Export organic gaps**
    1. Add a **Google Sheets** node named `Export to Sheets: Organic Gaps`.
    2. Connect `Format keyword gaps -> Export to Sheets: Organic Gaps`.
    3. Configure append mode and select a dedicated sheet tab.

14. **Create the question-keyword branch**
    1. Add a **Wait** node named `Wait before questions (6s)`.
    2. Connect `Configuration -> Wait before questions (6s)`.
    3. Set amount to `6`.
    4. Add an **SE Ranking** node named `Get question keywords`.
    5. Connect `Wait before questions (6s) -> Get question keywords`.
    6. Configure:
       - Resource: `keywordResearch`
       - Operation: `getQuestions`
       - Keyword: `{{ $('Configuration').item.json.seed_topic }}`
       - Source: `{{ $('Configuration').item.json.source }}`
       - Additional fields:
         - limit: `30`
         - intents: `I`
         - volumeFrom: `{{ $('Configuration').item.json.volume_min }}`

15. **Format and export question keywords**
    1. Add a **Code** node named `Format questions`.
    2. Connect `Get question keywords -> Format questions`.
    3. Format rows with:
       `question, volume, difficulty, cpc, serp_features, intent, seed_topic, date`
    4. Join array fields like SERP features and intents with commas.
    5. Add a **Google Sheets** node named `Export to Sheets: Questions`.
    6. Connect `Format questions -> Export to Sheets: Questions`.
    7. Configure append mode to a dedicated tab.

16. **Create your AI-topics branch**
    1. Add a **Wait** node named `Wait before your AI topics (9s)`.
    2. Connect `Configuration -> Wait before your AI topics (9s)`.
    3. Set amount to `9`.
    4. Add an **SE Ranking** node named `Get your AI topics`.
    5. Connect `Wait before your AI topics (9s) -> Get your AI topics`.
    6. Configure:
       - Resource: `aiSearch`
       - Operation: `getPromptsByTarget`
       - Engine: `ai-overview`
       - Scope: `{{ $('Configuration').item.json.scope }}`
       - Domain: `{{ $('Configuration').item.json.your_domain }}`
       - Source: `{{ $('Configuration').item.json.source }}`
       - Additional fields:
         - sort: `volume`
         - sortOrder: `desc`
         - limit: `{{ $('Configuration').item.json.results_limit }}`
         - volumeFrom: `500`

17. **Format your AI topics**
    1. Add a **Code** node named `Format your AI topics`.
    2. Connect `Get your AI topics -> Format your AI topics`.
    3. Map each prompt to:
       `prompt, volume, appearance_type, snippet_preview, your_domain, date`
    4. For `snippet_preview`, safely extract up to 200 characters from `answer.text`.

18. **Export your AI topics**
    1. Add a **Google Sheets** node named `Export to Sheets: Your AI Topics`.
    2. Connect `Format your AI topics -> Export to Sheets: Your AI Topics`.
    3. Configure append mode to a dedicated tab.

19. **Create competitor AI-topics delays**
    1. Add a **Wait** node named `Wait before competitor topics (12s)`.
    2. Connect `Configuration -> Wait before competitor topics (12s)`.
    3. Set amount to `12`.
    4. Add another **Wait** node named `Wait before competitor topics (10s)`.
    5. Connect `Configuration -> Wait before competitor topics (10s)`.
    6. In the original workflow this node is named `(10s)` but also set to `12`; keep or correct this as desired.

20. **Create first competitor AI-topics request**
    1. Add an **SE Ranking** node named `Get 1st competitor AI topics`.
    2. Connect `Wait before competitor topics (12s) -> Get 1st competitor AI topics`.
    3. Configure:
       - Resource: `aiSearch`
       - Operation: `getPromptsByBrand`
       - Engine: `ai-overview`
       - Brand name: `{{ $('Configuration').item.json.competitor_brand_1 }}`
       - Source: `{{ $('Configuration').item.json.source }}`
       - Additional fields:
         - sort: `volume`
         - sortOrder: `desc`
         - limit: `{{ $('Configuration').item.json.results_limit }}`
         - volumeFrom: `{{ $('Configuration').item.json.volume_min }}`
         - multiKeywordIncluded: `seo`

21. **Create second competitor AI-topics request**
    1. Add an **SE Ranking** node named `Get 2nd competitor AI topics`.
    2. Connect `Wait before competitor topics (10s) -> Get 2nd competitor AI topics`.
    3. Use the same settings, but set brand name to:
       `{{ $('Configuration').item.json.competitor_brand_2 }}`

22. **Merge your topics with competitor topic responses**
    1. Add a **Merge** node named `Merge your topics and competitor topics`.
    2. Set number of inputs to `3`.
    3. Connect:
       - `Format your AI topics -> Merge your topics and competitor topics` input 1
       - `Get 1st competitor AI topics -> Merge your topics and competitor topics` input 2
       - `Get 2nd competitor AI topics -> Merge your topics and competitor topics` input 3

23. **Create the competitor gap-detection code**
    1. Add a **Code** node named `Find topics you're missing`.
    2. Connect `Merge your topics and competitor topics -> Find topics you're missing`.
    3. Implement logic to:
       - load config
       - define a list of SEO-related terms
       - detect whether a prompt is SEO-related by substring match
       - separate your formatted prompt rows from competitor raw API responses
       - create a set of your existing prompts
       - flatten competitor prompt arrays
       - assign competitor labels according to input order
       - exclude prompts already present in your set
       - deduplicate by normalized prompt text
       - output:
         `prompt, volume, their_appearance_type, snippet_preview, competitor, gap_note, date`
       - set `gap_note` to `Competitor appears here — you do not`
       - limit to 300 rows

24. **Export competitor AI-topic gaps**
    1. Add a **Google Sheets** node named `Export to Sheets: Competitor AI Topics`.
    2. Connect `Find topics you're missing -> Export to Sheets: Competitor AI Topics`.
    3. Configure append mode to a dedicated tab.

25. **Add credentials**
    1. In every SE Ranking node, assign your `SE Ranking account` credential.
    2. In every Google Sheets node, assign your `Google Sheets account` OAuth2 credential.

26. **Configure spreadsheet targets**
    1. In the provided JSON, all Google Sheets nodes point to `gid=0` and blank document URLs.
    2. Replace each with your spreadsheet URL.
    3. Ideally use separate tabs such as:
       - `AI Leaderboard`
       - `Organic Gaps`
       - `Questions`
       - `Your AI Topics`
       - `Competitor AI Topics`

27. **Optionally add sticky notes**
    1. Add a general note describing workflow purpose, requirements, setup, and customization.
    2. Add a leaderboard note.
    3. Add an AI footprint comparison note.
    4. Add a form-sharing note with iframe example.

28. **Activate and test**
    1. Save the workflow.
    2. Activate it.
    3. Open the Form Trigger production URL.
    4. Submit a test domain, brand, competitors, seed topic, and market.
    5. Confirm:
       - all SE Ranking nodes return data
       - each formatting node emits rows
       - each Google Sheets node appends to the correct tab

29. **Recommended post-build adjustments**
    1. Change the second competitor wait node label or value so the name matches its actual duration.
    2. Consider replacing hardcoded `volumeFrom: 500` in `Get your AI topics` with `{{ $('Configuration').item.json.volume_min }}` for consistency.
    3. Consider giving each export node a different sheet tab instead of all using `gid=0`.
    4. Consider stronger domain validation in the form or an IF node after `Configuration`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking API token is required | [SE Ranking API dashboard](https://online.seranking.com/admin.api.dashboard.html) |
| SE Ranking community node must be installed in n8n | [npm package: @seranking/n8n-nodes-seranking](https://www.npmjs.com/package/@seranking/n8n-nodes-seranking) |
| Activate the workflow to generate a production form URL | Relevant to the `Domain Input Form` node |
| The form can be embedded on a website using an iframe | Context: public submission UI |
| This workflow is currently inactive in the provided JSON | `active: false` |
| All Google Sheets export nodes currently have empty spreadsheet URLs and must be configured before use | Applies to every Google Sheets export node |
| All Google Sheets export nodes currently reference `gid=0`; using separate tabs is strongly recommended | Prevents mixed datasets in one tab |
| Competitor 2 falls back to competitor 1 if left blank | Useful for understanding duplicate-competitor behavior |
| AI topic gap detection uses heuristic SEO-term filtering rather than semantic classification | Important for maintenance and result interpretation |