Generate AI SEO blog posts with human review using SE Ranking and Claude

https://n8nworkflows.xyz/workflows/generate-ai-seo-blog-posts-with-human-review-using-se-ranking-and-claude-13381


# Generate AI SEO blog posts with human review using SE Ranking and Claude

## 1. Workflow Overview

**Workflow name:** Generate AI blog posts with human review using SE Ranking and Claude  
**Purpose:** Automate SEO blog production by (1) pulling keyword opportunities from SE Ranking, (2) generating briefs and full drafts with Claude (Anthropic), and (3) enforcing a **human approval step** via email + webhook before content is considered publishable. Drafts and decisions are tracked in **Google Sheets**.

**Target use cases**
- Content/marketing teams scaling editorial output while keeping human QA.
- SEO agencies producing multiple client articles from keyword opportunity data.
- Teams who want AI drafts but require compliance/brand review before publishing.

### 1.1 Entry Points
- **Manual entry (batch generation):** “Run Content Pipeline” (Manual Trigger)
- **Approval entry (review decision):** “Approval Webhook” (Webhook Trigger)

### 1.2 Logical Blocks
1. **Configuration & Run Trigger**: Sets domain/brand thresholds and controls batch size.
2. **Keyword Research & Scoring (SE Ranking + Code)**: Pulls keywords, scores them, selects top targets.
3. **SERP Enrichment (Related + PAA)**: Fetches related keywords and People Also Ask questions.
4. **Prompt Batching**: Combines all keyword datasets into one structure for the AI.
5. **AI Brief Generation (Claude)**: Produces content briefs for multiple keywords in one call.
6. **AI Draft Writing (Claude)**: Writes multiple full markdown articles in one call.
7. **Human Review Packaging + Tracking**: Splits the AI output into per-article columns, stores in Sheets, emails reviewer with approve/reject links.
8. **Approval Routing & Status Update**: Webhook receives decision, updates Google Sheets status, responds with an HTML page.
9. **Post-Approval Preparation (Optional Publishing)**: Retrieves approved drafts and splits them into individual items ready for CMS publishing (WordPress not implemented here).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Configuration & Run Trigger
**Overview:** Starts the content pipeline manually and defines all runtime parameters (domain, thresholds, reviewer email, webhook URL).  
**Nodes involved:** `Run Content Pipeline`, `Configuration`

#### Node: Run Content Pipeline
- **Type / role:** Manual Trigger; starts workflow execution manually.
- **Key config:** No parameters.
- **Outputs:** Single empty item to `Configuration`.
- **Failure modes:** None (manual execution only).

#### Node: Configuration
- **Type / role:** Set node; creates a JSON configuration object used across the workflow.
- **Configuration choices:**
  - Uses **raw JSON output** (one object).
  - Key settings:
    - `your_domain`, `source` (e.g., `us`)
    - `min_volume`, `max_difficulty` (note: `max_difficulty` is not actually enforced in the scoring node as written—see edge cases)
    - `articles_per_run` (intended batch size)
    - `target_word_count`, `brand_voice`, `reviewer_email`
    - `approval_webhook` (base URL used in emails)
- **Expressions/variables used downstream:** referenced via `$('Configuration').first().json.<field>`
- **Output connections:** To `Get Keyword Opportunities`.
- **Failure modes / edge cases:**
  - If `approval_webhook` is wrong/unreachable, approval links won’t work.
  - Missing/invalid `source` may break SE Ranking queries.
  - If `reviewer_email` invalid, email send will fail.

---

### Block 2.2 — Keyword Research & Scoring (SE Ranking + Code)
**Overview:** Pulls keyword opportunities for the domain from SE Ranking, then computes an “opportunity score” and selects a subset for article generation.  
**Nodes involved:** `Get Keyword Opportunities`, `Filter & Score Opportunities`

#### Node: Get Keyword Opportunities
- **Type / role:** SE Ranking community node (`@seranking/n8n-nodes-seranking.seRanking`); queries keyword opportunities for a domain.
- **Configuration choices:**
  - **Operation:** `getKeywords`
  - Inputs:
    - `domain` from config: `{{ $json.your_domain }}`
    - `source` from config: `{{ $json.source }}`
  - Additional fields:
    - `limit: 100`
    - order by `volume desc`
    - `volumeFrom: {{ $json.min_volume }}`
- **Credentials:** `seRankingApi` token required.
- **Outputs:** Items representing keyword opportunities (keyword, volume, difficulty, position, cpc, etc. as provided by API).
- **Failure modes / edge cases:**
  - Community node must be installed on a **self-hosted n8n**.
  - Auth/token invalid → 401/403.
  - API rate limits/timeouts.
  - Response schema changes could break downstream assumptions (e.g., `difficulty`, `position` missing).

#### Node: Filter & Score Opportunities
- **Type / role:** Code node; transforms SE Ranking results into scored candidates.
- **Configuration choices (logic):**
  - Reads config via `$('Configuration').first().json`.
  - For each keyword item:
    - Computes:
      - `volumeScore = min(volume/100, 50)`
      - `difficultyScore = (100 - difficulty) * 0.3`
      - `positionBonus = 20 if position between 11 and 30`
      - `opportunity_score = round(sum)`
    - Builds output object with:
      - `keyword`, `volume`, `difficulty`, `current_position`, `cpc`
      - `estimated_traffic = round(volume * 0.15)`
      - `content_type` derived from keyword text (tutorial/comparison/listicle/etc.)
  - Filters: `opportunity_score > 30`
  - Sorts descending by score
  - Slices to `config.articles_per_run`
  - **Returns only one item:** `return scored.length > 0 ? [scored[0]] : [];`
- **Connections:** Outputs to both `Get Related Keywords` and `Get Questions (PAA)`.
- **Critical edge cases / potential bugs:**
  - **Batch-size bug:** despite slicing to `articles_per_run`, the node returns **only the first** scored item. This contradicts the later “batch prompt” design and the email template expecting 3 articles.
  - **`max_difficulty` is never enforced.** Keywords above desired difficulty may still pass if score is high enough.
  - If SE Ranking returns `difficulty` undefined, calculations can yield `NaN` and later filters may drop everything.
  - If there are no keywords after filtering, the workflow ends early (returns empty array).

---

### Block 2.3 — SERP Enrichment (Related Keywords + People Also Ask)
**Overview:** For chosen keyword(s), fetches related keywords and PAA questions from SE Ranking, then merges the datasets.  
**Nodes involved:** `Get Related Keywords`, `Get Questions (PAA)`, `Merge Keyword Data`

#### Node: Get Related Keywords
- **Type / role:** SE Ranking node; retrieves related keywords.
- **Configuration choices:**
  - Resource: `keywordResearch`
  - Operation: `getRelated`
  - `keyword: {{ $json.keyword }}`
  - `source: {{ $json.source }}`
  - limit 20
- **Inputs:** From `Filter & Score Opportunities` item.
- **Outputs:** Related keyword dataset.
- **Failure modes:** Same as SE Ranking node above; plus empty related results are possible.

#### Node: Get Questions (PAA)
- **Type / role:** SE Ranking node; retrieves question keywords (People Also Ask style).
- **Configuration choices:**
  - Resource: `keywordResearch`
  - Operation: `getQuestions`
  - `keyword: {{ $('Filter & Score Opportunities').item.json.keyword }}`
  - `source: {{ $json.source }}`
  - limit 10
- **Inputs:** Connected from `Filter & Score Opportunities`, but the keyword is referenced explicitly from that node.
- **Outputs:** Question keyword dataset.
- **Edge cases:**
  - Expression uses `$('Filter & Score Opportunities').item...` which depends on item context. If later adapted to true batching/multiple items, this can mismatch per-item questions unless rewritten.
  - If PAA endpoint returns different field names than expected, later code may not find `json.keywords`.

#### Node: Merge Keyword Data
- **Type / role:** Merge node; combines results from Related + Questions.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Inputs:**
  - Input 0: `Get Related Keywords`
  - Input 1: `Get Questions (PAA)`
- **Outputs:** Combined item that includes both inputs’ JSON structures (depends on node’s combineAll behavior).
- **Edge cases:**
  - If one branch returns no items, merge behavior may produce no output (depending on n8n merge semantics in this mode).
  - CombineAll can create unexpected nesting/arrays; downstream code assumes access via `.all()` calls rather than merged JSON.

---

### Block 2.4 — Prompt Batching
**Overview:** Collects keyword candidates plus related/PAA data into a single `keywords_batch` array for the AI brief prompt.  
**Nodes involved:** `Combine all keywords into single prompt`

#### Node: Combine all keywords into single prompt
- **Type / role:** Code node; builds one JSON payload for Claude.
- **Configuration choices (logic):**
  - Reads:
    - `keywords = $('Filter & Score Opportunities').all()`
    - `relatedData = $('Get Related Keywords').all()`
    - `questionsData = $('Get Questions (PAA)').all()`
  - Creates `batchData = keywords.map((kw, index) => ({ ... }))` and assigns:
    - `keyword`, `volume`, `difficulty`, `content_type`
    - `related: relatedData[index]?.json.keywords || []`
    - `questions: questionsData[index]?.json.keywords || []`
  - Returns:
    - `keywords_batch`, `brand`, `target_word_count`, `brand_voice`
- **Inputs:** From `Merge Keyword Data` (but it doesn’t use `$input`—it pulls from other nodes by name).
- **Outputs:** To `Create Content Brief`.
- **Edge cases / failure types:**
  - If earlier nodes output only 1 keyword (current behavior), the batch will contain 1 entry, but later prompts/email assume 3.
  - Assumes SE Ranking nodes return `json.keywords`; if actual property differs, `related/questions` become empty arrays silently.
  - Index-based pairing assumes parallel arrays are aligned; this breaks if any branch returns missing items.

---

### Block 2.5 — AI Brief Generation (Claude / Anthropic)
**Overview:** Sends the full keyword batch to Claude to generate detailed briefs for each keyword.  
**Nodes involved:** `Create Content Brief`

#### Node: Create Content Brief
- **Type / role:** Anthropic (LangChain) node `@n8n/n8n-nodes-langchain.anthropic`; LLM call for brief generation.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
  - `system` prompt enforces:
    - Current year formatting using `{{ $now.format('YYYY') }}`
    - SEO requirements and structure constraints
    - Output expectations: “complete article in markdown format” (note: this is slightly inconsistent for a “brief” stage; but user message requests briefs)
  - User message content:
    - `Create detailed content briefs for ALL of these keywords: {{ JSON.stringify($json.keywords_batch, null, 2) }}`
    - Labels expected: `## Brief 1`, `## Brief 2`, `## Brief 3`
  - `maxTokens: 8000`, `temperature: 0.7`
  - `simplify: false` → output includes `content` array objects.
- **Inputs:** From batching node.
- **Outputs:** To `Write Article Draft`.
- **Failure modes / edge cases:**
  - Anthropic auth/quotas/rate limits.
  - Token limit: large batches/long keyword sets can exceed context or output limits.
  - Output formatting not guaranteed; downstream expects a text structure that the next node can use as input.

---

### Block 2.6 — AI Draft Writing (Claude / Anthropic)
**Overview:** Uses the generated briefs to draft full articles (multiple in one response).  
**Nodes involved:** `Write Article Draft`

#### Node: Write Article Draft
- **Type / role:** Anthropic LLM node; generates full markdown articles.
- **Configuration choices:**
  - Model: `claude-opus-4-6`
  - `system` prompt: detailed writing/SEO/formatting rules.
  - User message references briefs via: `{{ $json.content[0].text }}`
  - Reinforces current date/year constraints and formatting:
    - Must label outputs as:
      - `## Article 1: [keyword]`
      - `## Article 2: [keyword]`
      - `## Article 3: [keyword]`
  - `maxTokens: 16000`, `temperature: 0.7`
- **Inputs:** Output from `Create Content Brief`.
- **Outputs:** To `Prepare Review Package`.
- **Edge cases / failure types:**
  - If the brief output doesn’t exist at `$json.content[0].text`, expression fails.
  - If Claude does not follow “## Article N:” headers exactly, the splitter in the next block can fail or mis-split.
  - Response may be too large; truncation happens later for email/sheets columns but may still exceed Sheets cell limits.

---

### Block 2.7 — Human Review Packaging + Tracking (Sheets + Email)
**Overview:** Parses the LLM output into per-article fields, stores to Google Sheets, and sends a review email with approve/reject links.  
**Nodes involved:** `Prepare Review Package`, `Save Draft to Sheets`, `Send Review Email`

#### Node: Prepare Review Package
- **Type / role:** Code node; splits combined AI output into multiple sheet/email fields.
- **Configuration choices (logic):**
  - Reads config: `$('Configuration').first().json`
  - Reads LLM output: `const rawText = $input.first().json.content[0].text;`
  - Splits on regex: `/## Article \d+:\s*/i`
  - Generates `review_id = review_<timestamp>`
  - Builds output fields:
    - `review_id`, `status: pending_review`, `created_at`, `total_articles`, `brand`, `reviewer_email`
    - For each article N:
      - `keyword_N` extracted from first line of article block
      - `article_N` truncated to `MAX_CHARS = 40000` with `[TRUNCATED]`
      - `word_count_N`
- **Outputs:** To `Save Draft to Sheets`.
- **Edge cases:**
  - If Claude output lacks the expected headers, `articleBlocks` may be 1 huge block or empty.
  - Keyword extraction assumes first line is the title/keyword; may be wrong if article starts with metadata or blank lines.
  - Content truncation protects email size but not necessarily Google Sheets cell limits (Sheets has a ~50k char per cell limit; 40k is safer but still close).

#### Node: Save Draft to Sheets
- **Type / role:** Google Sheets node; append or update draft package.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching columns: `brand` (this is unusual; it will update rows by brand rather than by review_id)
  - Auto-map input data to schema with columns:
    - `review_id`, `status`, `created_at`, `total_articles`, `brand`, `reviewer_email`
    - `keyword_1..3`, `article_1..3`, `word_count_1..3`
  - `documentId` and `sheetName` are set via resource locator UI but appear blank in JSON export for docId.
- **Credentials:** Google Sheets OAuth2.
- **Outputs:** To `Send Review Email`.
- **Edge cases / failure types:**
  - If `documentId` not configured, node fails.
  - Matching on `brand` can overwrite previous runs for the same brand. Most likely you want matching on `review_id` and operation `append`.
  - API limits and permission issues.
  - Large cell content may exceed limits and error.

#### Node: Send Review Email
- **Type / role:** Email Send node (SMTP); sends review email with approve/reject links.
- **Configuration choices:**
  - Operation: `sendAndWait` (uses n8n’s approval-capable email flow)
  - HTML message includes previews via `substring(0,1500)` for each article.
  - Approval links:
    - `{{ $json.approval_webhook || "https://your-n8n-instance.com/webhook/content-approval" }}?action=approve&review_id={{ $json.review_id }}`
    - same for reject
  - Subject includes review id.
  - `toEmail: {{ $json.reviewer_email }}`
- **Credentials:** SMTP account.
- **Outputs:** Not connected further (review happens via webhook).
- **Edge cases:**
  - If fewer than 3 articles exist, references like `$json.article_2.substring(...)` can throw an expression error when `article_2` is undefined.
  - `approval_webhook` is not included in the output of `Prepare Review Package`; unless it exists from upstream item, the template falls back to the hardcoded URL.
  - SMTP failures (auth, TLS, spam rules, message size).

---

### Block 2.8 — Approval Webhook, Routing, and Status Updates
**Overview:** Receives reviewer decision via webhook query parameters, routes approve vs reject, updates Google Sheets status, and returns an HTML confirmation page.  
**Nodes involved:** `Approval Webhook`, `Route Decision`, `Update Status Approved`, `Update Status Rejected`, `Respond to Webhook Approved`, `Respond to Webhook Rejected`

#### Node: Approval Webhook
- **Type / role:** Webhook trigger; entry point for approval links.
- **Configuration choices:**
  - Path: `content-approval`
  - Response mode: `responseNode` (requires Respond to Webhook node in each path)
- **Inputs:** External HTTP GET from email links; uses query params:
  - `action=approve|reject`
  - `review_id=<id>`
- **Outputs:** To `Route Decision`.
- **Edge cases / failure types:**
  - Exposed webhook must be reachable from reviewer’s network.
  - No authentication/signature → anyone with the link can approve/reject. Consider adding a secret token.

#### Node: Route Decision
- **Type / role:** Switch node; routes based on `$json.query.action`.
- **Configuration choices:**
  - Rule “Approved” when `query.action == "approve"`
  - Rule “Rejected” when `query.action == "reject"`
- **Outputs:**
  - Approved → `Update Status Approved`
  - Rejected → `Update Status Rejected`
- **Edge cases:**
  - If `action` missing/other value, no output path triggers → webhook request may hang because response node never runs.

#### Node: Update Status Approved
- **Type / role:** Google Sheets update; sets status to approved for the row matching `review_id`.
- **Configuration choices:**
  - Operation: `update`
  - Matching columns: `review_id`
  - Writes:
    - `status = approved`
    - `review_id = {{ $json.query.review_id }}`
    - `created_at = {{ $now.toISO() }}`
- **Outputs:** To both `Get Approved Content` and `Respond to Webhook Approved` (two parallel outputs from same node).
- **Edge cases:**
  - If no row with `review_id` exists, update may fail or do nothing (depends on node behavior).
  - Document/sheet misconfiguration.

#### Node: Update Status Rejected
- **Type / role:** Google Sheets update; sets status rejected by `review_id`.
- **Configuration choices:** Same as approved but `status = rejected`.
- **Outputs:** To `Respond to Webhook Rejected`.

#### Node: Respond to Webhook Approved
- **Type / role:** Respond to Webhook; returns an HTML page confirming approval.
- **Configuration choices:** Respond with `text` containing HTML (green “approved” page).
- **Inputs:** From `Update Status Approved`.
- **Edge cases:** If this node is not reached (e.g., routing mismatch), request stays open.

#### Node: Respond to Webhook Rejected
- **Type / role:** Respond to Webhook; returns an HTML rejection page.
- **Inputs:** From `Update Status Rejected`.

---

### Block 2.9 — Post-Approval Preparation (Optional Publishing)
**Overview:** After approval, fetches approved row from Sheets and splits stored columns into individual article items suitable for a CMS publishing step.  
**Nodes involved:** `Get Approved Content`, `Split Articles` (Publishing is only noted via sticky note; no publishing node exists)

#### Node: Get Approved Content
- **Type / role:** Google Sheets read (lookup).
- **Configuration choices:**
  - Filter: `review_id` equals `{{ $('Approval Webhook').first().json.query.review_id }}`
- **Inputs:** From `Update Status Approved`.
- **Outputs:** To `Split Articles`.
- **Edge cases:**
  - If multiple rows share same review_id (should not happen), may return multiple items.
  - If none found, Split Articles fails or outputs empty.

#### Node: Split Articles
- **Type / role:** Code node; converts a single sheet row into multiple items (one per article).
- **Configuration choices:**
  - Reads `keyword_1/article_1/...` through `_3`
  - For each present article, emits an item:
    - `keyword`, `title` (capitalizes first letter), `content`, `word_count`
- **Outputs:** None connected (intended next step: WordPress/CMS).
- **Edge cases:**
  - Title capitalization is simplistic; for multi-word keywords it only uppercases first character of entire string.
  - If article fields were truncated, published content would be truncated unless you store full text elsewhere.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documentation / context |  |  | ## Generate AI SEO content with human review using SE Ranking and Claude… (includes setup + requirements + links: https://www.npmjs.com/package/@seranking/n8n-nodes-seranking, https://online.seranking.com/admin.api.dashboard.html) |
| Run Content Pipeline | Manual Trigger | Manual start of generation pipeline |  | Configuration |  |
| Configuration | Set | Defines runtime config variables | Run Content Pipeline | Get Keyword Opportunities |  |
| Keyword Research Phase | Sticky Note | Section label |  |  | ### Keyword Research… |
| Get Keyword Opportunities | SE Ranking (community) | Pull keyword opportunities for domain | Configuration | Filter & Score Opportunities | ### Keyword Research… |
| Filter & Score Opportunities | Code | Score/filter/select keywords | Get Keyword Opportunities | Get Related Keywords; Get Questions (PAA) | ### Keyword Research… |
| Get Related Keywords | SE Ranking (community) | Fetch related keywords | Filter & Score Opportunities | Merge Keyword Data |  |
| Get Questions (PAA) | SE Ranking (community) | Fetch PAA questions | Filter & Score Opportunities | Merge Keyword Data |  |
| Merge Keyword Data | Merge | Combine related + questions streams | Get Related Keywords; Get Questions (PAA) | Combine all keywords into single prompt |  |
| Combine all keywords into single prompt | Code | Build `keywords_batch` payload for AI | Merge Keyword Data | Create Content Brief |  |
| Content Brief Phase | Sticky Note | Section label |  |  | ### Content Brief Generation… |
| Create Content Brief | Anthropic (LangChain) | Generate briefs for keyword batch | Combine all keywords into single prompt | Write Article Draft | ### Content Brief Generation… |
| Content Writing Phase | Sticky Note | Section label |  |  | ### AI Content Writing… |
| Write Article Draft | Anthropic (LangChain) | Generate full drafts from briefs | Create Content Brief | Prepare Review Package | ### AI Content Writing… |
| Prepare Review Package | Code | Split AI output; create review package | Write Article Draft | Save Draft to Sheets |  |
| Save Draft to Sheets | Google Sheets | Store drafts + review metadata | Prepare Review Package | Send Review Email |  |
| Send Review Email | Email Send (SMTP) | Email reviewer with approve/reject links | Save Draft to Sheets |  | ### Human Review (Required)… |
| Human Review Phase | Sticky Note | Section label |  |  | ### Human Review (Required)… |
| Approval Webhook | Webhook | Receive approve/reject action |  | Route Decision | ### Human Review (Required)… |
| Route Decision | Switch | Route approve vs reject | Approval Webhook | Update Status Approved; Update Status Rejected |  |
| Update Status Approved | Google Sheets | Mark review approved | Route Decision (Approved) | Get Approved Content; Respond to Webhook Approved |  |
| Get Approved Content | Google Sheets | Fetch approved content row by review_id | Update Status Approved | Split Articles |  |
| Split Articles | Code | Emit one item per approved article | Get Approved Content |  | ### Publishing (Optional)… |
| Publishing Phase | Sticky Note | Section label |  |  | ### Publishing (Optional)… |
| Respond to Webhook Approved | Respond to Webhook | Return approval HTML response | Update Status Approved |  |  |
| Update Status Rejected | Google Sheets | Mark review rejected | Route Decision (Rejected) | Respond to Webhook Rejected |  |
| Respond to Webhook Rejected | Respond to Webhook | Return rejection HTML response | Update Status Rejected |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (self-hosted recommended because of the community node requirement).

2. **Add Sticky Notes (optional but matching the workflow):**
   - Add “Overview” sticky note with the provided content and links:
     - https://www.npmjs.com/package/@seranking/n8n-nodes-seranking  
     - https://online.seranking.com/admin.api.dashboard.html
   - Add sticky notes: “Keyword Research Phase”, “Content Brief Phase”, “Content Writing Phase”, “Human Review Phase”, “Publishing Phase”.

3. **Add node: Manual Trigger**
   - Name: `Run Content Pipeline`
   - Connect to `Configuration`.

4. **Add node: Set**
   - Name: `Configuration`
   - Mode: **Raw JSON**
   - Paste a config object with at least:
     - `your_domain`, `your_brand`, `brand_voice`
     - `source`
     - `min_volume`, `max_difficulty`
     - `articles_per_run`, `target_word_count`
     - `reviewer_email`
     - `approval_webhook` = `https://<your-n8n-host>/webhook/content-approval`
   - Connect to `Get Keyword Opportunities`.

5. **Install and configure the SE Ranking community node**
   - Install `@seranking/n8n-nodes-seranking` on the n8n instance.
   - Create credentials: **SE Ranking API token**.

6. **Add node: SE Ranking**
   - Name: `Get Keyword Opportunities`
   - Operation: `getKeywords`
   - Set:
     - Domain: `{{ $json.your_domain }}`
     - Source: `{{ $json.source }}`
     - Additional fields: limit 100, order volume desc, volumeFrom `{{ $json.min_volume }}`
   - Connect to `Filter & Score Opportunities`.

7. **Add node: Code**
   - Name: `Filter & Score Opportunities`
   - Implement the scoring logic (volume/difficulty/position bonus) and output enriched fields.
   - Important: if you want true batching, return **all selected items**, not only the first.
   - Connect outputs to both:
     - `Get Related Keywords`
     - `Get Questions (PAA)`

8. **Add node: SE Ranking (Related)**
   - Name: `Get Related Keywords`
   - Resource: keyword research
   - Operation: related keywords
   - Keyword: `{{ $json.keyword }}`
   - Source: `{{ $json.source }}`
   - Limit: 20
   - Connect to `Merge Keyword Data` (input 0).

9. **Add node: SE Ranking (Questions/PAA)**
   - Name: `Get Questions (PAA)`
   - Operation: questions
   - Keyword: use the current item keyword (prefer `{{ $json.keyword }}` if you implement batching correctly)
   - Source: `{{ $json.source }}`
   - Limit: 10
   - Connect to `Merge Keyword Data` (input 1).

10. **Add node: Merge**
    - Name: `Merge Keyword Data`
    - Mode: `combine`
    - Combine by: `combineAll`
    - Connect to `Combine all keywords into single prompt`.

11. **Add node: Code**
    - Name: `Combine all keywords into single prompt`
    - Build `{ keywords_batch: [...], brand, target_word_count, brand_voice }` by collecting data from the keyword + related + questions nodes.
    - Connect to `Create Content Brief`.

12. **Add node: Anthropic (LangChain)**
    - Name: `Create Content Brief`
    - Credential: Anthropic API key
    - Model: choose a Claude model (e.g., Sonnet)
    - System prompt: include brand voice, SEO requirements, and strict current-year rules using `$now`.
    - User message: include `{{ JSON.stringify($json.keywords_batch, null, 2) }}` and request clearly labeled briefs.
    - Connect to `Write Article Draft`.

13. **Add node: Anthropic (LangChain)**
    - Name: `Write Article Draft`
    - Model: choose a higher-quality model (e.g., Opus)
    - User message should reference briefs from prior output (commonly `{{ $json.content[0].text }}` if simplify=false).
    - Require labeled sections: `## Article 1: ...` etc.
    - Connect to `Prepare Review Package`.

14. **Add node: Code**
    - Name: `Prepare Review Package`
    - Parse the LLM output text and split on `## Article N:` headers.
    - Generate `review_id`, set `status=pending_review`, and create columns `keyword_1/article_1/word_count_1` etc.
    - Connect to `Save Draft to Sheets`.

15. **Add node: Google Sheets**
    - Name: `Save Draft to Sheets`
    - Credential: Google Sheets OAuth2
    - Operation: `appendOrUpdate` (or preferably `append` to avoid overwrites)
    - Configure Spreadsheet `documentId` and `sheetName`.
    - Ensure the sheet has columns matching the package fields (review_id, status, created_at, total_articles, brand, reviewer_email, keyword_1..3, article_1..3, word_count_1..3).
    - Connect to `Send Review Email`.

16. **Add node: Email Send (SMTP)**
    - Name: `Send Review Email`
    - Credential: SMTP
    - To: `{{ $json.reviewer_email }}`
    - Subject: include `{{ $json.review_id }}`
    - Message: HTML email with previews and links to webhook:
      - `.../webhook/content-approval?action=approve&review_id={{ $json.review_id }}`
      - `.../webhook/content-approval?action=reject&review_id={{ $json.review_id }}`
    - (Optional) keep “sendAndWait” approval options if you rely on n8n approval features, but this workflow actually uses the webhook links for the decision.

17. **Add node: Webhook**
    - Name: `Approval Webhook`
    - Path: `content-approval`
    - Response mode: `responseNode`
    - This is a second entry point, separate from the manual trigger.
    - Connect to `Route Decision`.

18. **Add node: Switch**
    - Name: `Route Decision`
    - Rules:
      - If `{{ $json.query.action }}` equals `approve` → Approved output
      - If equals `reject` → Rejected output
    - Connect Approved → `Update Status Approved`
    - Connect Rejected → `Update Status Rejected`

19. **Add node: Google Sheets (Update Approved)**
    - Name: `Update Status Approved`
    - Operation: `update`
    - Match on `review_id`
    - Set status = `approved`
    - Connect to:
      - `Get Approved Content`
      - `Respond to Webhook Approved`

20. **Add node: Google Sheets (Update Rejected)**
    - Name: `Update Status Rejected`
    - Operation: `update`
    - Match on `review_id`
    - Set status = `rejected`
    - Connect to `Respond to Webhook Rejected`

21. **Add node: Google Sheets (Get Approved Content)**
    - Name: `Get Approved Content`
    - Filter where `review_id` equals the webhook query review id.
    - Connect to `Split Articles`.

22. **Add node: Code**
    - Name: `Split Articles`
    - Convert sheet row columns into multiple items (one per article) with fields: title/content/keyword/word_count.
    - (Optional) connect next to WordPress or another CMS node for publishing.

23. **Add nodes: Respond to Webhook**
    - Name: `Respond to Webhook Approved` (returns HTML “approved” page)
    - Name: `Respond to Webhook Rejected` (returns HTML “rejected” page)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| SE Ranking community node is required (self-hosted n8n). | https://www.npmjs.com/package/@seranking/n8n-nodes-seranking |
| SE Ranking API token dashboard link. | https://online.seranking.com/admin.api.dashboard.html |
| Workflow design intent: keyword opportunities → briefs → drafts → email approval → track in Google Sheets. | Overview sticky note content |
| Customization parameters: `min_volume`, `max_difficulty`, `articles_per_run`, model selection. | Overview sticky note content |
| Publishing is optional and not implemented; approved articles are only prepared (Split Articles) for a CMS step. | “Publishing (Optional)” sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.