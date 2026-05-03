Discover viral social media trends with Gemini Flash & Apify scraping

https://n8nworkflows.xyz/workflows/discover-viral-social-media-trends-with-gemini-flash---apify-scraping-12178


# Discover viral social media trends with Gemini Flash & Apify scraping

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow detects potentially viral topics by combining **Google Trends RSS** (search intent) with **real social engagement signals** scraped from **TikTok, Instagram, and X (Twitter)** via **Apify actors**, then asks **Gemini Flash** (via the n8n AI Agent) to synthesize multi-platform trend intelligence and posts per-topic alerts to **Discord**.

**Target use cases:**
- Daily “what’s trending + why” intelligence for content teams
- Early detection of cross-platform momentum (search + social)
- Automated, per-topic reporting into a team channel

### Logical blocks
**1.1 Scheduling & region input**  
Runs daily and defines which Google Trends region (geo) to pull.

**1.2 Google Trends acquisition & normalization**  
Fetches Google Trends RSS and converts XML into structured JSON trends.

**1.3 Social scraping (Apify) across platforms**  
Uses the top Google Trends keywords to query TikTok/Instagram/X and returns raw datasets.

**1.4 Platform output shaping & data aggregation**  
Normalizes each platform’s dataset into a compact summary, merges them, and produces a single merged JSON text for LLM input.

**1.5 AI synthesis with Gemini Flash**  
AI Agent prompts Gemini to infer topics, summarize discussion, estimate engagement, and assign trend strength with a strict JSON schema.

**1.6 Parsing, splitting, and Discord delivery**  
Parses the AI output JSON, splits topics into items, and posts formatted messages to Discord.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & region input

**Overview:** Triggers the workflow on a schedule and sets the target region used by Google Trends RSS.

**Nodes involved:**  
- Schedule Trigger  
- Edit Fields

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point; time-based execution.
- **Configuration (interpreted):** Runs daily at **08:00** (server timezone).
- **Connections:**  
  - Output → **Edit Fields**
- **Edge cases / failures:**  
  - Timezone mismatch: “8 AM” is n8n server timezone, not necessarily user locale.
  - If n8n is asleep/down at trigger time, run may be skipped depending on instance behavior.

#### Node: Edit Fields
- **Type / role:** `n8n-nodes-base.set` — defines configuration data for downstream nodes.
- **Configuration:** Raw JSON output:
  - `region: "US"`
- **Key expressions/variables:** None (static JSON).
- **Connections:**  
  - Input ← Schedule Trigger  
  - Output → Fetch Google Trends RSS
- **Edge cases / failures:**  
  - Invalid region code will still form a URL but may yield empty/changed RSS output.
  - Whitespace/casing is later normalized in the HTTP node.

---

### 2.2 Google Trends acquisition & normalization

**Overview:** Pulls Google Trends RSS for a region and converts the RSS XML to a structured JSON list of trends and related news.

**Nodes involved:**  
- Fetch Google Trends RSS  
- Format Google Output

#### Node: Fetch Google Trends RSS
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches RSS XML from Google Trends.
- **Configuration:**
  - URL expression:  
    `https://trends.google.com/trending/rss?geo={{ $json.region.toUpperCase().trim() }}`
- **Input/Output:**  
  - Input includes `region` from **Edit Fields**.  
  - Output is the HTTP response (node later expects XML in `items[0].json.data`).
- **Connections:**  
  - Input ← Edit Fields  
  - Output → Format Google Output
- **Edge cases / failures:**
  - Google may rate-limit or block requests (HTTP 429/503).
  - Response shape: The next Code node assumes `items[0].json.data` contains XML; depending on n8n HTTP Request settings/version, the body might be in a different field (e.g., `body`). If so, parsing will fail or produce empty results.
  - RSS schema changes can break regex-based extraction.

#### Node: Format Google Output
- **Type / role:** `n8n-nodes-base.code` — parses RSS XML into normalized JSON.
- **Configuration choices:**
  - Uses regex helpers to extract tags like `<title>`, `<ht:approx_traffic>`, `<pubDate>`, `<ht:picture>`, and repeated `<ht:news_item>` blocks.
  - Produces output:
    - `source: "Google Trends"`
    - `geo: $('Edit Fields').first().json.region`
    - `generated_at: ISO timestamp`
    - `trends: [{ keyword, approx_traffic, published_at, image, news[] }]`
- **Key expressions/variables:**
  - Reads region via: `$('Edit Fields').first().json.region`
  - Reads XML via: `items[0].json.data`
- **Connections:**  
  - Input ← Fetch Google Trends RSS  
  - Output → TikTok Scrapper, Instagram Scrapper, X Scrapper (fan-out)
- **Edge cases / failures:**
  - **Fragile XML parsing**: Regex can break on CDATA, namespaces, formatting, or unexpected tag nesting.
  - If RSS contains no `<item>` blocks, `trends` will be `[]` and scrapers will query with empty lists.
  - `published_at` conversion uses `new Date(d)`—may yield `Invalid Date` if format changes.

---

### 2.3 Social scraping (Apify) across platforms

**Overview:** Uses the top 5 Google Trends keywords to query Apify actors for TikTok videos, Instagram hashtag posts, and X tweets.

**Nodes involved:**  
- TikTok Scrapper  
- Instagram Scrapper  
- X Scrapper

#### Node: TikTok Scrapper
- **Type / role:** `@apify/n8n-nodes-apify.apify` — runs an Apify actor and returns its dataset items.
- **Configuration:**
  - Actor: **TikTok Data Extractor (clockworks/free-tiktok-scraper)** (`OtzYfK1ndEGdwWFKQ`)
  - Operation: “Run actor and get dataset”
  - Custom JSON body (expression-built):
    - `searchQueries`: top 5 trend keywords  
      `{{ JSON.stringify($json.trends.slice(0, 5).map(t => t.keyword)) }}`
    - `resultsPerPage: 5`
    - `searchType: "video"`
    - `sortType: "most-liked"`
    - Downloads disabled (videos/covers)
- **Credentials:** Apify API token (`apifyApi Credential`)
- **Connections:**  
  - Input ← Format Google Output  
  - Output → Tiktok Output
- **Edge cases / failures:**
  - Actor may return different fields depending on upstream changes.
  - If `trends` is empty, `searchQueries` becomes `[]` (may yield empty dataset or actor error depending on actor validation).
  - Network timeouts / Apify rate limits / actor run failures.
- **Version-specific notes:** Apify node v1; behavior depends on n8n Apify integration version.

#### Node: Instagram Scrapper
- **Type / role:** `@apify/n8n-nodes-apify.apify` — runs Instagram hashtag scraping actor.
- **Configuration:**
  - Actor: **Instagram Free Hashtag Scraper (scrapesmith/instagram-free-hashtag-scraper)** (`7YASHOuDdgJuPjqsP`)
  - Custom JSON body:
    - `hashtags`: top 5 keywords with whitespace removed  
      `t.keyword.replace(/\s+/g, '')`
    - `resultsLimit: 5`
    - `enhanceUserSearchWithWildcard: true`
- **Credentials:** Apify API token
- **Connections:**  
  - Input ← Format Google Output  
  - Output → Instagram Output
- **Edge cases / failures:**
  - Hashtag normalization may produce invalid/odd tags (e.g., punctuation, non-Latin).
  - Actor results may omit `likesCount` etc.; downstream tries fallbacks.
  - Platform scraping volatility (Instagram changes can degrade actor quality).

#### Node: X Scrapper
- **Type / role:** `@apify/n8n-nodes-apify.apify` — runs tweet scraping actor.
- **Configuration:**
  - Actor: **Tweet Scraper Pay-Per Result** (`CJdippxWmn9uRfooo`)
  - Custom JSON body:
    - `searchTerms`: top 5 keywords
    - `maxItems: 10`
    - `sort: "Latest"`
    - `includeUserInfo: true`
- **Credentials:** Apify API token
- **Connections:**  
  - Input ← Format Google Output  
  - Output → X Output
- **Edge cases / failures:**
  - Paid actor: cost implications; may fail if account balance limits.
  - Output fields can vary (`full_text` vs `text`, `view_count` may be missing).
  - Keyword ambiguity can return noisy “Latest” results.

---

### 2.4 Platform output shaping & data aggregation

**Overview:** Converts each platform dataset into a compact summary, merges the three platform summaries, and serializes them into a single text blob for the AI Agent prompt.

**Nodes involved:**  
- Tiktok Output  
- Instagram Output  
- X Output  
- Merge  
- Code in JavaScript

#### Node: Tiktok Output
- **Type / role:** `n8n-nodes-base.code` — maps TikTok dataset items into a summary.
- **Configuration:**
  - Reads all input items from Apify dataset.
  - Determines `keyword` from `items[0]?.json.searchQuery` (fallback `"Trending"`).
  - Outputs:
    - `platform: "tiktok"`
    - `keyword`
    - `summary`: top 5 entries containing `caption`, `views` (`playCount`), `likes` (`diggCount`)
- **Connections:**  
  - Input ← TikTok Scrapper  
  - Output → Merge (input index 0)
- **Edge cases / failures:**
  - If Apify output doesn’t include `searchQuery`, keyword fallback becomes generic.
  - If fields differ (`text` vs `caption`), caption may be undefined (code tries both).
  - Views/likes default to 0 if absent.

#### Node: Instagram Output
- **Type / role:** `n8n-nodes-base.code` — normalizes Instagram hashtag results.
- **Configuration:**
  - `keyword` from `items[0]?.json.query` fallback `"Trending"`.
  - Summary fields:
    - `caption`
    - `likes` from `likesCount` OR `edge_liked_by?.count` OR 0
  - Takes top 5.
- **Connections:**  
  - Input ← Instagram Scrapper  
  - Output → Merge (input index 1)
- **Edge cases / failures:**
  - Captions can be empty or missing.
  - Like counts vary by actor schema; fallbacks help but are not exhaustive.

#### Node: X Output
- **Type / role:** `n8n-nodes-base.code` — normalizes tweet items.
- **Configuration:**
  - `keyword` from `items[0]?.json.keyword` fallback `"Trending"`.
  - Summary fields:
    - `text` from `full_text` or `text`
    - `views` from `view_count` (default 0)
    - `likes` from `favorite_count` (default 0)
  - Takes top 5.
- **Connections:**  
  - Input ← X Scrapper  
  - Output → Merge (input index 2)
- **Edge cases / failures:**
  - Actor may use different field names for metrics (e.g., `likeCount`).
  - View count often missing depending on data source/API.

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines 3 inputs into one stream.
- **Configuration:**
  - `numberInputs: 3`
  - Effectively waits for all three branches then outputs combined items (one per incoming item, depending on merge behavior).
- **Connections:**  
  - Inputs ← Tiktok Output, Instagram Output, X Output  
  - Output → Code in JavaScript
- **Edge cases / failures:**
  - If any branch produces zero items, merge behavior can lead to empty output or missing data depending on n8n merge mode defaults. Here it relies on the node’s multi-input merge semantics; ensure all three produce at least one item.
  - If scrapers fail hard (node error), workflow may stop unless “continue on fail” is enabled (note: sticky note claims resilience, but these nodes are configured with `retryOnFail`, not explicit “continue on fail”).

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — serializes merged items into a compact LLM input string.
- **Configuration:**
  - Uses `$items()` to collect all merged items.
  - Outputs a single item with:
    - `merged_data_text: JSON.stringify(items.map(i => i.json), null, 2)`
- **Connections:**  
  - Input ← Merge  
  - Output → AI Agent
- **Edge cases / failures:**
  - Large datasets can exceed token limits in the AI model; current workflow already truncates to top 5 per platform, mitigating risk.
  - If merge output is empty, `merged_data_text` becomes `"[]"`, leading to low-signal AI results.

---

### 2.5 AI synthesis with Gemini Flash

**Overview:** Uses an AI Agent with a strict JSON output schema to infer topics, summarize discussion, evaluate engagement, and rate trend strength.

**Nodes involved:**  
- AI Agent  
- Google Gemini Chat Model

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM backend to the AI Agent.
- **Configuration:**
  - Model: `models/gemini-flash-latest`
- **Credentials:** `googlePalmApi Credential` (Gemini API key)
- **Connections:**  
  - AI language model output → AI Agent (ai_languageModel connection)
- **Edge cases / failures:**
  - Invalid key / billing disabled → auth errors.
  - Model name changes/deprecation → request failures.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model, returns model output.
- **Configuration choices:**
  - Prompt includes the merged social data:
    - `{{ $json.merged_data_text }}`
  - System message enforces **strict JSON output** with exact keys:
    - Root key must be `topics` (array)
    - Each topic must include: `topic_name`, `discussion_summary`, `engagement_signals { total_estimated_views, total_estimated_likes }`, `trend_strength`, `insight`, `content_recommendation`
  - `retryOnFail: true`
- **Connections:**  
  - Input ← Code in JavaScript  
  - Output → Parse Output
- **Edge cases / failures:**
  - Model may still output non-JSON or partial JSON; downstream Parse Output attempts cleanup.
  - Engagement totals are “estimated” by the model and may be inconsistent with input metrics unless carefully constrained.
  - If Gemini returns tool/agent metadata instead of `output` text in `item.json.output`, Parse Output may fail (depends on AI Agent node output structure in your n8n version).

---

### 2.6 Parsing, splitting, and Discord delivery

**Overview:** Parses the AI response into JSON, splits each topic into its own item, and sends formatted reports to Discord via webhook.

**Nodes involved:**  
- Parse Output  
- Split Out  
- Discord

#### Node: Parse Output
- **Type / role:** `n8n-nodes-base.code` — converts AI text output into JSON items.
- **Configuration:**
  - Reads `item.json.output` as the text payload.
  - Strips markdown fences ```json / ``` then `JSON.parse`.
  - On parse failure, outputs:
    - `{ error: true, raw_output: text }`
- **Connections:**  
  - Input ← AI Agent  
  - Output → Split Out
- **Edge cases / failures:**
  - If AI Agent output field name differs (not `output`), parsing will produce `undefined` and fail.
  - If valid JSON but not matching schema, Split Out may fail (missing `topics`).

#### Node: Split Out
- **Type / role:** `n8n-nodes-base.splitOut` — splits an array field into multiple items.
- **Configuration:**
  - `fieldToSplitOut: "topics"`
- **Connections:**  
  - Input ← Parse Output  
  - Output → Discord
- **Edge cases / failures:**
  - If Parse Output returned `{error:true...}` without `topics`, this node will error or output nothing.
  - If `topics` is not an array, split will fail.

#### Node: Discord
- **Type / role:** `n8n-nodes-base.discord` — sends a message to Discord using a webhook.
- **Authentication:** Webhook (`discordWebhookApi Credential`)
- **Configuration:**
  - Message template (Indonesian labels) using fields from each topic item:
    - `{{ $json.topic_name }}`
    - `{{ $json.discussion_summary }}`
    - Views/Likes line uses:  
      `{{ $json.engagement_signals.total_estimated_views || $json.engagement_signals.total_estimated_likes || '0' }}`
    - `{{ $json.insight }}`
    - `{{ $json.content_recommendation }}`
- **Connections:**  
  - Input ← Split Out
- **Edge cases / failures:**
  - If engagement_signals missing, the expression may throw unless `engagement_signals` exists; schema enforcement should ensure it, but parse-fail cases won’t.
  - Discord webhook revoked/invalid → 401/404.
  - Rate limits if many topics; Discord may throttle.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled workflow entry point | — | Edit Fields | ### **AI-Driven Social Intelligence Agent** / This workflow is a professional-grade market intelligence tool that automatically synchronizes search trends with real-world social media activity to identify high-potential viral content. / ### **How it works** / 1. **Schedule Trigger:** Automatically runs at your preferred interval (e.g., every morning) to fetch fresh market data. / 2. **Multi-Source Acquisition:** - **Google Trends:** Monitors regional RSS feeds to capture rising search interests. / - **Social Media Scrapers:** Uses **Apify** actors to safely extract trending posts from TikTok, Instagram, and X (Twitter) without account risk. / 3. **Data Aggregation:** Custom JavaScript logic merges disparate platform data into a unified context, optimizing AI token efficiency. / 4. **AI Synthesis & Intelligence:** **Google Gemini** cross-references the data to identify "The Common Thread"—topics trending across multiple platforms simultaneously. / 5. **Granular Delivery:** Uses a **Split Out** mechanism to deliver each trend as an individual, readable report to **Discord**. / ### **Setup steps** / 1. **API Keys:** Provide your **Apify API Token** and **Google Gemini API Key**. / 2. **Region Configuration:** Set your target country code (e.g., `JP`, `ID`, `US`) in the **"Edit Fields"** node. / 3. **Discord Integration:** Paste your Discord Webhook URL into the **Discord** node. / 4. **Resilience:** All scrapers are pre-configured with "Continue on Fail" to ensure report delivery even if one source is down. / ### **Requirements** / - n8n instance. / - Apify Account (for Social Media scraping). / - Google Gemini API Key. / - Discord Server (for receiving reports). |
| Edit Fields | set | Defines region config | Schedule Trigger | Fetch Google Trends RSS | (same as above) |
| Fetch Google Trends RSS | httpRequest | Pulls Google Trends RSS XML | Edit Fields | Format Google Output | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| Format Google Output | code | Parses RSS XML into structured trends JSON | Fetch Google Trends RSS | TikTok Scrapper; Instagram Scrapper; X Scrapper | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| TikTok Scrapper | apify | Scrapes TikTok data for top trend keywords | Format Google Output | Tiktok Output | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| Tiktok Output | code | Normalizes TikTok dataset to compact summary | TikTok Scrapper | Merge | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| Instagram Scrapper | apify | Scrapes Instagram hashtag results for top keywords | Format Google Output | Instagram Output | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| Instagram Output | code | Normalizes Instagram dataset to compact summary | Instagram Scrapper | Merge | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| X Scrapper | apify | Scrapes X/Twitter data for top keywords | Format Google Output | X Output | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| X Output | code | Normalizes X dataset to compact summary | X Scrapper | Merge | ## **1. Multi-Platform Data Fetching** / Extracts raw trending data from Google Search (via RSS) and deep-scrapes TikTok, Instagram, and X using Apify's robust industrial actors. |
| Merge | merge | Combines 3 platform summaries | Tiktok Output; Instagram Output; X Output | Code in JavaScript | ## **2. AI Synthesis & Trend Intelligence** / Aggregates all raw social data and uses LLMs (Gemini) to cross-reference topics, removing noise and identifying the "why" behind the trend. |
| Code in JavaScript | code | Serializes merged summaries into LLM input text | Merge | AI Agent | ## **2. AI Synthesis & Trend Intelligence** / Aggregates all raw social data and uses LLMs (Gemini) to cross-reference topics, removing noise and identifying the "why" behind the trend. |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM provider (Gemini Flash) | — | AI Agent (ai_languageModel) | ## **2. AI Synthesis & Trend Intelligence** / Aggregates all raw social data and uses LLMs (Gemini) to cross-reference topics, removing noise and identifying the "why" behind the trend. |
| AI Agent | agent | Produces structured trend intelligence JSON | Code in JavaScript; Google Gemini Chat Model | Parse Output | ## **2. AI Synthesis & Trend Intelligence** / Aggregates all raw social data and uses LLMs (Gemini) to cross-reference topics, removing noise and identifying the "why" behind the trend. |
| Parse Output | code | Parses AI output string into JSON | AI Agent | Split Out | ## **3. Granular Reporting & Alerting** / Converts complex AI analysis into readable formats and sends instant, per-topic alerts to your team via Discord Webhooks. |
| Split Out | splitOut | Splits `topics[]` into per-topic items | Parse Output | Discord | ## **3. Granular Reporting & Alerting** / Converts complex AI analysis into readable formats and sends instant, per-topic alerts to your team via Discord Webhooks. |
| Discord | discord | Sends per-topic messages to Discord | Split Out | — | ## **3. Granular Reporting & Alerting** / Converts complex AI analysis into readable formats and sends instant, per-topic alerts to your team via Discord Webhooks. |
| Sticky Note | stickyNote | Documentation note | — | — | ### **AI-Driven Social Intelligence Agent** / (content as written in the note) |
| Sticky Note1 | stickyNote | Documentation note | — | — | ## **1. Multi-Platform Data Fetching** / (content as written in the note) |
| Sticky Note2 | stickyNote | Documentation note | — | — | ## **2. AI Synthesis & Trend Intelligence** / (content as written in the note) |
| Sticky Note3 | stickyNote | Documentation note | — | — | ## **3. Granular Reporting & Alerting** / (content as written in the note) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set it to run **daily at 08:00** (adjust timezone as needed).

2) **Create “Edit Fields” (Set node)**
   - Mode: **Raw**
   - JSON:
     ```json
     { "region": "US" }
     ```
   - Connect: **Schedule Trigger → Edit Fields**

3) **Create “Fetch Google Trends RSS” (HTTP Request node)**
   - Method: GET
   - URL (expression):
     - `https://trends.google.com/trending/rss?geo={{ $json.region.toUpperCase().trim() }}`
   - Connect: **Edit Fields → Fetch Google Trends RSS**

4) **Create “Format Google Output” (Code node, JavaScript)**
   - Paste logic that:
     - Reads XML from the HTTP response field (in this workflow: `items[0].json.data`)
     - Extracts `<item>` blocks and key tags (`title`, `pubDate`, `ht:approx_traffic`, `ht:picture`, etc.)
     - Outputs `{ source, geo, generated_at, trends: [...] }`
   - Connect: **Fetch Google Trends RSS → Format Google Output**
   - Note: If your HTTP node stores response body in `body` instead of `data`, update the code accordingly.

5) **Create Apify credentials**
   - In n8n Credentials: add **Apify API** credential with your Apify token.

6) **Create “TikTok Scrapper” (Apify node)**
   - Operation: **Run actor and get dataset**
   - Actor: `clockworks/free-tiktok-scraper` (ID `OtzYfK1ndEGdwWFKQ`)
   - Body (expression) includes:
     - `searchQueries`: top 5 keywords from `$json.trends`
     - sort “most-liked”, resultsPerPage 5, downloads false
   - Select Apify credential
   - Connect: **Format Google Output → TikTok Scrapper**

7) **Create “Instagram Scrapper” (Apify node)**
   - Actor: `scrapesmith/instagram-free-hashtag-scraper` (ID `7YASHOuDdgJuPjqsP`)
   - Body: hashtags from top 5 keywords, whitespace removed; resultsLimit 5
   - Connect: **Format Google Output → Instagram Scrapper**

8) **Create “X Scrapper” (Apify node)**
   - Actor: `kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest` (ID `CJdippxWmn9uRfooo`)
   - Body: searchTerms from top 5 keywords; maxItems 10; sort Latest; includeUserInfo true
   - Connect: **Format Google Output → X Scrapper**

9) **Create platform normalization Code nodes**
   - **“Tiktok Output”**: output `{platform:"tiktok", keyword, summary:[{caption,views,likes}]}` (top 5)
   - **“Instagram Output”**: output `{platform:"instagram", keyword, summary:[{caption,likes}]}` (top 5)
   - **“X Output”**: output `{platform:"x_twitter", keyword, summary:[{text,views,likes}]}` (top 5)
   - Connect:
     - TikTok Scrapper → Tiktok Output
     - Instagram Scrapper → Instagram Output
     - X Scrapper → X Output

10) **Create “Merge” (Merge node)**
   - Set **Number of Inputs = 3**
   - Connect:
     - Tiktok Output → Merge (Input 1)
     - Instagram Output → Merge (Input 2)
     - X Output → Merge (Input 3)

11) **Create “Code in JavaScript” (Code node)**
   - Collect all merged items and output:
     - `merged_data_text = JSON.stringify(items.map(i => i.json), null, 2)`
   - Connect: **Merge → Code in JavaScript**

12) **Create Gemini credentials**
   - In n8n Credentials: create **Google PaLM / Gemini** credential (API key) used by the Gemini node.

13) **Create “Google Gemini Chat Model” (Gemini Chat Model node)**
   - Model: `models/gemini-flash-latest`
   - Select your Google Gemini credential

14) **Create “AI Agent” (LangChain Agent node)**
   - Prompt includes `{{ $json.merged_data_text }}`
   - System message: enforce strict JSON schema with root key `topics` as shown in the workflow
   - Connect:
     - Code in JavaScript → AI Agent (main)
     - Google Gemini Chat Model → AI Agent (ai_languageModel)

15) **Create “Parse Output” (Code node)**
   - Read AI response text from `item.json.output`
   - Strip ``` fences and `JSON.parse`
   - Connect: **AI Agent → Parse Output**

16) **Create “Split Out” (Split Out node)**
   - Field to split: `topics`
   - Connect: **Parse Output → Split Out**

17) **Create Discord webhook credential**
   - In n8n Credentials: add **Discord Webhook** credential with your Discord webhook URL.

18) **Create “Discord” (Discord node)**
   - Authentication: webhook
   - Message content uses the per-topic fields (`topic_name`, `discussion_summary`, `engagement_signals`, `insight`, `content_recommendation`)
   - Connect: **Split Out → Discord**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Driven Social Intelligence Agent” description, setup steps (Apify token, Gemini key, region in Edit Fields, Discord webhook), and requirements | Sticky note in workflow canvas (no external link provided) |
| Block labels: “1. Multi-Platform Data Fetching”, “2. AI Synthesis & Trend Intelligence”, “3. Granular Reporting & Alerting” | Sticky notes used as section headers on the canvas |
| Apify Actors referenced: TikTok Data Extractor, Instagram Free Hashtag Scraper, Tweet Scraper Pay-Per Result | Accessible in Apify Console via the actor IDs shown in each Apify node configuration |

