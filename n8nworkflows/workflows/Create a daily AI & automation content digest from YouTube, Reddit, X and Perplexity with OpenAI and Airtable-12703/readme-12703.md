Create a daily AI & automation content digest from YouTube, Reddit, X and Perplexity with OpenAI and Airtable

https://n8nworkflows.xyz/workflows/create-a-daily-ai---automation-content-digest-from-youtube--reddit--x-and-perplexity-with-openai-and-airtable-12703


# Create a daily AI & automation content digest from YouTube, Reddit, X and Perplexity with OpenAI and Airtable

## 1. Workflow Overview

**Workflow name:** Content Report Automation SKOOL-1  
**Goal:** Every day at **8:00 AM (America/Chicago)**, collect trending content about AI + automation from **YouTube**, **Reddit**, **X (via Apify)**, and **Perplexity**, then:
- Summarize/analyze each source with LLMs (OpenAI + OpenRouter)
- Store granular items in **Airtable**
- Synthesize everything into a **daily HTML email digest** (Gmail)
- Generate and store **5 strategic content ideas** (Airtable)

### 1.1 Scheduled Trigger (Entry Point)
Runs daily at 8 AM and launches 4 parallel branches (YouTube, Reddit, X, Perplexity).

### 1.2 YouTube Discovery → Enrichment → Transcript Summaries → Airtable + Merge
Searches YouTube for “n8n” videos from the last 24h window, filters out Shorts (duration threshold), fetches transcripts via Apify, summarizes into “Quick Summary” and “Deep Dive Summary”, stores each in Airtable, and prepares an aggregated `youtubeData` array for the report.

### 1.3 Reddit Collection → One-line Summaries → Airtable + Merge
Pulls top 5 rising posts from r/n8n, summarizes each post into `{title, link, summary}`, stores in Airtable, and prepares an aggregated `redditData` array for the report.

### 1.4 X Collection (Apify) → LLM Ranking & Topic Mining → Airtable + Merge
Scrapes top tweets for a set of keywords over the last day, uses an LLM to select top tweets and infer trending topics, stores top tweets in Airtable, and prepares an `xData` object for the report.

### 1.5 Perplexity News → JSON Normalization → Airtable + Merge
Queries Perplexity for top 3 AI/automation news stories, normalizes them into a strict JSON array, stores stories in Airtable, and prepares `perplexityData` for the report.

### 1.6 Merge → Final Report + Email + Content Ideation
Merges all four data products into one object (`allData`), generates:
- A styled **HTML email digest** (Report Generator) and sends via Gmail
- A structured **content strategy output** (Content Brainstorm) split into individual ideas and saved to Airtable

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger / Orchestration

**Overview:** Starts the workflow daily and fans out into four parallel data-collection branches.  
**Nodes involved:** `Daily at 8 AM`

#### Node: Daily at 8 AM
- **Type / role:** Schedule Trigger — timed entry point.
- **Config:** Runs **daily at 08:00** (timezone set at workflow level to **America/Chicago**).
- **Outputs:** Connects to 4 nodes in parallel:
  - `n8n Trending` (Reddit)
  - `Scrape X` (X/Apify)
  - `Get Videos` (YouTube)
  - `Perplexity AI News` (Perplexity)
- **Failure modes / edge cases:**
  - Timezone mismatch if instance timezone differs from workflow settings.
  - Overlapping executions if prior run is still processing (depends on n8n concurrency/execution settings).

---

### Block 2.2 — YouTube Scraper: Search → Video details → Filter shorts → Transcript scraping → LLM summaries → Airtable → Aggregate for report

**Overview:** Finds recent “n8n” videos, filters out short-form content, pulls transcripts from Apify, summarizes with OpenAI, stores each summary in Airtable, then aggregates all summarized videos into a `youtubeData` array for the final report.  
**Nodes involved:** `Get Videos`, `Get Video Data`, `Filter Out Shorts`, `Get Transcripts`, `Transcript Analysis`, `Structured Output Parser1`, `OpenAI Chat Model1`, `YouTube Airtable`, `Aggregate2`

#### Node: Get Videos
- **Type / role:** YouTube node (Search videos) — discovery.
- **Config choices:**
  - Resource: **video**
  - Query (`q`): `n8n`
  - Region: **US**
  - Order: **relevance**
  - SafeSearch: **moderate**
  - Limit: **10**
  - `publishedAfter`: expression computing an ISO timestamp for “roughly last 24 hours” with a 7:00 AM anchor.
- **Output:** Items include `id.videoId` for each search result.
- **Connections:** → `Get Video Data`
- **Failure modes / edge cases:**
  - OAuth scope/consent issues (YouTube OAuth2 credential).
  - API quota exhaustion.
  - `publishedAfter` expression errors if date math fails.

#### Node: Get Video Data
- **Type / role:** HTTP Request — fetch full video metadata from YouTube Data API v3.
- **Config choices:**
  - GET `https://www.googleapis.com/youtube/v3/videos?`
  - Query params:
    - `key=YOURAPIKEY` (placeholder; must be replaced)
    - `id={{ $json.id.videoId }}`
    - `part=contentDetails,snippet,statistics`
- **Output:** `items[0]` with duration, title, thumbnails, channel, view count, etc.
- **Connections:** → `Filter Out Shorts`
- **Failure modes / edge cases:**
  - Invalid/absent API key.
  - If `items` is empty (video removed/region blocked), downstream expressions referencing `items[0]` will fail.

#### Node: Filter Out Shorts
- **Type / role:** IF — remove short videos.
- **Config choices:**
  - Boolean condition computed via an inline JS IIFE:
    - Parses `items[0].contentDetails.duration` (ISO 8601)
    - Converts to seconds
    - Keeps videos **> 210 seconds** (3m30s)
- **Outputs:** True path continues; false path is unused (effectively discards).
- **Connections:** (true) → `Get Transcripts`
- **Failure modes / edge cases:**
  - `duration` missing or malformed breaks regex parsing.
  - `items[0]` missing → expression failure.

#### Node: Get Transcripts
- **Type / role:** HTTP Request — Apify actor call to extract YouTube transcripts.
- **Config choices:**
  - POST `https://api.apify.com/v2/acts/karamelo~youtube-transcripts/run-sync-get-dataset-items`
  - JSON body includes:
    - `urls`: `https://www.youtube.com/watch?v={{ $json.items[0].id }}`
    - `maxRetries: 8`
    - Uses Apify Proxy with group `BUYPROXIES94952`
  - Headers include `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder)
- **Output:** Transcript dataset items; workflow later reads `captions` and `videoId`.
- **Connections:** → `Transcript Analysis`
- **Failure modes / edge cases:**
  - Missing/invalid Apify token.
  - Actor can return no transcript (captions disabled, video blocked).
  - Proxy group may not exist on your Apify account.

#### Node: Transcript Analysis
- **Type / role:** LangChain LLM Chain — summarization.
- **Model:** Provided through `OpenAI Chat Model1`.
- **Prompt behavior:**
  - Inputs transcript: `{{ $json.captions }}`
  - Auto-corrects misheard “n8n” variants (NA10/NADN/etc).
  - Produces **English output** (translates if needed).
  - Demands strict JSON with keys: `Title`, `Link`, `Quick Summary`, `Deep Dive Summary`
  - Must include `https://www.youtube.com/watch?v={{ $json.videoId }}` in Link.
- **Output parsing:** Uses `Structured Output Parser1`.
- **Connections:** → `Aggregate2` and → `YouTube Airtable` (both from main output)
- **Failure modes / edge cases:**
  - Transcript too long → LLM truncation or refusal.
  - If actor output doesn’t include `captions` or `videoId`, prompt constraints break.
  - Parser failures if LLM adds extra text or invalid JSON.

#### Node: OpenAI Chat Model1
- **Type / role:** LLM provider node (OpenAI Chat).
- **Config:** `gpt-4.1-mini`.
- **Connections:** AI languageModel input for `Transcript Analysis`.
- **Failure modes:** Credential invalid, model not available, rate limits.

#### Node: Structured Output Parser1
- **Type / role:** Structured JSON parser for `Transcript Analysis`.
- **Schema expectation:** object with `Title`, `Link`, `Quick Summary`, `Deep Dive Summary`.
- **Connections:** AI outputParser input for `Transcript Analysis`.
- **Failure modes:** Strict schema mismatch → chain errors.

#### Node: YouTube Airtable
- **Type / role:** Airtable Create — stores each summarized YouTube video.
- **Base/Table:** “AI Content Hub” → “YouTube Videos”
- **Field mapping (key points):**
  - URL: built using `$('Get Transcripts').item.json.videoId`
  - Title/Views/Channel/Thumbnail: taken from `$('Filter Out Shorts').item.json.items[0]...`
  - Summaries: from `$json.output["Quick Summary"]`, `$json.output["Deep Dive Summary"]`
- **Connections:** none (terminal for this branch’s storage).
- **Failure modes / edge cases:**
  - Airtable auth/token invalid.
  - Base/table/field names mismatch (if template not copied).
  - Cross-node `.item` references can break if item pairing differs (common when parallelism or batching changes). This is a key fragility: it assumes the current item context aligns between `Transcript Analysis`, `Get Transcripts`, and `Filter Out Shorts`.

#### Node: Aggregate2
- **Type / role:** Aggregate — collects all Transcript Analysis items into `youtubeData`.
- **Config:** `aggregateAllItemData`, destination field name: `youtubeData`
- **Connections:** → `Merge` (input 0)
- **Failure modes:** If no videos pass filter, aggregate may output empty array (report generator must handle).

**Sticky note context:**  
“## Youtube Scraper — grabs top 10 videos in your niche; analyzes transcripts; filters out shorts”

---

### Block 2.3 — Reddit Scraper: Rising posts → LLM summary → Airtable + Aggregate for report

**Overview:** Fetches 5 rising posts from r/n8n, summarizes each into a compact structure, stores them in Airtable, and aggregates them into `redditData` for the final report.  
**Nodes involved:** `n8n Trending`, `Reddit Analysis`, `OpenAI Chat Model`, `Structured Output Parser2`, `Reddit Airtable`, `Aggregate`, `Edit Fields2`

#### Node: n8n Trending
- **Type / role:** Reddit node — fetch posts.
- **Config:**
  - Operation: `getAll`
  - Subreddit: `n8n`
  - Category: `rising`
  - Limit: 5
- **Connections:** → `Reddit Analysis`
- **Failure modes:** OAuth issues, subreddit rate limits, empty results.

#### Node: Reddit Analysis
- **Type / role:** LangChain LLM Chain — extract title/URL + one-sentence summary.
- **Model:** Provided via `OpenAI Chat Model`.
- **Input text composed from item fields:**
  - `selftext`, `title`, `url`
- **Prompt:** “Identify the Reddit post title and URL and include a one sentence summary…”
- **Output parsing:** `Structured Output Parser2` (expects an array of objects).
- **Connections:** → `Aggregate` and → `Reddit Airtable`
- **Failure modes:**
  - `selftext` can be empty; should still summarize based on title.
  - Parser mismatch if model outputs markdown or non-JSON.

#### Node: OpenAI Chat Model
- **Type / role:** OpenAI Chat model provider.
- **Config:** `gpt-4.1-mini`
- **Connections:** AI languageModel for `Reddit Analysis`.

#### Node: Structured Output Parser2
- **Type / role:** Structured JSON parser.
- **Expected schema:** JSON array like `[{ title, link, summary }]`
- **Connections:** AI outputParser for `Reddit Analysis`.

#### Node: Reddit Airtable
- **Type / role:** Airtable Create — stores Reddit posts.
- **Base/Table:** “AI Content Hub” → “Reddit Posts”
- **Field mapping:**
  - URL/Title/Summary from `$json.output[0]...` (note: it uses index 0, assuming exactly one object per item; but parser schema example is an array)
  - Upvotes/Engagement from `$('n8n Trending').item.json.ups`
- **Connections:** none.
- **Failure modes / edge cases:**
  - If the parser returns an array with >1 object, only `[0]` is used.
  - `ups` may be missing depending on Reddit API shape.
  - Cross-node `.item` coupling can break if item linkage changes.

#### Node: Aggregate
- **Type / role:** Aggregate — collects all Reddit analysis items.
- **Config:** `aggregateAllItemData`
- **Connections:** → `Edit Fields2`
- **Failure modes:** Empty result set leads to empty aggregate output.

#### Node: Edit Fields2
- **Type / role:** Set — normalize field name to match report input contract.
- **Config:** sets `redditData = {{ $json.data }}` (array)
- **Connections:** → `Merge` (input 1)
- **Failure modes:** If `Aggregate` outputs a different structure than `{data: ...}`, expression fails.

**Sticky note context:**  
“## Reddit Scraper — grabs top 5 rising posts in r/n8n”

---

### Block 2.4 — X Scraper (Apify): scrape tweets → aggregate → LLM selects top tweets + trending topics → save top tweets → prepare xData for report

**Overview:** Uses Apify to scrape tweets for keyword set, uses an LLM to rank top 5 and infer trending topics, saves top tweets to Airtable, and prepares `xData` for report generation.  
**Nodes involved:** `Scrape X`, `Aggregate1`, `Twitter Analysis`, `OpenAI Chat Model2`, `Structured Output Parser3`, `Split Out`, `X Airtable`, `Edit Fields`, `Aggregate` (indirectly not), `Merge` (later)

#### Node: Scrape X
- **Type / role:** HTTP Request — Apify tweet scraper actor execution.
- **Config:**
  - POST `https://api.apify.com/v2/acts/apidojo~tweet-scraper/run-sync-get-dataset-items`
  - JSON body includes:
    - Date window: `start={{ $now.minus({days:1}).toFormat('yyyy-MM-dd') }}`, `end={{ $now.toFormat('yyyy-MM-dd') }}`
    - `maxItems: 50`
    - Minimum engagement thresholds:
      - favorites ≥ 100, replies ≥ 10, retweets ≥ 10
    - `searchTerms`: `n8n`, `ai automation`, `ai agent`, `claude`
    - sort: `Top`, language: `en`
  - Authorization bearer token placeholder `YOUR_TOKEN_HERE`
- **Connections:** → `Aggregate1`
- **Failure modes / edge cases:**
  - Apify token invalid.
  - Actor may return fewer than 50 items (filters too strict).
  - X scraping instability (rate limiting, content access).

#### Node: Aggregate1
- **Type / role:** Aggregate — groups all scraped tweet items into one payload.
- **Config:** `aggregateAllItemData`
- **Connections:** → `Twitter Analysis`

#### Node: Twitter Analysis
- **Type / role:** LangChain LLM Chain — ranking + topic mining.
- **Model:** `OpenAI Chat Model2` (gpt-4.1-mini).
- **Input text:** `={{ $json.data }}` (expects aggregated tweet array in `data`)
- **Prompt:** Detailed instructions to:
  - rank top 5 by `likeCount`, tie-break by retweet/view
  - extract top 3–5 trending topics
  - output markdown sections
- **Output parsing:** `Structured Output Parser3` expects strict JSON with:
  - `top_tweets[]` containing {rank, tweet_id, text, like_count, url}
  - `trending_topics[]` containing {rank, title, description}
  - `metadata`
- **Connections:**
  - → `Split Out` (to store individual tweets)
  - → `Edit Fields` (to pass xData to merge)
- **Failure modes / edge cases:**
  - Source tweets may use different field names (`fullText` vs `text`, `twitterUrl` vs `url`)—prompt tries to account for both, but parsing requires model to normalize correctly.
  - Parser failure if model outputs markdown instead of JSON (despite parser).
  - If `Aggregate1` structure differs (no `.data`), analysis input becomes incorrect.

#### Node: OpenAI Chat Model2
- **Type / role:** OpenAI provider for Twitter Analysis.
- **Config:** `gpt-4.1-mini`
- **Connections:** AI languageModel for `Twitter Analysis`.

#### Node: Structured Output Parser3
- **Type / role:** Structured JSON parser for Twitter Analysis output.
- **Connections:** AI outputParser for `Twitter Analysis`.

#### Node: Split Out
- **Type / role:** Split Out — iterates through top tweets.
- **Config:** split `output.top_tweets` into separate items.
- **Connections:** → `X Airtable`
- **Failure modes:** If `output.top_tweets` missing or not an array, split fails.

#### Node: X Airtable
- **Type / role:** Airtable Create — stores top tweets.
- **Base/Table:** “AI Content Hub” → “Tweets”
- **Field mapping:** URL, Like Count, Tweet Text from split items (`$json.url`, `$json.like_count`, `$json.text`)
- **Connections:** none.
- **Failure modes:** Airtable field type mismatch (Like Count numeric), missing keys.

#### Node: Edit Fields
- **Type / role:** Set — normalize LLM output for report input contract.
- **Config:** sets `xData = {{ $json.output }}` (object)
- **Connections:** → `Merge` (input 2)

**Sticky note context:**  
“## Twitter Scraper — grabs top 50 tweets amongst a number of keywords; identifies top 5 tweets & trending topics”

---

### Block 2.5 — Perplexity Web Search: ask for top AI news → normalize to JSON array → store in Airtable → prepare perplexityData for report

**Overview:** Pulls “top 3 trending AI news stories” via Perplexity, reformats into a strict JSON array with headline/content, stores items into Airtable, and also provides raw `perplexityData` text for downstream report generation.  
**Nodes involved:** `Perplexity AI News`, `Format Message`, `OpenAI Chat Model3`, `Structured Output Parser4`, `Split Out1`, `Perplexity Airtable`, `Edit Fields1`

#### Node: Perplexity AI News
- **Type / role:** Perplexity node — web-connected answer generation.
- **Config:**
  - Model: `sonar-pro`
  - Prompt asks: “top 3 trending news stories in AI and AI automation today, YYYY-MM-DD; brief summary of each”
  - Simplify enabled (so output is easier to consume in `$json.message`)
- **Connections:**
  - → `Format Message` (for structured extraction)
  - → `Edit Fields1` (for raw text used in report)
- **Failure modes:** API key/auth issues, model availability, output not consistently structured.

#### Node: Format Message
- **Type / role:** LangChain LLM Chain — strict JSON extraction.
- **Model:** `OpenAI Chat Model3` (gpt-4.1-mini).
- **Input text:** `={{ $json.message }}`
- **Prompt constraints:**
  - Extract exactly 3 stories
  - Output ONLY a JSON array
  - Remove citation markers like `[1]`
- **Output parsing:** `Structured Output Parser4` (expects JSON array).
- **Connections:** → `Split Out1`
- **Failure modes:**
  - Perplexity output may not clearly delineate 3 stories.
  - Parser failure if model adds text outside JSON.

#### Node: OpenAI Chat Model3
- **Type / role:** OpenAI provider.
- **Config:** `gpt-4.1-mini`
- **Connections:** AI languageModel for `Format Message`.

#### Node: Structured Output Parser4
- **Type / role:** Structured JSON parser for news array.
- **Connections:** AI outputParser for `Format Message`.

#### Node: Split Out1
- **Type / role:** Split Out — creates one item per news story.
- **Config:** splits `output` and includes all other fields.
- **Connections:** → `Perplexity Airtable`
- **Failure modes:** If `output` isn’t an array, split fails.

#### Node: Perplexity Airtable
- **Type / role:** Airtable Create — stores each news story.
- **Base/Table:** “AI Content Hub” → “Perplexity News”
- **Field mapping:**
  - Headline: `{{ $json.output.headline }}`
  - Summary: `{{ $json.output.content }}`
- **Connections:** none.

#### Node: Edit Fields1
- **Type / role:** Set — normalize for report input contract.
- **Config:** `perplexityData = {{ $json.message }}`
- **Connections:** → `Merge` (input 3)

**Sticky note context:**  
“## Web Search — finds top three AI news stories of the day via perplexity”

---

### Block 2.6 — Merge + Data Synthesis: combine sources → create HTML email → send Gmail

**Overview:** Combines the four source outputs, aggregates to a single `allData` object, generates an HTML email digest with OpenRouter GPT‑4.1, and emails it via Gmail.  
**Nodes involved:** `Merge`, `Aggregate3`, `Report Generator`, `OpenRouter Chat Model`, `Send a message`

#### Node: Merge
- **Type / role:** Merge (multi-input) — combines the four branches.
- **Config:** `numberInputs: 4`
- **Expected inputs:**
  1. From `Aggregate2`: `youtubeData`
  2. From `Edit Fields2`: `redditData`
  3. From `Edit Fields`: `xData`
  4. From `Edit Fields1`: `perplexityData`
- **Output:** One stream that is later aggregated into `allData`.
- **Failure modes / edge cases:**
  - If one branch produces no items, merge behavior can lead to missing keys or no merged output depending on merge mode defaults (n8n Merge node behavior is sensitive to input cardinality).

#### Node: Aggregate3
- **Type / role:** Aggregate — produce one object containing all merged data.
- **Config:** `aggregateAllItemData`, destination `allData`
- **Connections:** → `Report Generator` and → `Content Brainstorm`
- **Failure modes:** If Merge emits multiple items unexpectedly, `allData` may become an array-of-objects rather than a single object.

#### Node: Report Generator
- **Type / role:** LangChain LLM Chain — HTML digest generation.
- **Model:** `OpenRouter Chat Model` (openai/gpt-4.1).
- **Input:** `={{ $json.allData }}`
- **Prompt:** Large HTML spec:
  - Only HTML output (no markdown)
  - Includes header, TL;DR, sections for YouTube/Reddit/X/Perplexity, content ideas section
  - Use only top 3 items for most sections; all X trending topics; remove citations
- **Connections:** → `Send a message`
- **Failure modes:**
  - HTML can exceed email size limits if LLM is verbose (especially deep dives).
  - If upstream structures differ from assumed schema (`youtubeData[].output...`), content may be blank or malformed.
  - No output parser here; invalid HTML may slip through.

#### Node: OpenRouter Chat Model
- **Type / role:** LLM provider via OpenRouter.
- **Config:** Model `openai/gpt-4.1`
- **Connections:** AI languageModel for both `Report Generator` and `Content Brainstorm`.
- **Failure modes:** OpenRouter key invalid, model routing errors, rate limits.

#### Node: Send a message
- **Type / role:** Gmail — send email.
- **Config:**
  - To: `YOUREMAIL` (placeholder)
  - Subject: `Daily Content Report YYYY-MM-DD`
  - Body: `{{ $json.text }}` (expects the Report Generator output field to be `text`)
  - `appendAttribution: false`
- **Connections:** none.
- **Failure modes / edge cases:**
  - Gmail OAuth expired/needs re-auth.
  - If `Report Generator` output is in a different field than `text`, email body becomes empty.
  - Gmail may sanitize some HTML/CSS; inline styles are used to reduce this risk.

**Sticky note context:**  
“## Data Synthesis & Report Generation — daily email report created; airtable data uploaded”

---

### Block 2.7 — Content Strategy: analyze all data → produce structured ideas → split → store in Airtable

**Overview:** Uses the combined `allData` payload to produce a structured content strategy response (patterns, trends, and 5 content ideas), then stores each idea as a record in Airtable.  
**Nodes involved:** `Content Brainstorm`, `Structured Output Parser`, `Ideas`, `Create a record`

#### Node: Content Brainstorm
- **Type / role:** LangChain LLM Chain — strategy/ideation.
- **Model:** OpenRouter GPT‑4.1 via `OpenRouter Chat Model`.
- **Input:** `={{ $json.allData }}`
- **Prompt:** Large “content strategy system” with a viral psychology framework; requests **5 strategic content ideas**.
- **Output parsing:** `Structured Output Parser` (very detailed schema).
- **Connections:** → `Ideas`
- **Failure modes:**
  - Output schema is complex; parser failures are common if the model omits fields.
  - Large prompt + large input may hit context limits; consider truncating `allData` or reducing deep-dive lengths.

#### Node: Structured Output Parser
- **Type / role:** Structured JSON parser for content strategy output.
- **Expected:** Object with `data_synthesis` and `content_ideas[]` each containing deep nested fields.
- **Connections:** AI outputParser for `Content Brainstorm`.

#### Node: Ideas
- **Type / role:** Split Out — one item per content idea.
- **Config:** splits `output.content_ideas`
- **Connections:** → `Create a record`
- **Failure modes:** Missing `output.content_ideas` breaks split.

#### Node: Create a record
- **Type / role:** Airtable Create — store each content idea.
- **Base/Table:** “AI Content Hub” → “Content Ideas”
- **Field mapping highlights:**
  - Idea Title: `{{ $json.title }}`
  - Description: `{{ $json.content_overview }}`
  - Why/Hook/Storyline: formatted multi-line strings assembling nested fields such as:
    - `why_this_idea.strategic_reasoning`
    - `hook_strategy.multi_modal_components.spoken_hook.line_1/2/3`
    - storyline intro/points/outro fields
- **Failure modes:**
  - If any nested field is missing, Airtable mapping may evaluate to empty or error (depending on n8n expression strictness).
  - Airtable long text limits (very long storylines) could truncate.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily at 8 AM | Schedule Trigger | Daily execution entry point | — | n8n Trending; Scrape X; Get Videos; Perplexity AI News | ## 📄 Daily Content Report / Created by Chase AI (https://www.skool.com/chase-ai) … (APIs needed, resources, costs) |
| Get Videos | YouTube | Search recent videos for niche keyword | Daily at 8 AM | Get Video Data | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| Get Video Data | HTTP Request | Fetch full metadata for each video ID | Get Videos | Filter Out Shorts | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| Filter Out Shorts | IF | Remove videos under 3m30s | Get Video Data | Get Transcripts | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| Get Transcripts | HTTP Request | Apify actor call to fetch YouTube transcripts | Filter Out Shorts | Transcript Analysis | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| OpenAI Chat Model1 | OpenAI Chat Model (LangChain) | LLM provider for transcript summarization | — | Transcript Analysis (ai_languageModel) | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| Structured Output Parser1 | Structured Output Parser (LangChain) | Enforce transcript summary JSON schema | — | Transcript Analysis (ai_outputParser) | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| Transcript Analysis | LLM Chain (LangChain) | Summarize transcript into quick + deep dive JSON | Get Transcripts | Aggregate2; YouTube Airtable | ## Youtube Scraper — grabs top 10 videos; analyzes transcripts; filters out shorts |
| YouTube Airtable | Airtable | Store each summarized YouTube video | Transcript Analysis | — | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Aggregate2 | Aggregate | Collect all YouTube summaries into `youtubeData` | Transcript Analysis | Merge | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| n8n Trending | Reddit | Fetch top 5 rising posts from r/n8n | Daily at 8 AM | Reddit Analysis | ## Reddit Scraper — grabs top 5 rising posts in r/n8n |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for Reddit summarization | — | Reddit Analysis (ai_languageModel) | ## Reddit Scraper — grabs top 5 rising posts in r/n8n |
| Structured Output Parser2 | Structured Output Parser (LangChain) | Enforce Reddit summary JSON schema | — | Reddit Analysis (ai_outputParser) | ## Reddit Scraper — grabs top 5 rising posts in r/n8n |
| Reddit Analysis | LLM Chain (LangChain) | Extract title/link + 1 sentence summary | n8n Trending | Aggregate; Reddit Airtable | ## Reddit Scraper — grabs top 5 rising posts in r/n8n |
| Reddit Airtable | Airtable | Store Reddit post summaries | Reddit Analysis | — | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Aggregate | Aggregate | Collect all Reddit items | Reddit Analysis | Edit Fields2 | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Edit Fields2 | Set | Rename aggregated Reddit array to `redditData` | Aggregate | Merge | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Scrape X | HTTP Request | Apify tweet scraping by keywords | Daily at 8 AM | Aggregate1 | ## Twitter Scraper — grabs top 50 tweets; identifies top 5 tweets & trending topics |
| Aggregate1 | Aggregate | Combine tweet items into one `data` array | Scrape X | Twitter Analysis | ## Twitter Scraper — grabs top 50 tweets; identifies top 5 tweets & trending topics |
| OpenAI Chat Model2 | OpenAI Chat Model (LangChain) | LLM provider for X analysis | — | Twitter Analysis (ai_languageModel) | ## Twitter Scraper — grabs top 50 tweets; identifies top 5 tweets & trending topics |
| Structured Output Parser3 | Structured Output Parser (LangChain) | Enforce X analysis JSON schema | — | Twitter Analysis (ai_outputParser) | ## Twitter Scraper — grabs top 50 tweets; identifies top 5 tweets & trending topics |
| Twitter Analysis | LLM Chain (LangChain) | Pick top tweets + trending topics | Aggregate1 | Split Out; Edit Fields | ## Twitter Scraper — grabs top 50 tweets; identifies top 5 tweets & trending topics |
| Split Out | Split Out | Emit top tweets as individual items | Twitter Analysis | X Airtable | ## Twitter Scraper — grabs top 50 tweets; identifies top 5 tweets & trending topics |
| X Airtable | Airtable | Store top tweets | Split Out | — | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Edit Fields | Set | Store `xData = output` for report | Twitter Analysis | Merge | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Perplexity AI News | Perplexity | Fetch top 3 AI/automation news stories | Daily at 8 AM | Format Message; Edit Fields1 | ## Web Search — finds top three AI news stories of the day via perplexity |
| OpenAI Chat Model3 | OpenAI Chat Model (LangChain) | LLM provider for Perplexity normalization | — | Format Message (ai_languageModel) | ## Web Search — finds top three AI news stories of the day via perplexity |
| Structured Output Parser4 | Structured Output Parser (LangChain) | Enforce Perplexity JSON news array | — | Format Message (ai_outputParser) | ## Web Search — finds top three AI news stories of the day via perplexity |
| Format Message | LLM Chain (LangChain) | Extract 3 stories into strict JSON | Perplexity AI News | Split Out1 | ## Web Search — finds top three AI news stories of the day via perplexity |
| Split Out1 | Split Out | Emit each news story as item | Format Message | Perplexity Airtable | ## Web Search — finds top three AI news stories of the day via perplexity |
| Perplexity Airtable | Airtable | Store Perplexity news stories | Split Out1 | — | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Edit Fields1 | Set | Store raw `perplexityData` text | Perplexity AI News | Merge | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Merge | Merge | Combine YouTube/Reddit/X/Perplexity inputs | Aggregate2; Edit Fields2; Edit Fields; Edit Fields1 | Aggregate3 | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Aggregate3 | Aggregate | Create `allData` bundle for downstream LLMs | Merge | Report Generator; Content Brainstorm | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| OpenRouter Chat Model | OpenRouter Chat Model (LangChain) | LLM provider for report + brainstorm | — | Report Generator (ai_languageModel); Content Brainstorm (ai_languageModel) | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Report Generator | LLM Chain (LangChain) | Generate final HTML email digest | Aggregate3 | Send a message | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Send a message | Gmail | Email the HTML digest | Report Generator | — | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce content strategy JSON schema | — | Content Brainstorm (ai_outputParser) | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Content Brainstorm | LLM Chain (LangChain) | Produce patterns + 5 content ideas | Aggregate3 | Ideas | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Ideas | Split Out | Emit one Airtable record per idea | Content Brainstorm | Create a record | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Create a record | Airtable | Store content ideas | Ideas | — | ## Data Synthesis & Report Generation — daily email report created; airtable data uploaded |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## 📄 Daily Content Report … (links/resources/costs) |
| Sticky Note | Sticky Note | Comment block | — | — | ## Youtube Scraper … |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Reddit Scraper … |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Twitter Scraper … |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Web Search … |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Data Synthesis & Report Generation … |
| Sticky Note6 | Sticky Note | Empty canvas note | — | — |  |
| Sticky Note7 | Sticky Note | Empty canvas note | — | — |  |
| Sticky Note8 | Sticky Note | Empty canvas note | — | — |  |
| Sticky Note9 | Sticky Note | Comment block | — | — | @[youtube](m-7egcppFPk) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow + settings**
   1. Create a new workflow in n8n.
   2. Set workflow timezone to **America/Chicago**.
   3. (Optional) Configure an error workflow if desired (this workflow references one).

2) **Add trigger**
   1. Add node: **Schedule Trigger**
   2. Configure: run daily at **08:00**.
   3. Create 4 outgoing connections to: YouTube branch, Reddit branch, X branch, Perplexity branch.

3) **YouTube branch (search → details → filter → transcript → summarize → Airtable → aggregate)**
   1. Add **YouTube** node “Get Videos”:
      - Resource: Video search
      - Query: `n8n`
      - Region: US; order: relevance; safeSearch: moderate
      - Limit: 10
      - PublishedAfter: expression that returns ISO timestamp for last day window (as in workflow).
      - Credential: **YouTube OAuth2** (Google) with YouTube Data API enabled.
   2. Add **HTTP Request** “Get Video Data”:
      - GET `https://www.googleapis.com/youtube/v3/videos`
      - Query params: `key=<YOUR_YT_API_KEY>`, `id={{$json.id.videoId}}`, `part=contentDetails,snippet,statistics`
   3. Add **IF** “Filter Out Shorts”:
      - Condition: convert `items[0].contentDetails.duration` to seconds and keep > 210.
   4. Add **HTTP Request** “Get Transcripts” (Apify):
      - POST `https://api.apify.com/v2/acts/karamelo~youtube-transcripts/run-sync-get-dataset-items`
      - Header: `Authorization: Bearer <APIFY_TOKEN>`
      - JSON body: include `urls: ["https://www.youtube.com/watch?v={{ $json.items[0].id }}"]` and proxy options as needed.
   5. Add **OpenAI Chat Model** “OpenAI Chat Model1”:
      - Model: `gpt-4.1-mini`
      - Credential: OpenAI API key.
   6. Add **Structured Output Parser** “Structured Output Parser1”:
      - Schema example for `{Title, Link, Quick Summary, Deep Dive Summary}`.
   7. Add **LLM Chain** “Transcript Analysis”:
      - Text input: `{{$json.captions}}`
      - Prompt: enforce JSON-only output, translation to English, and link with `{{$json.videoId}}`.
      - Connect AI Language Model: `OpenAI Chat Model1`
      - Connect Output Parser: `Structured Output Parser1`
   8. Add **Airtable** node “YouTube Airtable” (Create):
      - Credential: Airtable Personal Access Token
      - Base: AI Content Hub (or your copy)
      - Table: YouTube Videos
      - Map fields (URL, Title, Views, Channel, Thumbnail, Quick Summary, Deep Dive Summary) using the same expressions.
   9. Add **Aggregate** “Aggregate2”:
      - Mode: aggregate all item data
      - Destination: `youtubeData`
   10. Connect: Trigger → Get Videos → Get Video Data → Filter Out Shorts (true) → Get Transcripts → Transcript Analysis → (a) YouTube Airtable and (b) Aggregate2.

4) **Reddit branch (fetch → summarize → Airtable → aggregate → set)**
   1. Add **Reddit** node “n8n Trending”:
      - Operation: Get All
      - Subreddit: `n8n`
      - Category: `rising`
      - Limit: 5
      - Credential: Reddit OAuth2.
   2. Add **OpenAI Chat Model** “OpenAI Chat Model” (`gpt-4.1-mini`).
   3. Add **Structured Output Parser2** expecting array of `{title, link, summary}`.
   4. Add **LLM Chain** “Reddit Analysis”:
      - Text: include `selftext`, `title`, `url`
      - Prompt: one sentence summary + include title and URL
      - Attach model and parser.
   5. Add **Airtable** “Reddit Airtable” (Create):
      - Base/table: AI Content Hub → Reddit Posts
      - Map URL/Title/Summary from LLM output; Upvotes from original Reddit item.
   6. Add **Aggregate** “Aggregate” (aggregate all item data).
   7. Add **Set** “Edit Fields2”:
      - Set `redditData = {{$json.data}}`
   8. Connect: Trigger → n8n Trending → Reddit Analysis → (a) Reddit Airtable and (b) Aggregate → Edit Fields2.

5) **X branch (Apify scrape → aggregate → analyze → Airtable + set)**
   1. Add **HTTP Request** “Scrape X” (Apify tweet scraper):
      - POST `https://api.apify.com/v2/acts/apidojo~tweet-scraper/run-sync-get-dataset-items`
      - Header: `Authorization: Bearer <APIFY_TOKEN>`
      - JSON body: keywords array and thresholds (min favorites/replies/retweets), date window last 1 day, maxItems 50.
   2. Add **Aggregate** “Aggregate1” (aggregate all item data).
   3. Add **OpenAI Chat Model** “OpenAI Chat Model2” (`gpt-4.1-mini`).
   4. Add **Structured Output Parser3** expecting object with `top_tweets`, `trending_topics`, `metadata`.
   5. Add **LLM Chain** “Twitter Analysis”:
      - Text: `{{$json.data}}`
      - Prompt: rank top 5 by likes; output format (then rely on parser to enforce JSON).
      - Attach model + parser.
   6. Add **Split Out** “Split Out”:
      - Field: `output.top_tweets`
   7. Add **Airtable** “X Airtable” (Create):
      - Table: Tweets
      - Map URL, Like Count, Tweet Text from split items.
   8. Add **Set** “Edit Fields”:
      - Set `xData = {{$json.output}}`
   9. Connect: Trigger → Scrape X → Aggregate1 → Twitter Analysis → (a) Split Out → X Airtable and (b) Edit Fields.

6) **Perplexity branch (ask → normalize → Airtable + set)**
   1. Add **Perplexity** node “Perplexity AI News”:
      - Model: `sonar-pro`
      - Prompt: “top 3 trending news stories… today {{$now}}…”
      - Credential: Perplexity API key.
   2. Add **OpenAI Chat Model** “OpenAI Chat Model3” (`gpt-4.1-mini`).
   3. Add **Structured Output Parser4** expecting array of `{headline, content}`.
   4. Add **LLM Chain** “Format Message”:
      - Input text: `{{$json.message}}`
      - Prompt: extract exactly 3 stories, remove citations, JSON-only
      - Attach model + parser.
   5. Add **Split Out** “Split Out1”:
      - Field: `output`
      - Include all other fields: on
   6. Add **Airtable** “Perplexity Airtable” (Create):
      - Table: Perplexity News
      - Map Headline and Summary from the split object.
   7. Add **Set** “Edit Fields1”:
      - Set `perplexityData = {{$json.message}}`
   8. Connect: Trigger → Perplexity AI News → (a) Format Message → Split Out1 → Perplexity Airtable and (b) Edit Fields1.

7) **Merge + aggregate allData**
   1. Add **Merge** node “Merge” with **4 inputs**.
   2. Connect:
      - Input 0: `Aggregate2` (youtubeData)
      - Input 1: `Edit Fields2` (redditData)
      - Input 2: `Edit Fields` (xData)
      - Input 3: `Edit Fields1` (perplexityData)
   3. Add **Aggregate** node “Aggregate3”:
      - Aggregate all item data
      - Destination field: `allData`
   4. Connect: Merge → Aggregate3.

8) **Generate + send HTML report**
   1. Add **OpenRouter Chat Model** “OpenRouter Chat Model”:
      - Model: `openai/gpt-4.1`
      - Credential: OpenRouter API key.
   2. Add **LLM Chain** “Report Generator”:
      - Input: `{{$json.allData}}`
      - Prompt: HTML-only digest spec (as in workflow).
      - Attach model: OpenRouter Chat Model.
   3. Add **Gmail** node “Send a message”:
      - To: your email
      - Subject: `Daily Content Report {{$now.toFormat('yyyy-MM-dd')}}`
      - Message/body: `{{$json.text}}`
      - Credential: Gmail OAuth2.
   4. Connect: Aggregate3 → Report Generator → Send a message.

9) **Generate + store content ideas**
   1. Add **Structured Output Parser** “Structured Output Parser” (complex schema for ideas).
   2. Add **LLM Chain** “Content Brainstorm”:
      - Input: `{{$json.allData}}`
      - Prompt: content strategist system prompt requesting 5 ideas
      - Attach model: OpenRouter Chat Model
      - Attach parser: Structured Output Parser
   3. Add **Split Out** “Ideas”:
      - Field: `output.content_ideas`
   4. Add **Airtable** node “Create a record”:
      - Table: Content Ideas
      - Map Idea Title/Description/Why/Hook/Storyline using nested fields.
   5. Connect: Aggregate3 → Content Brainstorm → Ideas → Create a record.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## 📄 Daily Content Report — Created by Chase AI” | https://www.skool.com/chase-ai |
| APIs needed: Apify, YouTube, Reddit, Airtable | Apify: https://console.apify.com/settings/integrations ; YouTube creds doc: https://docs.n8n.io/integrations/builtin/credentials/google/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal ; Reddit creds doc: https://docs.n8n.io/integrations/builtin/credentials/reddit/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal ; Airtable creds doc: https://docs.n8n.io/integrations/builtin/credentials/airtable/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal |
| Apify actors referenced | YouTube Transcript Scraper: https://console.apify.com/actors/1s7eXiaukVuOr4Ueg/input ; Twitter Scraper: https://console.apify.com/actors/61RPP7dywgiy0JPD0/input |
| Google Cloud console needed for YouTube Data API key | https://console.cloud.google.com/ |
| Airtable template base | https://airtable.com/appsi00aU0KfhF76Z/shrUtGawO8D1DoaO4 |
| Estimated per-run costs (from sticky note) | X Scraper $0.02; YouTube Scraper $0.07; Reddit $0; Perplexity $0.01; LLMs $0.22; Total ~$0.32 |
| Disclaimer (provided by user) | Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… (legal/public data; compliant content) |
| “@[youtube](m-7egcppFPk)” | Present as a sticky note (appears to be an internal reference tag) |