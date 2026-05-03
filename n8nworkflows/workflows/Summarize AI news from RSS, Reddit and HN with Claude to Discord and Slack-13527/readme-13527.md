Summarize AI news from RSS, Reddit and HN with Claude to Discord and Slack

https://n8nworkflows.xyz/workflows/summarize-ai-news-from-rss--reddit-and-hn-with-claude-to-discord-and-slack-13527


# Summarize AI news from RSS, Reddit and HN with Claude to Discord and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs daily (6:00 AM) to collect AI/tech news from multiple sources (RSS/Atom feeds, Reddit JSON, and Hacker News API), deduplicate items, optionally check against a PostgreSQL history, extract full article text via Jina Reader, analyze each article with Claude (Anthropic) into structured JSON, rank items, compile a final digest with Claude, and deliver it to **Discord** (embeds) and **Slack** (blocks). Optionally, it stores run metrics and articles in **PostgreSQL**.

**Target use cases:**
- Daily internal AI news briefing for a team (Slack) and community (Discord)
- Automated monitoring of AI model releases, research, tools, security, and open-source news
- Maintaining historical digest data for analytics and cross-day deduplication (Postgres enabled)

### 1.1 Trigger & Configuration
- A schedule trigger starts the flow; a configuration node provides all runtime settings (webhooks, limits, Postgres toggle).

### 1.2 Run Tracking (Optional PostgreSQL)
- Generates a run timestamp and (optionally) inserts a run record in Postgres; otherwise a synthetic run id is used.

### 1.3 Feed Collection & Parsing
- Builds a list of feed sources, fetches each feed, and parses items across different feed formats into a normalized “article” structure.

### 1.4 Deduplication (In-run + optional cross-run)
- Removes duplicates by URL hash + fuzzy title matching; optionally removes articles already seen recently in Postgres.

### 1.5 Content Extraction (Jina Reader)
- Extracts full text for top N items (to improve LLM analysis quality).

### 1.6 AI Analysis & Ranking (Claude)
- Claude analyzes each article into JSON (summary, importance, categories, sentiment, etc.).  
- A scoring function ranks articles using topic weights + social-signal boosts.

### 1.7 Digest Compilation (Claude)
- Claude compiles the selected top items into a structured digest JSON (lead story, top stories, quick hits, trend note).

### 1.8 Formatting, Delivery, Storage (Optional PostgreSQL)
- Formats Discord embeds, Slack blocks, and a Markdown digest.  
- Sends to Discord webhook + Slack channel.  
- Optionally stores analyzed articles and marks the run completed in Postgres.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & User Configuration
**Overview:** Runs daily at 6 AM and centralizes all user-editable settings (webhook URLs, selection limits, and Postgres toggle).  
**Nodes involved:**  
- `Run daily at 6 AM`
- `⚙️ Configure digest settings`

#### Node: Run daily at 6 AM
- **Type/role:** Schedule Trigger; entry point.
- **Config choices:** Cron expression `0 6 * * *` (server timezone).
- **Outputs:** To `⚙️ Configure digest settings`.
- **Edge cases:** Timezone mismatch; missed executions if n8n is down.

#### Node: ⚙️ Configure digest settings
- **Type/role:** Set node; runtime configuration container.
- **Key settings produced:**
  - `discord_webhook_url` (string)
  - `slack_channel_id` (string)
  - `max_extract_articles` (number, default 25)
  - `max_digest_articles` (number, default 12)
  - `dedup_lookback_days` (number, default 7)
  - `feed_lookback_hours` (number, default 24)
  - `use_postgres` (boolean, default false)
- **Connections:** Output to `Initialize run timestamp`.
- **Edge cases:** Missing/invalid webhook URL or Slack channel id will cause downstream delivery failures.

---

### Block 2.2 — Run Tracking (Optional)
**Overview:** Creates a run identifier and optionally writes a tracking row in Postgres. This run id is propagated through the pipeline.  
**Nodes involved:**  
- `Initialize run timestamp`
- `Create run record`

#### Node: Initialize run timestamp
- **Type/role:** Code node; generates `runDate`, `startedAt`, and a UUID-like `runId`.
- **Config choices:** Uses JS to build a random GUID-style identifier even when Postgres is disabled.
- **Outputs:** Single item: `{ runDate, startedAt, runId }` → `Create run record`.
- **Edge cases:** Minimal; relies on JS Date and random values.

#### Node: Create run record
- **Type/role:** Postgres node (Execute Query); inserts run row if enabled, otherwise returns a synthetic row.
- **Config choices (interpreted):**
  - If `use_postgres=true`:  
    `INSERT INTO digest_runs (run_date) VALUES ('YYYY-MM-DD') RETURNING id, run_date, started_at`
  - If `use_postgres=false`:  
    `SELECT '<runId>' AS id, '<runDate>' AS run_date, NOW() AS started_at`
- **Credentials:** `postgres: nxsi-postgres`
- **alwaysOutputData:** true (keeps flow running even if query returns no rows)
- **Outputs:** To `Build feed source list`
- **Failure modes:**
  - DB auth/connection errors when enabled
  - Missing tables (`digest_runs`) if schema not created

---

### Block 2.3 — Feed Collection & Parsing
**Overview:** Defines feed sources, fetches them, and parses them into normalized article items across HN API, Reddit JSON, RSS, and Atom.  
**Nodes involved:**  
- `Build feed source list`
- `Fetch feed content`
- `Parse articles from feeds`
- `Any articles found?`
- `Mark run as empty` (only on “no articles” path)

#### Node: Build feed source list
- **Type/role:** Code node; produces one item per feed.
- **Configuration choices:**
  - Hard-coded feed array with fields: `url`, `name`, `category`, `feedType`
  - Adds `runId` from `Create run record` (`items[0].json.id`)
- **Feeds included (10):**
  - HN Algolia API query about AI/LLM/GPT/Claude with `points>20`
  - TechCrunch AI RSS
  - The Verge AI RSS
  - Ars Technica RSS
  - Simon Willison Atom
  - MIT Technology Review RSS
  - Reddit: r/LocalLLaMA hot
  - Reddit: r/selfhosted hot
  - Anthropic RSS
  - OpenAI RSS
- **Outputs:** One feed item per source → `Fetch feed content`
- **Edge cases:** Feed URL changes, paywalls, rate limits, Reddit blocks (see HTTP node).

#### Node: Fetch feed content
- **Type/role:** HTTP Request; downloads feed raw content.
- **Config choices:**
  - URL from item: `{{$json.url}}`
  - Timeout 15s
  - Response format: text
  - Header `User-Agent: AI-News-Digest/1.0`
  - `onError=continueRegularOutput` (workflow continues even if some feeds fail)
- **Outputs:** To `Parse articles from feeds`
- **Failure modes:** 403/429 (especially Reddit), timeouts, invalid SSL, upstream downtime.

#### Node: Parse articles from feeds
- **Type/role:** Code node; normalizes fetched responses into article objects.
- **Key logic:**
  - Uses `feed_lookback_hours` (default 24) to filter by publish time.
  - Handles formats:
    - `hn_api`: parses JSON hits; uses `hit.created_at`; computes `socialScore = points + 2*num_comments`
    - `reddit`: parses JSON `data.children`; uses `created_utc`; transforms Reddit permalinks; `socialScore = score + 2*num_comments`; excerpt from `selftext`
    - `atom`: regex-based parsing of `<entry>`; reads `<title>`, `<link href>`, `<published>/<updated>`, `<summary>/<content>`
    - `rss` (default): regex-based parsing of `<item>`; reads `<title>`, `<link>`, `<pubDate>`, `<description>`
  - Attaches: `title, url, source, category, publishedAt, socialScore, excerpt, runId`
  - Collects per-feed parse errors into `errors[]`; stores on first article as `_feedErrors` when present.
  - Sorts by `socialScore` descending.
  - If zero articles: returns a special item with `{ url:'', _noArticles:true, articleCount:0, ... }`
- **Outputs:** Many article items (or one sentinel item) → `Any articles found?`
- **Edge cases/failures:**
  - Regex parsing can break on malformed XML or namespaces
  - Some RSS items don’t have `pubDate` (filter becomes permissive)
  - Mapping assumes HTTP results and feedMeta align by index; partial HTTP failures can desync in some n8n execution patterns (mitigated by continue-on-error but still index-based)

#### Node: Any articles found?
- **Type/role:** IF node; routes based on whether `$json.url` is non-empty.
- **True path:** to `Deduplicate by URL and title`
- **False path:** to `Mark run as empty`
- **Edge cases:** Sentinel item relies on empty `url`. If parser ever emits an article with empty URL, it could incorrectly route.

#### Node: Mark run as empty
- **Type/role:** Postgres Execute Query; marks status `no_articles` when enabled.
- **Config choices:**
  - If `use_postgres=true`: updates `digest_runs` row by id from `Create run record`
  - Else: `SELECT 1`
- **alwaysOutputData:** true
- **Failure modes:** DB schema mismatch (`status`, `completed_at`, `duration_seconds` columns must exist for this exact query).

---

### Block 2.4 — Deduplication (In-run + Postgres lookback)
**Overview:** Removes duplicates within the run using URL hashes and fuzzy title matching, then optionally removes items already stored recently in Postgres.  
**Nodes involved:**  
- `Deduplicate by URL and title`
- `Check database for recent duplicates`
- `Remove already-seen articles`

#### Node: Deduplicate by URL and title
- **Type/role:** Code node; in-run dedup.
- **Key logic:**
  - Computes `urlHash` using a djb2 hash of URL.
  - Normalizes titles: lowercase, remove non-alphanumerics, collapse whitespace.
  - Fuzzy dedup: Levenshtein distance ratio `< 0.2` against previously seen titles.
  - Returns a single object:
    - `articles: unique[]` (each article augmented with `urlHash`, `titleNormalized`)
    - `hashListSql`: comma-separated `'hash'` list for SQL IN()
    - `count`, `runId`
- **Outputs:** To `Check database for recent duplicates`
- **Edge cases:**
  - False positives for similar titles across different stories
  - URL hash collision is unlikely but possible
  - `hashListSql` is constructed without parameter binding (safe-ish because hashes are generated internally, but still not ideal style)

#### Node: Check database for recent duplicates
- **Type/role:** Postgres Execute Query; cross-run dedup if enabled.
- **Config choices:**
  - If `use_postgres=true`:  
    selects `url_hash` from `digest_articles` where `url_hash IN (...)` and `created_at` within `dedup_lookback_days` (default 7)
  - If disabled: returns empty result set
- **alwaysOutputData:** true
- **Outputs:** To `Remove already-seen articles`
- **Failure modes:** Missing table/columns (`digest_articles.url_hash`, `created_at`), DB issues.

#### Node: Remove already-seen articles
- **Type/role:** Code node; filters out articles whose `urlHash` appears in Postgres result set.
- **Inputs:**
  - `$input.all()` from Postgres query results
  - Reads dedup list from `Deduplicate by URL and title`
- **Outputs:** Single object `{ articles:fresh[], count, runId }` → `Select top articles for extraction`
- **Edge cases:** When Postgres disabled, `knownHashes` remains empty, so it behaves as “no cross-run dedup”.

---

### Block 2.5 — Content Extraction (Jina Reader)
**Overview:** Selects top N articles and extracts full text via Jina Reader to provide richer context for AI analysis.  
**Nodes involved:**  
- `Select top articles for extraction`
- `Extract full text via Jina`
- `Assemble extracted content`

#### Node: Select top articles for extraction
- **Type/role:** Code node; selects up to `max_extract_articles` (default 25).
- **Logic:** Sorts by `socialScore` desc; slices to limit; outputs one item per selected article.
- **Empty handling:** If no selected, outputs a sentinel `{ _empty:true, runId }`.
- **Outputs:** To `Extract full text via Jina`
- **Edge cases:** SocialScore is 0 for RSS/Atom sources; ordering then depends on original parse ordering.

#### Node: Extract full text via Jina
- **Type/role:** HTTP Request; calls Jina Reader.
- **Config choices:**
  - URL: `https://r.jina.ai/{{ $json.url }}`
  - Timeout 30s
  - Response format: JSON
  - Headers: `Accept: application/json`, `X-Return-Format: markdown`
  - `onError=continueRegularOutput`
- **Outputs:** To `Assemble extracted content`
- **Failure modes:** Paywalls, blocked sites, rate limits, invalid article URLs, large pages/timeouts.

#### Node: Assemble extracted content
- **Type/role:** Code node; merges Jina results with article metadata.
- **Logic:**
  - Aligns Jina results with `Select top articles for extraction` items by index.
  - Extracts markdown content (`jina.data.content` or `jina.content`), truncates to 8000 chars.
  - Tracks total Jina tokens if `jina.data.usage.tokens` exists.
  - Builds array of articles with `fullText`, `hasFullText`.
- **Outputs:** Single object `{ articles[], jinaTokens, count, runId }` → `Prepare analysis prompts`
- **Edge cases:** Index alignment issues if some HTTP calls fail unusually; content truncation may remove critical context.

---

### Block 2.6 — Per-Article AI Analysis (Claude Haiku) & Ranking
**Overview:** Builds per-article prompts, runs Claude analysis to produce structured JSON, then computes weighted ranking.  
**Nodes involved:**  
- `Prepare analysis prompts`
- `Claude Haiku (analyzer)` (model provider node)
- `Analyze article with Claude` (LLM chain)
- `Score and rank articles`

#### Node: Prepare analysis prompts
- **Type/role:** Code node; creates one prompt item per article.
- **Prompt content:** Requests **ONLY valid JSON** with fields:
  - `summary`, `importance (1-10)`, `categories`, `sentiment`, `key_entities`, `why_it_matters`, `reading_time_min`
- **Extra fields for tracking:** `_articleIndex`, `_article`, `_jinaTokens`, `_runId`, `_totalArticles`
- **Empty handling:** Creates a minimal prompt if there are no articles.
- **Outputs:** To `Analyze article with Claude`
- **Edge cases:** Large `fullText` still capped at 8000 chars.

#### Node: Claude Haiku (analyzer)
- **Type/role:** Anthropic Chat Model node; provides the model to the chain.
- **Model:** `claude-haiku-4-5-20251001`
- **Options:** temperature 0.2, max tokens 2048
- **Credentials:** `anthropicApi: nxsi-anthropic`
- **Connections:** Feeds the `ai_languageModel` input of `Analyze article with Claude`
- **Failure modes:** Missing/invalid API key, model availability, rate limits.

#### Node: Analyze article with Claude
- **Type/role:** LangChain LLM Chain; sends prompt and system rubric; expects JSON back.
- **System message highlights:**
  - Importance scoring guide
  - Allowed category strings list
  - Rules: JSON-only, low importance if missing text, “why it matters” must be actionable
- **Error handling:** `onError=continueRegularOutput`, `maxTries=2`, retry with 3s delay.
- **Outputs:** To `Score and rank articles`
- **Failure modes:** Non-JSON output, partial responses, timeouts, Anthropic errors.

#### Node: Score and rank articles
- **Type/role:** Code node; parses Claude output JSON, applies scoring, sorts.
- **Key logic:**
  - Extracts response text from `aiOut.text` or `aiOut.output`
  - Strips ```json fences and attempts JSON parse; fallback attempts to repair braces; final fallback uses a default “Analysis failed”.
  - Applies topic weighting (example weights):
    - `ai-models` 1.5, `ai-agents` 1.5, `ai-tools` 1.3, `security` 1.3, etc.
  - Adds social boost: `min((socialScore/200), 2)`
  - `weightedScore = importance * maxTopicWeight + socialBoost`
  - Token accounting:
    - Uses `tokenUsageEstimate` when available; otherwise estimates by character length / 4.
- **Outputs:** Single object `{ articles[], analysisInputTokens, analysisOutputTokens, jinaTokens, runId, count }` → `Select articles for digest`
- **Edge cases:** If Claude returns categories outside allowed list, weight defaults to 1.0; malformed JSON triggers fallback scoring.

---

### Block 2.7 — Digest Compilation (Claude Sonnet)
**Overview:** Selects top items for the digest, builds a compilation prompt, then uses Claude Sonnet to generate a structured digest JSON.  
**Nodes involved:**  
- `Select articles for digest`
- `Claude Sonnet (compiler)` (model provider node)
- `Compile digest with Claude`

#### Node: Select articles for digest
- **Type/role:** Code node; selects top `max_digest_articles` (default 12).
- **Digest structure logic:**
  - `lead` = first
  - `topStories` = next 4 (indexes 1–4)
  - `quickHits` = remainder (index 5+)
- **Prompt:** Requests **ONLY valid JSON** with:
  - `subject_line`, `lead_analysis`, `lead_why`,
  - `top_stories[]` (each with exactly 2-sentence summary + actionable why),
  - `quick_hits[]` one-liners,
  - `trend_note`
- **Outputs:** Single item with prompt + tracking fields → `Compile digest with Claude`
- **Edge cases:** If fewer than 6 selected, quick hits may be empty; prompt still valid.

#### Node: Claude Sonnet (compiler)
- **Type/role:** Anthropic Chat Model node (compiler model).
- **Model:** `claude-sonnet-4-5-20250929`
- **Options:** temperature 0.4, max tokens 8192
- **Credentials:** `anthropicApi: nxsi-anthropic`
- **Connections:** Feeds the `ai_languageModel` input of `Compile digest with Claude`

#### Node: Compile digest with Claude
- **Type/role:** LangChain Agent (conversational agent) to produce the final digest JSON.
- **System message constraints:** professional/approachable voice, actionable insights, exact 2-sentence top story summaries, one-line quick hits, JSON-only.
- **Retries:** `maxTries=3`, wait 5s, retry on fail enabled.
- **Outputs:** To `Format multi-channel outputs`
- **Failure modes:** Non-JSON output; overly long responses; API errors/rate limits.

---

### Block 2.8 — Formatting, Delivery, and Optional Storage
**Overview:** Converts digest JSON into Discord embeds, Slack blocks, and Markdown, estimates costs, sends messages, and optionally stores results and marks the run complete in Postgres.  
**Nodes involved:**  
- `Format multi-channel outputs`
- `Send digest to Discord`
- `Send digest to Slack`
- `Build article save query`
- `Save articles to database`
- `Mark run as completed`

#### Node: Format multi-channel outputs
- **Type/role:** Code node; “presentation + persistence payload builder”.
- **Key operations:**
  - Parses compiler output JSON (strips code fences; fallback to basic object).
  - Estimates token usage and cost:
    - Uses analysis tokens + compilation tokens
    - Uses internal pricing formula:
      - “haikuCost” = `(aIn*1 + aOut*5)/1e6`
      - “sonnetCost” = `(compIn*3 + compOut*15)/1e6`
      - **Note:** this is an estimate baked into code, not pulled from Anthropic.
  - Creates:
    - `discordEmbeds` (up to 10 embeds; lead story + up to 4 top stories + quick hits + footer)
    - `slackBlocks` (header/context/dividers/sections)
    - `markdown` digest
  - Builds `saveArticles[]` objects for DB insert.
- **Outputs:** Fan-out to:
  - `Send digest to Discord`
  - `Send digest to Slack`
  - `Build article save query`
- **Edge cases:**
  - Discord embed size limits (field lengths, total embeds); code truncates descriptions but not every field.
  - Slack block text limits (approx 3000 chars per text object); code truncates some sections.
  - JSON parse failure from compiler triggers degraded output.

#### Node: Send digest to Discord
- **Type/role:** HTTP Request; posts Discord webhook message with embeds.
- **Config choices:**
  - POST to `discord_webhook_url` from config
  - Body JSON string: `{ embeds: $json.discordEmbeds }`
  - Timeout 10s
  - `onError=continueRegularOutput`
- **Failure modes:** Invalid webhook URL, Discord rate limiting, payload too large, embed validation errors.

#### Node: Send digest to Slack
- **Type/role:** Slack node; posts a block message to a channel.
- **Config choices:**
  - Channel selected by id from config `slack_channel_id`
  - Message type: `block`
  - Blocks passed via `blocksUi` as stringified JSON `{ blocks: $json.slackBlocks }`
  - `includeLinkToWorkflow=false`
  - `onError=continueRegularOutput`
- **Credentials:** `slackApi: nxsi-slack`
- **Failure modes:** Missing scopes, invalid channel id, malformed blocks JSON, Slack rate limits.

#### Node: Build article save query
- **Type/role:** Code node; builds a bulk SQL INSERT when Postgres enabled.
- **Behavior:**
  - If `use_postgres=false`: returns `SELECT 1` and passes run data through.
  - Else: constructs `INSERT INTO digest_articles (...) VALUES (...),(... ) ON CONFLICT (url_hash) DO NOTHING`
  - Escapes strings by doubling `'` (basic SQL escaping).
  - Stores `digest_id` = `runId` (note naming mismatch vs sticky schema; see edge case)
- **Outputs:** To `Save articles to database`
- **Edge cases / integration issues:**
  - SQL is built as a single large query; can exceed max query size if many/large texts.
  - Column names in query must match your actual schema (`source_name`, `importance_score`, `digest_id` are assumed here).
  - Uses `::jsonb` casts for categories and key entities; table must be jsonb-compatible in those columns.

#### Node: Save articles to database
- **Type/role:** Postgres Execute Query; runs the generated insert.
- **alwaysOutputData:** true
- **Outputs:** To `Mark run as completed`
- **Failure modes:** Schema mismatch, large payload, DB constraints, missing `ON CONFLICT (url_hash)` unique constraint/index.

#### Node: Mark run as completed
- **Type/role:** Postgres Execute Query; writes final metrics to `digest_runs` when enabled.
- **Config choices:** If Postgres enabled, updates fields including:
  - status, feeds_checked (=10), articles_found, articles_after_dedup, articles_selected,
  - lead_story_title, cost tokens, cost USD, jina tokens, completed_at, duration_seconds
- **alwaysOutputData:** true
- **Edge cases:** Your `digest_runs` table must contain all referenced columns (`feeds_checked`, `articles_after_dedup`, `lead_story_title`, `jina_tokens_used`, etc.). The sticky schema provided in the workflow note is **not fully aligned** with these update fields.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run daily at 6 AM | scheduleTrigger | Daily entry point | — | ⚙️ Configure digest settings | # AI News Digest (overview block) |
| ⚙️ Configure digest settings | set | Central runtime config | Run daily at 6 AM | Initialize run timestamp | # AI News Digest (overview block) ; ## Configuration — All user settings… [Discord webhook setup](https://support.discord.com/hc/en-us/articles/228383668) \| [Slack webhook guide](https://api.slack.com/messaging/webhooks) |
| Initialize run timestamp | code | Create run date/time + runId | ⚙️ Configure digest settings | Create run record | ## Run Tracking — Creates a timestamped database record… |
| Create run record | postgres | Optional DB run row insert / fallback select | Initialize run timestamp | Build feed source list | ## Run Tracking — Creates a timestamped database record… ; ## Optional: PostgreSQL (tables/schema note) |
| Build feed source list | code | Define sources (RSS/Atom/Reddit/HN) | Create run record | Fetch feed content | ## Feed Collection — Fetches and parses articles… |
| Fetch feed content | httpRequest | Download raw feed content | Build feed source list | Parse articles from feeds | ## Feed Collection — Fetches and parses articles… |
| Parse articles from feeds | code | Parse + normalize items to articles | Fetch feed content | Any articles found? | ## Feed Collection — Fetches and parses articles… |
| Any articles found? | if | Branch on empty result | Parse articles from feeds | Deduplicate by URL and title (true), Mark run as empty (false) | ## Deduplication — Three layers… |
| Mark run as empty | postgres | Optional DB update (no articles) | Any articles found? (false) | — | ## Deduplication — Three layers… ; ## Run Tracking — Optional Postgres… |
| Deduplicate by URL and title | code | In-run URL hash + fuzzy title dedup | Any articles found? (true) | Check database for recent duplicates | ## Deduplication — Three layers… |
| Check database for recent duplicates | postgres | Optional cross-run dedup lookup | Deduplicate by URL and title | Remove already-seen articles | ## Deduplication — Three layers… ; ## Optional: PostgreSQL (tables/schema note) |
| Remove already-seen articles | code | Filter out hashes returned from DB | Check database for recent duplicates | Select top articles for extraction | ## Deduplication — Three layers… |
| Select top articles for extraction | code | Pick top N for full-text extraction | Remove already-seen articles | Extract full text via Jina | ## Content Extraction — Pulls full article text via [Jina Reader API](https://jina.ai/reader/)… |
| Extract full text via Jina | httpRequest | Jina Reader extraction | Select top articles for extraction | Assemble extracted content | ## Content Extraction — Pulls full article text via [Jina Reader API](https://jina.ai/reader/)… |
| Assemble extracted content | code | Merge full text + metadata | Extract full text via Jina | Prepare analysis prompts | ## Content Extraction — Pulls full article text via [Jina Reader API](https://jina.ai/reader/)… |
| Prepare analysis prompts | code | Build per-article LLM prompts | Assemble extracted content | Analyze article with Claude | ## AI Analysis — Claude Haiku scores each article… |
| Claude Haiku (analyzer) | lmChatAnthropic | Model provider for analysis | — | Analyze article with Claude (ai_languageModel) | ⚠️ Requires Anthropic API key… [console.anthropic.com](https://console.anthropic.com) … [nxsi.io/guides/claude-api-setup](https://nxsi.io/guides/claude-api-setup) ; ## AI Analysis — Claude Haiku… |
| Analyze article with Claude | chainLlm | Run analysis chain (JSON output) | Prepare analysis prompts | Score and rank articles | ⚠️ Requires Anthropic API key… ; ## AI Analysis — Claude Haiku… |
| Score and rank articles | code | Parse JSON + weighted scoring | Analyze article with Claude | Select articles for digest | ## AI Analysis — Claude Haiku… |
| Select articles for digest | code | Choose lead/top/quick hits + compiler prompt | Score and rank articles | Compile digest with Claude | ## Digest Compilation — Claude Sonnet compiles… |
| Claude Sonnet (compiler) | lmChatAnthropic | Model provider for compilation | — | Compile digest with Claude (ai_languageModel) | ## Digest Compilation — Claude Sonnet compiles… |
| Compile digest with Claude | agent | Compile final digest JSON | Select articles for digest | Format multi-channel outputs | ## Digest Compilation — Claude Sonnet compiles… |
| Format multi-channel outputs | code | Build Discord/Slack/Markdown + cost + DB payload | Compile digest with Claude | Send digest to Discord; Send digest to Slack; Build article save query | ## Delivery & Storage — Sends formatted digest… |
| Send digest to Discord | httpRequest | Deliver embeds via Discord webhook | Format multi-channel outputs | — | ## Delivery & Storage — Sends formatted digest… |
| Send digest to Slack | slack | Deliver blocks to Slack channel | Format multi-channel outputs | — | ## Delivery & Storage — Sends formatted digest… |
| Build article save query | code | Build bulk INSERT SQL (optional) | Format multi-channel outputs | Save articles to database | ## Delivery & Storage — Sends formatted digest… ; ## Optional: PostgreSQL (tables/schema note) |
| Save articles to database | postgres | Execute INSERT query | Build article save query | Mark run as completed | ## Delivery & Storage — Sends formatted digest… ; ## Optional: PostgreSQL (tables/schema note) |
| Mark run as completed | postgres | Update run metrics/status | Save articles to database | — | ## Delivery & Storage — Sends formatted digest… ; ## Run Tracking — Optional Postgres… |
| Overview — AI News Digest | stickyNote | Documentation | — | — |  |
| Section — Configuration | stickyNote | Documentation | — | — |  |
| Section — Run Tracking | stickyNote | Documentation | — | — |  |
| Section — Feed Collection | stickyNote | Documentation | — | — |  |
| Section — Deduplication | stickyNote | Documentation | — | — |  |
| Section — Content Extraction | stickyNote | Documentation | — | — |  |
| Section — AI Analysis | stickyNote | Documentation | — | — |  |
| Section — Digest Compilation | stickyNote | Documentation | — | — |  |
| Section — Delivery and Storage | stickyNote | Documentation | — | — |  |
| Warning — Anthropic API Cost | stickyNote | Documentation | — | — |  |
| Sticky Note | stickyNote | Documentation (Postgres schema) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
1. Add node **Schedule Trigger** named `Run daily at 6 AM`
2. Set cron to: `0 6 * * *`

2) **Add Configuration Node**
1. Add **Set** node named `⚙️ Configure digest settings`
2. Add fields:
   - `discord_webhook_url` (string): your Discord webhook URL
   - `slack_channel_id` (string): target Slack channel ID
   - `max_extract_articles` (number): 25
   - `max_digest_articles` (number): 12
   - `dedup_lookback_days` (number): 7
   - `feed_lookback_hours` (number): 24
   - `use_postgres` (boolean): false (set true only if you create DB tables + credentials)
3. Connect: `Run daily at 6 AM` → `⚙️ Configure digest settings`

3) **Create Run Timestamp**
1. Add **Code** node named `Initialize run timestamp`
2. Paste logic that outputs:
   - `runDate` = ISO date `YYYY-MM-DD`
   - `startedAt` = ISO timestamp
   - `runId` = random GUID-like string
3. Connect: `⚙️ Configure digest settings` → `Initialize run timestamp`

4) **Optional: Postgres Run Record**
1. Add **Postgres** node named `Create run record` (Operation: Execute Query)
2. Configure Postgres credentials (host/db/user/password or n8n credential).
3. Use an expression-based query:
   - If `use_postgres=true`: insert into `digest_runs` returning `id`
   - Else: select a synthetic row using `runId`
4. Connect: `Initialize run timestamp` → `Create run record`

5) **Build Feed List**
1. Add **Code** node `Build feed source list`
2. Create an array of feed objects with:
   - `url`, `name`, `category`, `feedType` (hn_api/rss/atom/reddit)
3. Add `runId` = the id returned from `Create run record`
4. Output one item per feed
5. Connect: `Create run record` → `Build feed source list`

6) **Fetch Feeds**
1. Add **HTTP Request** node `Fetch feed content`
2. URL: `{{$json.url}}`
3. Response format: **Text**
4. Timeout: 15000 ms
5. Header: `User-Agent: AI-News-Digest/1.0`
6. Set **On Error**: “Continue”
7. Connect: `Build feed source list` → `Fetch feed content`

7) **Parse Feeds Into Articles**
1. Add **Code** node `Parse articles from feeds`
2. Implement parsing for:
   - HN JSON hits
   - Reddit JSON children
   - Atom `<entry>` parsing
   - RSS `<item>` parsing
3. Filter by `feed_lookback_hours`
4. Output one item per normalized article (or sentinel with empty url)
5. Connect: `Fetch feed content` → `Parse articles from feeds`

8) **Branch If No Articles**
1. Add **IF** node `Any articles found?`
2. Condition: String `{{$json.url}}` is not empty
3. Connect: `Parse articles from feeds` → `Any articles found?`

9) **Optional: Mark Empty Run**
1. Add **Postgres** node `Mark run as empty` (Execute Query)
2. Query expression:
   - If enabled: update `digest_runs` status to `no_articles`
   - Else: `SELECT 1`
3. Connect: `Any articles found?` (false) → `Mark run as empty`

10) **Deduplicate Within Run**
1. Add **Code** node `Deduplicate by URL and title`
2. Implement:
   - djb2 URL hash (`urlHash`)
   - title normalization + Levenshtein fuzzy threshold
   - Output a single item `{ articles:[...], hashListSql:'...', runId }`
3. Connect: `Any articles found?` (true) → `Deduplicate by URL and title`

11) **Optional: Cross-run Dedup in Postgres**
1. Add **Postgres** node `Check database for recent duplicates`
2. Query expression:
   - If enabled: `SELECT url_hash FROM digest_articles WHERE url_hash IN (...) AND created_at > NOW() - INTERVAL '<dedup_lookback_days> days'`
   - Else: empty select
3. Connect: `Deduplicate by URL and title` → `Check database for recent duplicates`

12) **Filter Already-Seen**
1. Add **Code** node `Remove already-seen articles`
2. Build a set of returned `url_hash` and filter `articles[]`
3. Output `{ articles:fresh[], count, runId }`
4. Connect: `Check database for recent duplicates` → `Remove already-seen articles`

13) **Select Articles for Full-Text Extraction**
1. Add **Code** node `Select top articles for extraction`
2. Sort by `socialScore`, take top `max_extract_articles`
3. Output one item per article
4. Connect: `Remove already-seen articles` → `Select top articles for extraction`

14) **Extract Full Text (Jina)**
1. Add **HTTP Request** node `Extract full text via Jina`
2. URL: `https://r.jina.ai/{{$json.url}}`
3. Response format: JSON
4. Headers:
   - `Accept: application/json`
   - `X-Return-Format: markdown`
5. Timeout: 30000 ms
6. On Error: Continue
7. Connect: `Select top articles for extraction` → `Extract full text via Jina`

15) **Assemble Extracted Content**
1. Add **Code** node `Assemble extracted content`
2. Merge Jina result (markdown content) into each article as `fullText` (truncate to ~8000 chars)
3. Output single `{ articles:[...], jinaTokens, runId }`
4. Connect: `Extract full text via Jina` → `Assemble extracted content`

16) **Prepare Per-Article Analysis Prompts**
1. Add **Code** node `Prepare analysis prompts`
2. Output one item per article with:
   - `prompt` (JSON-only instruction)
   - `_article` metadata
3. Connect: `Assemble extracted content` → `Prepare analysis prompts`

17) **Configure Anthropic Credentials**
1. In n8n Credentials, create **Anthropic API** credential (your API key from https://console.anthropic.com).
2. Create **lmChatAnthropic** model node `Claude Haiku (analyzer)`:
   - Model: `claude-haiku-4-5-20251001`
   - Temperature: 0.2
   - Max tokens: 2048

18) **Analyze Each Article (LangChain Chain)**
1. Add node **LLM Chain** (LangChain) named `Analyze article with Claude`
2. Set input text to: `{{$json.prompt}}`
3. Add system message with scoring rubric and JSON-only rules (as in workflow)
4. Connect model: `Claude Haiku (analyzer)` → `Analyze article with Claude` via **ai_languageModel**
5. Connect: `Prepare analysis prompts` → `Analyze article with Claude`

19) **Score & Rank**
1. Add **Code** node `Score and rank articles`
2. Parse model output into JSON; compute weighted score (topic weights + social boost)
3. Output `{ articles:[...], analysisInputTokens, analysisOutputTokens, jinaTokens, runId }`
4. Connect: `Analyze article with Claude` → `Score and rank articles`

20) **Select Articles for Digest + Create Compilation Prompt**
1. Add **Code** node `Select articles for digest`
2. Use `max_digest_articles`; create lead/top/quick hits sections; build compiler prompt requiring JSON-only.
3. Connect: `Score and rank articles` → `Select articles for digest`

21) **Compiler Model + Agent**
1. Add **lmChatAnthropic** node `Claude Sonnet (compiler)`:
   - Model: `claude-sonnet-4-5-20250929`
   - Temperature: 0.4
   - Max tokens: 8192
2. Add **LangChain Agent** node `Compile digest with Claude`
   - Text: `{{$json.prompt}}`
   - System message enforcing structure/voice + JSON-only
3. Connect: `Claude Sonnet (compiler)` → `Compile digest with Claude` via **ai_languageModel**
4. Connect: `Select articles for digest` → `Compile digest with Claude`

22) **Format for Channels**
1. Add **Code** node `Format multi-channel outputs`
2. Parse compiler JSON; build:
   - `discordEmbeds` array
   - `slackBlocks` array
   - `markdown`
   - cost estimates and `saveArticles[]`
3. Connect: `Compile digest with Claude` → `Format multi-channel outputs`

23) **Send to Discord**
1. Add **HTTP Request** node `Send digest to Discord`
2. POST to `{{$('⚙️ Configure digest settings').first().json.discord_webhook_url}}`
3. Body: `{ "embeds": $json.discordEmbeds }`
4. On Error: Continue
5. Connect: `Format multi-channel outputs` → `Send digest to Discord`

24) **Send to Slack**
1. Add **Slack** node `Send digest to Slack`
2. Credentials: Slack OAuth token/app (`chat:write` etc.)
3. Channel: use id expression from config
4. Send message as blocks using `$json.slackBlocks`
5. On Error: Continue
6. Connect: `Format multi-channel outputs` → `Send digest to Slack`

25) **Optional: Save Articles + Mark Run Completed (Postgres)**
1. Add **Code** node `Build article save query`
   - If `use_postgres=false`, return `SELECT 1`
   - Else build a bulk insert into your `digest_articles`
2. Add **Postgres** node `Save articles to database` executing `{{$json.query}}`
3. Add **Postgres** node `Mark run as completed` updating your `digest_runs`
4. Connect:  
   `Format multi-channel outputs` → `Build article save query` → `Save articles to database` → `Mark run as completed`

**Important schema note:** The workflow’s Postgres UPDATE/INSERT statements expect specific columns (some beyond the sticky schema). Ensure your tables include all referenced fields or adjust the SQL to match your schema.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Jina Reader API used for full-text extraction (markdown output). | https://jina.ai/reader/ |
| Discord webhook setup reference. | https://support.discord.com/hc/en-us/articles/228383668 |
| Slack messaging/webhooks reference (workflow uses Slack node, not incoming webhook). | https://api.slack.com/messaging/webhooks |
| Anthropic API key creation instructions. | https://console.anthropic.com |
| External guide referenced in notes for Anthropic setup. | https://nxsi.io/guides/claude-api-setup |
| Optional Postgres is supported; nodes are designed to no-op when `use_postgres=false`. | Covered in “Optional: PostgreSQL” sticky note content in the workflow |
| Postgres schema in sticky note may not fully match the SQL used in DB nodes (additional columns are referenced in updates/inserts). | Review and reconcile `digest_runs` and `digest_articles` columns before enabling `use_postgres=true` |