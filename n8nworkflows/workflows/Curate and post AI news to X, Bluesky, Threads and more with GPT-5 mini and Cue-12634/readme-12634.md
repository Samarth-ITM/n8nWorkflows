Curate and post AI news to X, Bluesky, Threads and more with GPT-5 mini and Cue

https://n8nworkflows.xyz/workflows/curate-and-post-ai-news-to-x--bluesky--threads-and-more-with-gpt-5-mini-and-cue-12634


# Curate and post AI news to X, Bluesky, Threads and more with GPT-5 mini and Cue

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow curates recent AI news from multiple RSS feeds, generates platform-specific social posts using **GPT-5 mini**, then creates **multi-platform drafts in Cue** (X, Bluesky, Threads, Mastodon, Facebook). It logs processed articles to **Google Sheets** to avoid reposting.

**Primary use cases:**
- Daily AI/tech news digestion with consistent posting cadence
- Multi-platform content generation with per-platform voice constraints
- Editorial workflow: drafts created for review in Cue (optionally auto-publish)

### Logical blocks
**1.1 Scheduling & state lookup (Google Sheets)**  
Runs daily at 9am, pulls the list of previously posted URLs.

**1.2 Feed ingestion (RSS)**  
Fetches AI articles from 4 sources in parallel and merges them.

**1.3 De-dupe & selection**  
Filters to last 7 days, standardizes fields, compares against posted URLs, selects a random unposted article.

**1.4 AI writing (GPT-5 mini + structured output)**  
Generates 5 platform-tailored posts with strict constraints and structured JSON parsing.

**1.5 Draft creation (Cue) + logging (Google Sheets)**  
Creates a Cue draft with 5 platform slots, then appends the article metadata to the tracking sheet.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & state lookup (Google Sheets)

**Overview:**  
Triggers once per day and loads the tracking sheet that stores URLs already processed, so the workflow can skip duplicates.

**Nodes involved:**
- Daily 9am Trigger
- Get Recent Posts
- Trigger Feeds
- Collect Posted URLs

#### Node: Daily 9am Trigger
- **Type / role:** Schedule Trigger; entry point
- **Config choices:** Runs daily, `triggerAtHour: 9` (server/workflow timezone)
- **Input/Output:** No input → outputs to **Get Recent Posts**
- **Edge cases/failures:** Timezone mismatch (n8n instance timezone vs expectation); missed runs if instance down

#### Node: Get Recent Posts
- **Type / role:** Google Sheets (Read); fetch posted history
- **Config choices:**
  - Operation: **read**
  - Sheet name & document ID: **not set in JSON** (must be selected in UI)
  - `onError: continueRegularOutput` + `alwaysOutputData: true` to keep workflow running even if sheet read fails
- **Key variables/expressions:** None inside this node; later nodes reference its **params** for sheet/document IDs
- **Connections:**
  - Output → **Trigger Feeds** (to ensure RSS runs once)
  - Output → **Collect Posted URLs**
- **Edge cases/failures:**
  - OAuth not configured/expired → read fails; workflow continues but “posted list” may be empty
  - Wrong sheet columns (missing `article_url`) → downstream de-dupe becomes ineffective

#### Node: Trigger Feeds
- **Type / role:** Code; normalizes to single item so RSS feeds fire exactly once
- **Config choices:** Returns exactly one item `{ ready: true }` regardless of sheet row count
- **Connections:** Output → all four RSS nodes (parallel)
- **Edge cases:** Minimal; only fails if code node errors (unlikely)

#### Node: Collect Posted URLs
- **Type / role:** Aggregate; turns multiple sheet rows into a single array payload
- **Config choices:**
  - Include: **specified fields**
  - Field included: `article_url`
  - Destination field name: `postedURLs`
- **Input/Output:** Input = sheet rows → Output = 1 item `{ postedURLs: [{article_url: ...}, ...] }`
- **Connections:** Output → **Compare Lists** (Merge input 2 / index 1)
- **Edge cases:** If sheet read returned no items, aggregate may output empty list; de-dupe still works but posts may repeat if history unavailable

---

### 2.2 Feed ingestion (RSS)

**Overview:**  
Fetches articles daily from four AI RSS feeds and merges them into one stream.

**Nodes involved:**
- TechCrunch AI
- Ars Technica AI
- The Verge AI
- MIT Tech Review
- Merge All Feeds

#### Node: TechCrunch AI
- **Type / role:** RSS Feed Read; source 1
- **Config choices:** URL `https://techcrunch.com/category/artificial-intelligence/feed/`
- **Connections:** Output → **Merge All Feeds** (input 1 / index 0)
- **Edge cases:** Feed downtime, rate limits, malformed RSS

#### Node: Ars Technica AI
- **Type / role:** RSS Feed Read; source 2
- **Config choices:** URL `https://arstechnica.com/ai/feed/`
- **Connections:** Output → **Merge All Feeds** (input 2 / index 1)
- **Edge cases:** Same as above

#### Node: The Verge AI
- **Type / role:** RSS Feed Read; source 3
- **Config choices:** URL `https://www.theverge.com/rss/ai-artificial-intelligence/index.xml`
- **Connections:** Output → **Merge All Feeds** (input 3 / index 2)
- **Edge cases:** Same as above

#### Node: MIT Tech Review
- **Type / role:** RSS Feed Read; source 4
- **Config choices:** URL `https://www.technologyreview.com/topic/artificial-intelligence/feed`
- **Connections:** Output → **Merge All Feeds** (input 4 / index 3)
- **Edge cases:** Same as above; also potential paywall/summary truncation issues

#### Node: Merge All Feeds
- **Type / role:** Merge (multi-input); combines 4 RSS streams
- **Config choices:** `numberInputs: 4` (explicitly expects four inputs)
- **Input/Output:** 4 feed outputs → single combined stream
- **Connections:** Output → **Filter Recent**
- **Edge cases/failures:** If one feed fails to produce items, merge behavior can reduce output depending on n8n merge semantics; ensure all branches execute (they do via Trigger Feeds)

---

### 2.3 De-dupe & selection

**Overview:**  
Filters recent items, normalizes fields across feeds, aggregates new items into an array, compares against posted URLs from Sheets, and picks a random unposted article.

**Nodes involved:**
- Filter Recent
- Standardize Fields
- Collect New Articles
- Compare Lists
- Select Random Article

#### Node: Filter Recent
- **Type / role:** Code; filters and sorts articles
- **Config choices (logic):**
  - Keeps items within last **7 days** (if date exists)
  - If `pubDate/isoDate` missing → keeps the item (fails open)
  - Sorts newest-first by `isoDate || pubDate`
  - Limits to **30** items
- **Key variables:** `DAYS=7`, `MAX_ARTICLES=30`, `cutoff=now-7days`
- **Connections:** Output → **Standardize Fields**
- **Edge cases/failures:**
  - Invalid date strings can yield `Invalid Date` and sorting unpredictability
  - “Missing pubDate” items always pass filter (could include old content)

#### Node: Standardize Fields
- **Type / role:** Code; creates a consistent schema for downstream steps
- **Config choices (mapping):**
  - `url`: `link || url`
  - `title`: `title`
  - `description`: `contentSnippet || description || summary || ''`
  - `pubDate`: `isoDate || pubDate`
  - `source`: `creator || author || 'Unknown'`
- **Connections:** Output → **Collect New Articles**
- **Edge cases:** Some feeds use different keys (e.g., nested author); may produce `Unknown` or empty description

#### Node: Collect New Articles
- **Type / role:** Aggregate; packages all standardized RSS items into one array
- **Config choices:** Aggregate all item data into `newArticles`
- **Output shape:** `{ newArticles: [ {url,title,description,pubDate,source}, ... ] }`
- **Connections:** Output → **Compare Lists** (Merge input 1 / index 0)
- **Edge cases:** Large arrays increase token usage later if passed downstream (this workflow selects one article before AI step, so OK)

#### Node: Compare Lists
- **Type / role:** Merge; aligns `newArticles` and `postedURLs` into one execution context
- **Config choices:** Default merge behavior (no explicit mode set in JSON)
- **Connections:** Output → **Select Random Article**
- **Edge cases:** If one side outputs no item, downstream code may see missing arrays; current selection code defaults to `[]`

#### Node: Select Random Article
- **Type / role:** Code; de-duplicates and selects one candidate
- **Config choices (logic):**
  - Reads `newArticles` from `$input.first().json.newArticles`
  - Reads `postedURLs` from `$input.last().json.postedURLs`
  - Builds `postedURLs = postedData.map(item => item.article_url)`
  - Filters `available = newArticles.filter(article => !postedURLs.includes(article.url))`
  - If none available → returns `[]` (workflow ends naturally)
  - Else returns 1 random article item
- **Connections:** Output → **Write Social Posts**
- **Edge cases/failures:**
  - If sheet column is not exactly `article_url`, `postedURLs` becomes `[undefined,...]` → duplicates can slip through
  - URL normalization: same article with tracking params/redirect differences may not match (consider canonicalization if needed)

---

### 2.4 AI writing (GPT-5 mini + structured output)

**Overview:**  
Uses an n8n LangChain Agent configured with GPT-5 mini and a structured output parser to generate platform-tailored text for five social networks, without URLs.

**Nodes involved:**
- Write Social Posts
- GPT-5 mini
- Post Format

#### Node: Write Social Posts
- **Type / role:** LangChain Agent; prompt orchestration and response generation
- **Config choices:**
  - Prompt includes article `TITLE` and `SUMMARY` and requests 5 platform posts
  - Hard constraints:
    - Must start with a compelling hook
    - Include key insight/news
    - Stay within platform character limits
    - **Do NOT include any URLs**
  - System message defines tone and max lengths:
    - X 280, Bluesky 300, Threads 500, Mastodon 500, Facebook 600
  - `promptType: define`
  - `hasOutputParser: true` (expects structured output)
- **Connections:**
  - Receives AI model via **GPT-5 mini** (ai_languageModel connection)
  - Receives parser via **Post Format** (ai_outputParser connection)
  - Main output → **Prepare for Cue**
- **Edge cases/failures:**
  - Model may exceed character limits; no explicit post-generation validation step exists
  - If article description is empty, outputs may be generic
  - If model returns invalid JSON, parser will fail and node execution errors

#### Node: GPT-5 mini
- **Type / role:** OpenAI Chat Model; LLM backend
- **Config choices:** Model = `gpt-5-mini`
- **Credentials:** OpenAI API credential required
- **Connections:** Provides language model to **Write Social Posts**
- **Edge cases:** Auth errors, quota exhaustion, latency/timeouts, model availability changes

#### Node: Post Format
- **Type / role:** Structured Output Parser; enforces JSON schema
- **Config choices:** Manual JSON schema requiring:
  - `x`, `bluesky`, `threads`, `mastodon`, `facebook` (all strings, all required)
- **Connections:** Provides parser to **Write Social Posts**
- **Edge cases:** If the agent output deviates from schema, parsing fails

---

### 2.5 Draft creation (Cue) + logging (Google Sheets)

**Overview:**  
Packages the selected article + generated content, creates a multi-platform draft in Cue, then logs the URL/title/source/timestamp to Google Sheets.

**Nodes involved:**
- Prepare for Cue
- Create Draft in Cue
- Prepare Sheet Data
- Log to Sheet

#### Node: Prepare for Cue
- **Type / role:** Code; combines article + structured AI output
- **Config choices:**
  - `article = $('Select Random Article').item.json`
  - `content = $json.output` (expects the agent’s parsed payload in `output`)
  - Returns `{ article, content }`
- **Connections:** Output → **Create Draft in Cue**
- **Edge cases:**
  - If agent output key differs (not `output`), `content` will be undefined and Cue posting fails
  - Uses cross-node reference to “Select Random Article”; renaming nodes breaks this expression

#### Node: Create Draft in Cue
- **Type / role:** Cue node (resource: post); creates a post draft across platforms
- **Config choices:**
  - Operation: **post.create**
  - `profileId`: must be selected
  - `platforms.platform[]`: five entries, each with:
    - `content` mapped from `$json.content.<platform>`
    - `socialAccountId` selected per platform slot
  - Sticky-note guidance indicates slot mapping:
    - Slot 1 X/Twitter, Slot 2 Bluesky, Slot 3 Threads, Slot 4 Mastodon, Slot 5 Facebook
- **Credentials:** Cue API credential required
- **Connections:** Output → **Prepare Sheet Data**
- **Edge cases/failures:**
  - Missing/incorrect socialAccountId selections
  - Cue API auth/permission issues
  - Content too long for the destination platform (Cue may reject or truncate)
  - If you enable “Publish Immediately” (per note), accidental auto-post risk

#### Node: Prepare Sheet Data
- **Type / role:** Code; shapes logging row
- **Config choices:**
  - Pulls `article` from `$('Prepare for Cue').item.json`
  - Creates:
    - `article_url`, `title`, `source` (fallback `'AI News Digest'`), `processed_at` (ISO timestamp)
- **Connections:** Output → **Log to Sheet**
- **Edge cases:** If `article.url` missing, you’ll log blank URLs and de-dupe breaks

#### Node: Log to Sheet
- **Type / role:** Google Sheets (Append); logging/tracking
- **Config choices:**
  - Operation: **append**
  - Column mapping (define below): `title`, `source`, `article_url`, `processed_at`
  - Sheet/document IDs are dynamically taken from **Get Recent Posts** node params:
    - `sheetName`: `$('Get Recent Posts').params.sheetName.value`
    - `documentId`: `$('Get Recent Posts').params.documentId.value`
- **Credentials:** Google Sheets OAuth2 credential required
- **Edge cases/failures:**
  - If Get Recent Posts didn’t have params set (or node renamed), expressions break
  - Sheet missing the expected columns; append may fail or misplace data
  - OAuth token expiration

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 9am Trigger | Schedule Trigger | Daily workflow entry point | — | Get Recent Posts | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| Get Recent Posts | Google Sheets | Read tracking sheet of previously posted URLs | Daily 9am Trigger | Trigger Feeds; Collect Posted URLs | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Trigger Feeds | Code | Emit single item to trigger RSS reads once | Get Recent Posts | TechCrunch AI; Ars Technica AI; The Verge AI; MIT Tech Review | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| TechCrunch AI | RSS Feed Read | Fetch AI articles from TechCrunch | Trigger Feeds | Merge All Feeds | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| Ars Technica AI | RSS Feed Read | Fetch AI articles from Ars Technica | Trigger Feeds | Merge All Feeds | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| The Verge AI | RSS Feed Read | Fetch AI articles from The Verge | Trigger Feeds | Merge All Feeds | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| MIT Tech Review | RSS Feed Read | Fetch AI articles from MIT Tech Review | Trigger Feeds | Merge All Feeds | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| Merge All Feeds | Merge | Combine 4 RSS streams | TechCrunch AI; Ars Technica AI; The Verge AI; MIT Tech Review | Filter Recent | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| Filter Recent | Code | Keep last 7 days; sort; limit 30 | Merge All Feeds | Standardize Fields | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Standardize Fields | Code | Normalize RSS item schema | Filter Recent | Collect New Articles | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Collect New Articles | Aggregate | Aggregate all new articles into `newArticles` array | Standardize Fields | Compare Lists | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Collect Posted URLs | Aggregate | Aggregate sheet rows into `postedURLs` array | Get Recent Posts | Compare Lists | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Compare Lists | Merge | Bring `newArticles` and `postedURLs` together | Collect New Articles; Collect Posted URLs | Select Random Article | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Select Random Article | Code | Remove posted; pick one random new article | Compare Lists | Write Social Posts | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Write Social Posts | LangChain Agent | Generate 5 platform posts (no URLs) | Select Random Article | Prepare for Cue | ## 3. Write Posts<br><br>GPT-5 mini generates tailored content for each platform.<br><br>• Configure OpenAI credentials |
| GPT-5 mini | OpenAI Chat Model | LLM used by agent | — (AI connection) | Write Social Posts (ai_languageModel) | ## 3. Write Posts<br><br>GPT-5 mini generates tailored content for each platform.<br><br>• Configure OpenAI credentials |
| Post Format | Structured Output Parser | Enforce JSON schema for posts | — (AI connection) | Write Social Posts (ai_outputParser) | ## 3. Write Posts<br><br>GPT-5 mini generates tailored content for each platform.<br><br>• Configure OpenAI credentials |
| Prepare for Cue | Code | Package `{article, content}` for Cue | Write Social Posts | Create Draft in Cue | ## 4. Publish to Cue<br><br>Creates drafts in Cue for review. Logs to spreadsheet for tracking.<br><br>• Configure Cue credentials<br>• Select your social accounts<br><br>**Platforms:**<br>• Slot 1 → X/Twitter<br>• Slot 2 → Bluesky<br>• Slot 3 → Threads<br>• Slot 4 → Mastodon<br>• Slot 5 → Facebook |
| Create Draft in Cue | Cue | Create multi-platform draft post | Prepare for Cue | Prepare Sheet Data | ## 4. Publish to Cue<br><br>Creates drafts in Cue for review. Logs to spreadsheet for tracking.<br><br>• Configure Cue credentials<br>• Select your social accounts<br><br>**Platforms:**<br>• Slot 1 → X/Twitter<br>• Slot 2 → Bluesky<br>• Slot 3 → Threads<br>• Slot 4 → Mastodon<br>• Slot 5 → Facebook<br><br>**Tip:** To auto-publish instead of drafts, enable "Publish Immediately" in the Cue node settings. |
| Prepare Sheet Data | Code | Build logging row fields | Create Draft in Cue | Log to Sheet | ## 4. Publish to Cue<br><br>Creates drafts in Cue for review. Logs to spreadsheet for tracking.<br><br>• Configure Cue credentials<br>• Select your social accounts<br><br>**Platforms:**<br>• Slot 1 → X/Twitter<br>• Slot 2 → Bluesky<br>• Slot 3 → Threads<br>• Slot 4 → Mastodon<br>• Slot 5 → Facebook |
| Log to Sheet | Google Sheets | Append processed article row | Prepare Sheet Data | — | ## 4. Publish to Cue<br><br>Creates drafts in Cue for review. Logs to spreadsheet for tracking.<br><br>• Configure Cue credentials<br>• Select your social accounts<br><br>**Platforms:**<br>• Slot 1 → X/Twitter<br>• Slot 2 → Bluesky<br>• Slot 3 → Threads<br>• Slot 4 → Mastodon<br>• Slot 5 → Facebook |
| Intro | Sticky Note | Documentation / usage guidance | — | — | ## Curate AI news and generate platform-tailored posts for X, Bluesky, Threads, Mastodon & Facebook via Cue.<br><br>### How it works<br>• Fetches from 4 AI/tech RSS feeds daily<br>• Filters to articles from the last 7 days<br>• Skips already-posted articles using Google Sheets tracking<br>• Picks a random new article<br>• GPT-5 mini generates platform-tailored posts<br>• Creates draft in Cue for review<br><br>### How to use<br>• Configure your Google Sheets for tracking<br>• Add your OpenAI API credentials<br>• Set up the Cue node with your social accounts<br>• Activate the workflow - it runs daily at 9am<br><br>### Requirements<br>• Cue account (oncue.so)<br>• OpenAI API key<br>• Google Sheets<br><br>### Need Help?<br>Check out [docs.oncue.so](https://docs.oncue.so) for setup guides!<br><br>Happy Posting! |
| Section 1 | Sticky Note | Section header | — | — | ## 1. Fetch News<br><br>4 AI/tech RSS feeds checked daily.<br><br>• Replace feeds with your niche sources<br>• Keep 3-6 feeds for best results |
| Section 2 | Sticky Note | Section header | — | — | ## 2. Find New Stories<br><br>Filters to last 7 days, skips already-posted articles.<br><br>• Configure Google Sheets credentials<br>• Select your tracking spreadsheet<br>• Import tracking-sheet.csv to get started |
| Section 3 | Sticky Note | Section header | — | — | ## 3. Write Posts<br><br>GPT-5 mini generates tailored content for each platform.<br><br>• Configure OpenAI credentials |
| Section 4 | Sticky Note | Section header | — | — | ## 4. Publish to Cue<br><br>Creates drafts in Cue for review. Logs to spreadsheet for tracking.<br><br>• Configure Cue credentials<br>• Select your social accounts<br><br>**Platforms:**<br>• Slot 1 → X/Twitter<br>• Slot 2 → Bluesky<br>• Slot 3 → Threads<br>• Slot 4 → Mastodon<br>• Slot 5 → Facebook |
| Tip | Sticky Note | Operational tip | — | — | **Tip:** To auto-publish instead of drafts, enable "Publish Immediately" in the Cue node settings. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named:  
   *Curate & post AI news to X, Bluesky, Threads & more via GPT-5 mini & Cue*.

2. **Add Trigger**
   1) Add **Schedule Trigger** node named **Daily 9am Trigger**.  
   2) Set rule to run **every day at 09:00** (confirm timezone in n8n settings).

3. **Add Google Sheets “read” state lookup**
   1) Add **Google Sheets** node named **Get Recent Posts**.  
   2) Credentials: configure **Google Sheets OAuth2**.  
   3) Operation: **Read**.  
   4) Select your **Document** and **Sheet** (tracking sheet).  
   5) Enable:
      - **Continue On Fail** (equivalent to `onError: continueRegularOutput`)
      - **Always Output Data**
   6) Connect: **Daily 9am Trigger → Get Recent Posts**.

4. **Add “Trigger Feeds” normalizer**
   1) Add **Code** node named **Trigger Feeds** with JS returning one item:
      - Output: `[{ json: { ready: true } }]`
   2) Connect: **Get Recent Posts → Trigger Feeds**.

5. **Add 4 RSS feed readers**
   1) Add **RSS Feed Read** nodes with these names and URLs:
      - **TechCrunch AI** → `https://techcrunch.com/category/artificial-intelligence/feed/`
      - **Ars Technica AI** → `https://arstechnica.com/ai/feed/`
      - **The Verge AI** → `https://www.theverge.com/rss/ai-artificial-intelligence/index.xml`
      - **MIT Tech Review** → `https://www.technologyreview.com/topic/artificial-intelligence/feed`
   2) Connect **Trigger Feeds → each RSS node** (4 parallel connections).

6. **Merge RSS outputs**
   1) Add **Merge** node named **Merge All Feeds**.  
   2) Set **Number of Inputs = 4**.  
   3) Connect each RSS node output into Merge inputs 1–4.

7. **Filter to recent + limit**
   1) Add **Code** node **Filter Recent** with logic:
      - keep last 7 days
      - sort newest-first
      - limit 30
   2) Connect: **Merge All Feeds → Filter Recent**.

8. **Standardize RSS fields**
   1) Add **Code** node **Standardize Fields** mapping to:
      - `url`, `title`, `description`, `pubDate`, `source`
   2) Connect: **Filter Recent → Standardize Fields**.

9. **Aggregate new articles**
   1) Add **Aggregate** node **Collect New Articles**.  
   2) Set aggregate mode to **Aggregate All Item Data** into field **newArticles**.  
   3) Connect: **Standardize Fields → Collect New Articles**.

10. **Aggregate posted URLs from Sheets**
    1) Add **Aggregate** node **Collect Posted URLs**.  
    2) Include **only** field `article_url`.  
    3) Destination field name: **postedURLs**.  
    4) Connect: **Get Recent Posts → Collect Posted URLs**.

11. **Merge lists for comparison**
    1) Add **Merge** node **Compare Lists** (default settings).  
    2) Connect:
       - **Collect New Articles → Compare Lists** (input 1)
       - **Collect Posted URLs → Compare Lists** (input 2)

12. **Select random unposted article**
    1) Add **Code** node **Select Random Article** implementing:
       - read `newArticles` and `postedURLs`
       - filter out posted URLs
       - return `[]` if none
       - else return one random article item
    2) Connect: **Compare Lists → Select Random Article**.

13. **Add AI generation (LangChain Agent)**
    1) Add **OpenAI Chat Model** node named **GPT-5 mini**.
       - Credentials: **OpenAI API**
       - Model: `gpt-5-mini`
    2) Add **Structured Output Parser** node named **Post Format**.
       - Schema requires: `x`, `bluesky`, `threads`, `mastodon`, `facebook` (strings)
    3) Add **AI Agent** node named **Write Social Posts**.
       - Provide prompt containing article `TITLE` and `SUMMARY`
       - Include constraints: hook-first, no URLs, per-platform limits
       - System message: platform style guidelines
    4) Connect AI wiring:
       - **GPT-5 mini → Write Social Posts** via *ai_languageModel*
       - **Post Format → Write Social Posts** via *ai_outputParser*
    5) Connect main flow: **Select Random Article → Write Social Posts**.

14. **Prepare payload for Cue**
    1) Add **Code** node **Prepare for Cue**:
       - `article = $('Select Random Article').item.json`
       - `content = $json.output`
       - return `{ article, content }`
    2) Connect: **Write Social Posts → Prepare for Cue**.

15. **Create draft in Cue**
    1) Add **Cue** node named **Create Draft in Cue** (Cue n8n integration).
    2) Credentials: configure **Cue API**.
    3) Operation: **Post → Create**.
    4) Select **profileId**.
    5) Add 5 platform entries with `content` mapped:
       - Slot 1: `{{$json.content.x}}`
       - Slot 2: `{{$json.content.bluesky}}`
       - Slot 3: `{{$json.content.threads}}`
       - Slot 4: `{{$json.content.mastodon}}`
       - Slot 5: `{{$json.content.facebook}}`
       And choose the appropriate **socialAccountId** for each.
    6) (Optional) Enable **Publish Immediately** to auto-post instead of drafts.
    7) Connect: **Prepare for Cue → Create Draft in Cue**.

16. **Prepare logging row**
    1) Add **Code** node **Prepare Sheet Data** producing:
       - `article_url`, `title`, `source`, `processed_at` (ISO timestamp)
    2) Connect: **Create Draft in Cue → Prepare Sheet Data**.

17. **Append row to Google Sheets**
    1) Add **Google Sheets** node **Log to Sheet** (Append).
    2) Credentials: same **Google Sheets OAuth2**.
    3) Operation: **Append**.
    4) Map columns: `title`, `source`, `article_url`, `processed_at`.
    5) Use the same document/sheet as the reader (either select explicitly, or replicate the workflow’s approach by referencing the read node’s selected params).
    6) Connect: **Prepare Sheet Data → Log to Sheet**.

18. **Activate workflow** and run a manual execution once to validate:
    - RSS fetch returns items
    - Sheet read returns `article_url` values
    - AI agent produces valid structured output
    - Cue draft is created with 5 platforms populated
    - Sheet append succeeds

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Cue setup guides available here: **docs.oncue.so** | https://docs.oncue.so |
| Cue account required | https://oncue.so |
| Tip: To auto-publish instead of drafts, enable **“Publish Immediately”** in the Cue node settings. | Applies to “Create Draft in Cue” node |
| Tracking sheet suggestion: “Import tracking-sheet.csv to get started” | Applies to Google Sheets tracking setup |

