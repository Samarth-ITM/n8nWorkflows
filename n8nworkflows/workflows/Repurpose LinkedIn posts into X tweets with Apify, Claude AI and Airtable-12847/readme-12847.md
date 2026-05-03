Repurpose LinkedIn posts into X tweets with Apify, Claude AI and Airtable

https://n8nworkflows.xyz/workflows/repurpose-linkedin-posts-into-x-tweets-with-apify--claude-ai-and-airtable-12847


# Repurpose LinkedIn posts into X tweets with Apify, Claude AI and Airtable

## 1. Workflow Overview

**Title:** Repurpose LinkedIn posts into X tweets with Apify, Claude AI and Airtable  
**Workflow name (in JSON):** `TEMPL - Repurpose LK posts to X`

**Purpose:**  
This workflow automates a content pipeline that (1) scrapes recent LinkedIn posts, (2) optionally extracts text from LinkedIn carousel/PDF documents, (3) converts each LinkedIn post into two X formats (one standalone tweet + one thread of 3–7 tweets) using Claude via OpenRouter, (4) stores results in Airtable for review/approval, and (5) posts approved content to X either on a daily schedule or instantly via an Airtable “TWEET NOW” button.

### 1.1 Entry Points (3)
- **Weekly schedule trigger:** scrapes LinkedIn posts and generates tweet drafts.
- **Daily schedule trigger:** posts approved tweets whose publication date is due.
- **Webhook trigger:** posts one specific Airtable tweet record immediately (“TWEET NOW”).

### 1.2 Logical Blocks
1. **Configuration loading (Airtable → runtime config)**
2. **LinkedIn scraping (Apify) + post dedup/check**
3. **Carousel/PDF extraction (HTTP download → OpenAI vision/OCR → JSON parsing)**
4. **Persist LinkedIn post (Airtable: LK Posts)**
5. **LLM conversion (Claude/OpenRouter) + structured parsing**
6. **Persist tweets (Airtable: X Tweets) + update LK post status**
7. **Publishing (daily batch or manual webhook)**
   - Standalone tweet publishing
   - Thread publishing (reply chain) using a temporary stored tweet_id

---

## 2. Block-by-Block Analysis

### Block 1 — Weekly trigger + config loading
**Overview:** Loads workflow parameters from Airtable “Config” table and prepares a key/value config object used throughout the scraping and scheduling logic.

**Nodes involved:**
- `Weekly_OnSunday`
- `GetConfig`
- `FormatConfig`

#### Node: Weekly_OnSunday
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) – starts the weekly scraping pipeline.
- **Configuration:** Runs every week (the sticky note says “Sunday at midnight”; the node rule is weekly interval—exact day/time is determined by the node config in UI and instance timezone).
- **Outputs to:** `GetConfig`
- **Edge cases / failures:**
  - Timezone mismatch (instance timezone vs expected “Sunday midnight”).
  - If the workflow is inactive, nothing runs.

#### Node: GetConfig
- **Type / role:** Airtable Search (`n8n-nodes-base.airtable`) – reads all rows from Config table.
- **Configuration choices:**
  - Base: `LinkedIn2Twitter` (`appiS2JpMdnxAS5o4`)
  - Table: `Config` (`tbl7B3JblukFTGrAo`)
  - Operation: `search` (returns all config records)
- **Outputs to:** `FormatConfig`
- **Failure types:**
  - Airtable auth/token invalid.
  - Base/table ID mismatch.
  - API rate limiting.

#### Node: FormatConfig
- **Type / role:** Code node (`n8n-nodes-base.code`) – transforms Config rows into a single JSON object.
- **Key logic:** Builds `config[name] = value` depending on `Type` being `Integer` or `String`, using either `Integer Value` or `String Value`.
- **Output:** a single object like:
  - `profile_url`
  - `posted_limit` (e.g. `week`)
  - `max_posts` (integer)
  - plus other keys used later (notably `schedule_tweets_days_after_lk`)
- **Outputs to:** `ScrapeLastPosts`
- **Edge cases:**
  - Missing config rows/fields → downstream expressions become `undefined`.
  - Wrong `Type` values → falls back to `String Value`.
  - If `schedule_tweets_days_after_lk` is missing, later code defaults to `0`.

---

### Block 2 — Scrape LinkedIn posts (Apify) and iterate
**Overview:** Uses Apify’s LinkedIn scraper actor to fetch recent posts for the configured LinkedIn profile, then iterates through results.

**Nodes involved:**
- `ScrapeLastPosts`
- `LoopOverPosts`

#### Node: ScrapeLastPosts
- **Type / role:** Apify node (`@apify/n8n-nodes-apify.apify`) – runs an actor and returns dataset items.
- **Configuration choices:**
  - Operation: “Run actor and get dataset”
  - Actor: `https://console.apify.com/actors/A3cAPGpwBEG8RJwse`
  - Body uses config expressions:
    - `maxPosts`: from `FormatConfig.max_posts`
    - `postedLimit`: from `FormatConfig.posted_limit`
    - `targetUrls`: `[ FormatConfig.profile_url ]`
  - `executeOnce: true` (prevents repeated runs within a single execution)
  - `alwaysOutputData: true`
- **Outputs to:** `LoopOverPosts`
- **Failure types / edge cases:**
  - Apify token invalid / actor access denied.
  - Actor changes output schema (breaks downstream field references like `id`, `content`, `postedAt.date`, `document.transcribedDocumentUrl`).
  - Timeout / dataset too large.
  - LinkedIn scraping instability (blocked pages, empty results).
  - Cost note (from sticky): “Price: $2 per 1k posts”.

#### Node: LoopOverPosts
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) – processes scraped posts one by one.
- **Connections:**
  - Input: from `ScrapeLastPosts`
  - Output (index 1): to `SearchExistingPost` (main processing path)
  - Output (index 0): loop continuation (wired back later from `Switch` to continue batching)
- **Edge cases:**
  - Empty dataset → no items to process.
  - If batch size defaults, performance depends on dataset volume.

---

### Block 3 — Deduplicate/check post status in Airtable
**Overview:** Checks if a LinkedIn post already exists in Airtable and routes the flow depending on whether it’s new, needs conversion, or was already converted/tweeted.

**Nodes involved:**
- `SearchExistingPost`
- `Switch`

#### Node: SearchExistingPost
- **Type / role:** Airtable Search – find LK post by “LK Post ID”.
- **Configuration:**
  - Base: `LinkedIn2Twitter`
  - Table: `LK Posts` (`tblzy5zMwaTWaTHof`)
  - Filter formula: `={LK Post ID}='{{ $json.id }}'`
  - `alwaysOutputData: true` (important: if not found, node still outputs an item; downstream checks rely on optional chaining)
- **Outputs to:** `Switch`
- **Failure types:**
  - Airtable auth/base/table issues.
  - Formula errors if field names differ.

#### Node: Switch
- **Type / role:** Switch (`n8n-nodes-base.switch`) – routes by “existing record?” and “Status”.
- **Routing rules (interpreted):**
  1. **`no_post_found`:** if `SearchExistingPost.item.json.id` is empty → treat as new post
  2. **`not_converted_yet`:** if `SearchExistingPost.Status == 'Scrapped'`
  3. **`already_converted_or_posted`:** if Status != `Scrapped`
- **Connections:**
  - `no_post_found` → `IfIsCarousel` (start extraction & save)
  - `not_converted_yet` → `MergePost` (skips scraping-save step; reuses Airtable post record)
  - `already_converted_or_posted` → `LoopOverPosts` (skip this post and continue loop)
- **Edge cases:**
  - If Airtable returns multiple matches, `.item` may select one; duplicates can cause inconsistent routing.
  - If LK Posts Status values differ (typo), routing may be wrong.

---

### Block 4 — Extract carousel/PDF content (optional) + merge with post content
**Overview:** If the LinkedIn post contains a document/carousel URL, download it and extract structured slide text using OpenAI (vision/file input). Then merge extracted carousel slides with the original post content so downstream LLM can use both.

**Nodes involved:**
- `IfIsCarousel`
- `GetCarouselFile`
- `ExtractContentFromFile`
- `ParseJSON`
- `MergeContent`

#### Node: IfIsCarousel
- **Type / role:** IF (`n8n-nodes-base.if`) – checks if a carousel/document exists.
- **Condition:** `LoopOverPosts.item.json.document.transcribedDocumentUrl` is not empty.
- **Outputs:**
  - True → `GetCarouselFile`
  - False → `MergeContent` (second input of merge) to proceed without carousel slides
- **Edge cases:**
  - Apify field missing or renamed → condition fails, carousel is ignored.

#### Node: GetCarouselFile
- **Type / role:** HTTP Request – downloads the carousel document.
- **Configuration:**
  - URL: `LoopOverPosts.item.json.document.transcribedDocumentUrl`
  - Sends browser-like headers (Accept, Encoding, User-Agent etc.)
  - `alwaysOutputData: true`
- **Outputs to:** `ExtractContentFromFile`
- **Failure types:**
  - 403/404 from LinkedIn/Apify CDN.
  - Non-PDF content or corrupted file.
  - Large file download issues.

#### Node: ExtractContentFromFile
- **Type / role:** OpenAI node (`@n8n/n8n-nodes-langchain.openAi`) – OCR/layout extraction from the PDF.
- **Configuration choices:**
  - Model: `chatgpt-4o-latest`
  - Sends a **file** input (`$binary.data`) and a detailed instruction prompt requiring **pure JSON output**:
    - schema `{ "slides": [ { slide_number, type, content } ] }`
    - slide types: `front|content|cta`
- **Outputs to:** `ParseJSON`
- **Failure types / edge cases:**
  - OpenAI credential/model access issues.
  - Response not valid JSON despite instructions → handled by `ParseJSON` but can still throw if irrecoverable.
  - Missing binary data (if HTTP node not configured to download as binary).

#### Node: ParseJSON
- **Type / role:** Code node – extracts JSON from model output.
- **Logic:**
  - Reads `text = $input.first().json?.output[0]?.content[0]?.text`
  - If markdown code block exists (```json ... ```), extracts inside; else parses raw text.
  - Returns `{ slides: parsedJson }`
- **Outputs to:** `MergeContent`
- **Edge cases:**
  - If model output is empty, returns `{ slides: {} }` (downstream expects `.slides`, may become `{}` not `[]`).
  - Invalid JSON will throw and fail execution (no try/catch).

#### Node: MergeContent
- **Type / role:** Merge (`n8n-nodes-base.merge`) – combines either:
  - the parsed slide extraction output (true branch), or
  - a “no carousel” pass-through (false branch)
- **Configuration:** default merge behavior (not explicitly set in JSON).
- **Outputs to:** `SavePost`
- **Edge cases:**
  - Merge behavior matters: if the “no carousel” path doesn’t provide compatible fields, you can end with unexpected structure.
  - `alwaysOutputData: true` set on this node helps keep downstream alive.

---

### Block 5 — Save LinkedIn post into Airtable and prepare for conversion
**Overview:** Stores the scraped LinkedIn post (including optional carousel content) in the `LK Posts` table, then merges that stored record into the conversion path.

**Nodes involved:**
- `SavePost`
- `MergePost`

#### Node: SavePost
- **Type / role:** Airtable Create – creates a new LK post record.
- **Configuration:**
  - Base: `LinkedIn2Twitter`
  - Table: `LK Posts`
  - Fields written:
    - `Date` = `LoopOverPosts.postedAt.date`
    - `Status` = `"Scrapped"`
    - `Content` = `LoopOverPosts.content`
    - `Post URL` = `LoopOverPosts.linkedinUrl`
    - `LK Post ID` = `LoopOverPosts.id`
    - `Post Doc URL` = `LoopOverPosts.document.transcribedDocumentUrl` (optional)
    - `Post Img URL` = first image URL (optional)
    - `Carousel Content` = `JSON.stringify(MergeContent.item.json.slides ?? null)`
- **Outputs to:** `MergePost`
- **Failure types:**
  - Airtable schema mismatch (field names must match exactly).
  - Overlong strings (carousel JSON may be large for Airtable field limits depending on configuration).
  - Duplicate LK Post ID: this workflow creates new records only on “no_post_found”; duplicates still possible if formula search fails.

#### Node: MergePost
- **Type / role:** Merge – normalizes two possible sources for the “post to convert”:
  - From `SavePost` (newly created record), or
  - From `Switch` “not_converted_yet” (existing Airtable LK post record)
- **Outputs to:** `ComputeTweetDate`
- **Edge cases:**
  - Downstream expressions reference both `json.Content` and `json.fields.Content` to handle different shapes—this merge is why.

---

### Block 6 — Compute tweet publication date (relative scheduling)
**Overview:** Computes the default tweet publication date as LinkedIn post date plus `schedule_tweets_days_after_lk` days from Config.

**Nodes involved:**
- `ComputeTweetDate`

#### Node: ComputeTweetDate
- **Type / role:** Code node – date arithmetic using Luxon `DateTime`.
- **Key expressions:**
  - `nbDaysAfter = FormatConfig.schedule_tweets_days_after_lk || 0`
  - `lkDate = DateTime.fromISO(MergePost.first().json.Date || MergePost.first().json.fields.Date)`
  - returns `{ tweet_post_date: lkDate.plus({days: nbDaysAfter}).toISO() }`
- **Outputs to:** `ConvertPostIntoTweets`
- **Failure types / edge cases:**
  - If `DateTime` isn’t available (older n8n or different runtime), this will fail. (Most modern n8n code nodes provide Luxon.)
  - If LK post date is not ISO parseable, `fromISO` yields invalid date.
  - If config returns a string for `schedule_tweets_days_after_lk`, adding days may behave unexpectedly unless coerced to number.

---

### Block 7 — Convert LinkedIn post into tweets (Claude via OpenRouter) + structured parsing
**Overview:** Uses a LangChain “Chain LLM” node with Claude Sonnet 4.5 (OpenRouter) and a structured output parser to generate exactly two variations: (1) a standalone tweet and (2) a 3–7 tweet thread, returned as validated JSON.

**Nodes involved:**
- `Claude-4.5-Sonnet`
- `Structured Output Parser`
- `ConvertPostIntoTweets`

#### Node: Claude-4.5-Sonnet
- **Type / role:** OpenRouter chat model connector (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)
- **Configuration:**
  - Model: `anthropic/claude-sonnet-4.5`
- **Connected as:** `ai_languageModel` input to `ConvertPostIntoTweets`
- **Failure types:**
  - OpenRouter auth issues, insufficient credits, model unavailable.
  - Provider latency/timeouts.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)
- **Configuration:** Provides a JSON schema example that matches the expected structure:
  - `thread_variation.tweets[]` with `tweet_number`, `content`, `character_count`
  - `standalone_variation.content`, etc.
- **Connected as:** `ai_outputParser` input to `ConvertPostIntoTweets`
- **Edge cases:**
  - If the LLM output deviates, parser can fail and cause retries (the chain node has retry enabled).

#### Node: ConvertPostIntoTweets
- **Type / role:** LangChain Chain (`@n8n/n8n-nodes-langchain.chainLlm`) – runs the prompt + model + parser.
- **Configuration highlights:**
  - Very explicit prompt with constraints:
    - Exactly 2 variations (thread + standalone)
    - Tweet length rules, tone shift, max 2 hashtags, hook formulas, etc.
  - Inputs are built from merged post record:
    - LinkedIn post content:
      - `MergePost.item.json.Content || MergePost.item.json.fields.Content`
    - Carousel content (optional):
      - `MergePost.item.json['Carousel Content'] || MergePost.item.json.fields['Carousel Content']`
  - `hasOutputParser: true`
  - `retryOnFail: true`
- **Outputs to:**
  - `CreateStandaloneTweet` (standalone path)
  - `SplitThreadVariations` (thread tweets path)
- **Failure types / edge cases:**
  - If post content is empty/too short, prompt instructs to return an error string—but the structured parser expects schema, so this can hard-fail parsing unless the model still returns schema.
  - If carousel content is stored as a JSON-stringified value, the model receives a string representation (fine, but may include escape characters).
  - Character count accuracy depends on the model; workflow does not re-verify counts.

---

### Block 8 — Save generated tweets to Airtable + update LK post status
**Overview:** Stores the standalone tweet and each thread tweet in Airtable `X Tweets` with `Pending` status and a computed publication date; then updates the LinkedIn post record to `Converted`.

**Nodes involved:**
- `SplitThreadVariations`
- `CreateThreadTweet`
- `CreateStandaloneTweet`
- `UpdatePostStatus`
- `MergeTweets`

#### Node: SplitThreadVariations
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) – converts `thread_variation.tweets[]` into one item per tweet.
- **Field to split:** `output.thread_variation.tweets`
- **Outputs to:** `CreateThreadTweet`
- **Edge cases:**
  - If `thread_variation.tweets` missing/not an array → node fails or outputs nothing.

#### Node: CreateStandaloneTweet
- **Type / role:** Airtable Create – saves standalone tweet draft.
- **Configuration:**
  - Table: `X Tweets` (`tblGhyo7vlTUNkfoX`)
  - Fields:
    - `Status` = `Pending`
    - `Content` = `output.standalone_variation.content`
    - `Tweet Nb` = `0`
    - `Variation` = `Standalone`
    - `LK Post ID` = `LoopOverPosts.id`
    - `Publication Date` = `ComputeTweetDate.tweet_post_date`
- **Outputs to:** `UpdatePostStatus`
- **Failure types:**
  - Airtable schema mismatch / validation.
  - Content too long (should be <= 280 but not enforced here).

#### Node: CreateThreadTweet
- **Type / role:** Airtable Create – saves each thread tweet draft.
- **Configuration:**
  - Fields:
    - `Status` = `Pending`
    - `Content` = per-item `$json.content`
    - `Tweet Nb` = per-item `$json.tweet_number`
    - `Variation` = `Thread`
    - `LK Post ID` = `LoopOverPosts.id`
    - `Publication Date` = computed date
- **Outputs to:** `MergeTweets`
- **Edge cases:**
  - Tweet numbering assumed present and ordered; later posting sorts by `Tweet Nb`.

#### Node: UpdatePostStatus
- **Type / role:** Airtable Update – marks LK post as `Converted`.
- **Configuration:**
  - Table: `LK Posts`
  - Matching column: `LK Post ID`
  - Sets `Status = Converted`
- **Outputs to:** `MergeTweets`
- **Edge cases:**
  - If multiple LK post records share same LK Post ID, update may target unexpected record.

#### Node: MergeTweets
- **Type / role:** Merge – joins:
  - Thread tweet creates and
  - LK post status update
  so the loop can continue.
- **Outputs to:** `LoopOverPosts` (to process next scraped post)
- **Edge cases:**
  - Merge behavior can affect whether loop continues if one branch produces no items.

---

### Block 9 — Daily publishing trigger: fetch approved tweets due now
**Overview:** Each day at 12:30, retrieves all tweets to publish: approved, publication date before now, and either standalone tweets or the first tweet of each thread.

**Nodes involved:**
- `Daily_AtNoon`
- `SearchApprovedTweets`
- `SwitchStatusAndVariation`

#### Node: Daily_AtNoon
- **Type / role:** Schedule Trigger – daily publishing entry point.
- **Configuration:** Triggers at 12:30 (hours/minutes set).
- **Outputs to:** `SearchApprovedTweets`
- **Edge cases:** timezone mismatch.

#### Node: SearchApprovedTweets
- **Type / role:** Airtable Search – finds tweets ready to post.
- **Filter formula:**
  - `Status='Approved'`
  - `Publication Date` is before `NOW()`
  - and `(Variation='Standalone' OR Tweet Nb = 1)` (only first tweet of a thread is selected)
- **Sort:** by `Tweet Nb`
- **Outputs to:** `SwitchStatusAndVariation`
- **Failure types:** Airtable rate limits, formula mismatches, date field formatting issues.

#### Node: SwitchStatusAndVariation
- **Type / role:** Switch – ensures status is Approved and routes by Variation.
- **Rules:**
  - `standalone`: Approved + Standalone
  - `thread`: Approved + Thread
  - `not_approved`: Status !== Approved (safety route; not connected further in this workflow)
- **Outputs to:**
  - Standalone → `PostStandaloneTweet`
  - Thread → `InitTemporaryTweetId`
- **Edge cases:**
  - If Variation values differ from expected (`Standalone`/`Thread`), tweets won’t post.

---

### Block 10 — Manual publishing via webhook (“TWEET NOW”)
**Overview:** Airtable button calls an n8n webhook with a record ID; the workflow loads that tweet record and posts it if approved, using the same posting logic as the daily job.

**Nodes involved:**
- `Webhook_OnPostTweet`
- `GetTweet`
- `SwitchStatusAndVariation`

#### Node: Webhook_OnPostTweet
- **Type / role:** Webhook (`n8n-nodes-base.webhook`) – manual trigger endpoint.
- **Configuration:**
  - Path: `post-tweet`
  - Expects query parameter `id` (Airtable record ID)
- **Outputs to:** `GetTweet`
- **Failure types / edge cases:**
  - Anyone with URL could trigger unless protected (consider auth or secret token).
  - Missing `id` → `GetTweet` fails.

#### Node: GetTweet
- **Type / role:** Airtable Read/Get record (configured via `id`).
- **Configuration:**
  - `id = $json.query.id`
  - Table: `X Tweets`
- **Outputs to:** `SwitchStatusAndVariation`
- **Edge cases:**
  - Invalid record ID → Airtable error.
  - Note: Base cached name shows “Growth Machine” but base ID is the same `appiS2JpMdnxAS5o4`; if base actually differs, this would break.

---

### Block 11 — Post a standalone tweet + update status
**Overview:** Posts a single tweet to X and marks the Airtable tweet record as `Tweeted`.

**Nodes involved:**
- `PostStandaloneTweet`
- `UpdateStandaloneTweetStatus`

#### Node: PostStandaloneTweet
- **Type / role:** X/Twitter node (`n8n-nodes-base.twitter`) – creates a tweet.
- **Configuration:** `text = $json.Content`
- **Outputs to:** `UpdateStandaloneTweetStatus`
- **Failure types:**
  - X OAuth2 credential invalid/expired.
  - Tweet too long / duplicate / policy restrictions.
  - API errors (rate limit, 403).

#### Node: UpdateStandaloneTweetStatus
- **Type / role:** Airtable Update – sets tweet record `Status=Tweeted`.
- **Configuration:**
  - Uses `id = {{ $json.query.id }}` and matches on `id`
  - Sets `Status = Tweeted`
- **Important edge case (design issue):**
  - After `PostStandaloneTweet`, the incoming `$json` is typically the Twitter API response, **not** the original webhook query. If this path is triggered by the daily schedule, there is no `query.id` at all.
  - Result: this update may fail or update nothing unless the data shape happens to still contain `query.id`.  
  **Mitigation:** store Airtable record ID in a dedicated field before posting, or use “Keep Input Data” / merge original item back before update.

---

### Block 12 — Post a thread (reply chain) + update status using a temporary tweet_id
**Overview:** For a thread, the workflow fetches all tweets in that thread from Airtable, posts tweet #1, then posts subsequent tweets as replies to the last posted tweet. It stores the latest tweet ID in an n8n Data Table (`temporary_variables`) under key `tweet_id`.

**Nodes involved:**
- `InitTemporaryTweetId`
- `SearchAllThreadTweets`
- `LoopOverTweets`
- `GetLastTweetId`
- `IfEmpty`
- `PostFirstThreadTweet`
- `PostReplyTweet`
- `Merge`
- `UpdateTemporaryTweetId`
- `UpdateThreadTweetStatus`

#### Node: InitTemporaryTweetId
- **Type / role:** Data Table upsert (`n8n-nodes-base.dataTable`) – resets/initializes `tweet_id` to empty.
- **Configuration:**
  - Data table: `temporary_variables` (`P1OpzB1QUCvmqmuk`)
  - Upserts row where `name='tweet_id'` with `value=''`
- **Outputs to:** `SearchAllThreadTweets`
- **Edge cases:**
  - Concurrency risk: If two threads post simultaneously, they share the same `tweet_id` key and will overwrite each other. Use per-execution/per-thread keys (e.g., `tweet_id_<LK Post ID>`).

#### Node: SearchAllThreadTweets
- **Type / role:** Airtable Search – fetches all tweets for the same LK Post ID and Variation=Thread.
- **Filter formula:**
  - `LK Post ID` equals current item’s LK Post ID
  - `Variation` equals current Variation (Thread)
- **Sort:** by `Tweet Nb`
- **Outputs to:** `LoopOverTweets`
- **Failure types:** Airtable errors, missing LK Post ID.

#### Node: LoopOverTweets
- **Type / role:** Split In Batches – iterates through the thread tweets in order.
- **Outputs:** second output goes to `GetLastTweetId` for each tweet item; loop continues after status update.

#### Node: GetLastTweetId
- **Type / role:** Data Table get – reads current `tweet_id`.
- **Outputs to:** `IfEmpty`

#### Node: IfEmpty
- **Type / role:** IF – decides whether this is the first tweet in the thread.
- **Condition:** `GetLastTweetId.value` is empty.
- **True →** `PostFirstThreadTweet`  
- **False →** `PostReplyTweet`

#### Node: PostFirstThreadTweet
- **Type / role:** Twitter post – posts tweet #1.
- **Text:** `LoopOverTweets.item.json.Content`
- **Outputs to:** `Merge` (input 0)

#### Node: PostReplyTweet
- **Type / role:** Twitter post reply – posts a reply to the stored last tweet id.
- **Text:** `LoopOverTweets.item.json.Content`
- **inReplyToStatusId:** `GetLastTweetId.item.json.value`
- **Outputs to:** `Merge` (input 1)
- **Failure types:**
  - If stored tweet_id is invalid/empty, reply fails.
  - If X API does not return `id` in the expected field, next steps break.

#### Node: Merge
- **Type / role:** Merge – normalizes outputs from “first tweet” vs “reply tweet” into a single stream.
- **Outputs to:** `UpdateTemporaryTweetId`

#### Node: UpdateTemporaryTweetId
- **Type / role:** Data Table upsert – saves the newly posted tweet id.
- **Value:** `value = $json.id` (expects Twitter response contains `id`)
- **Outputs to:** `UpdateThreadTweetStatus`
- **Edge cases:**
  - Twitter node response schema differences (might be `id_str` or nested).
  - Concurrency risk as noted above.

#### Node: UpdateThreadTweetStatus
- **Type / role:** Airtable Update – marks the current thread tweet record as `Tweeted`.
- **ID:** `LoopOverTweets.item.json.id` (Airtable record id)
- **Outputs to:** `LoopOverTweets` to continue
- **Failure types:** Airtable update errors; missing `id` if search returned unexpected data.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow description & setup notes |  |  | # Repurpose LinkedIn posts to X (Twitter)… Airtable template link: https://airtable.com/appiS2JpMdnxAS5o4/shrXh9iXTnL9fX63L/tbl7B3JblukFTGrAo/viwgajzRGnydUDjnw |
| Weekly_OnSunday | Schedule Trigger | Weekly pipeline entrypoint |  | GetConfig | ##  1. Weekly Trigger\n\nEach Sunday at midnight |
| GetConfig | Airtable | Load config table | Weekly_OnSunday | FormatConfig | ## 2. Get config |
| FormatConfig | Code | Build config object | GetConfig | ScrapeLastPosts | ## 2. Get config |
| ScrapeLastPosts | Apify | Scrape LinkedIn posts dataset | FormatConfig | LoopOverPosts | ## 3. Scrape LinkedIn posts\n\nApify API doc here:\nhttps://console.apify.com/actors/A3cAPGpwBEG8RJwse/information/latest/readme\n\nPrice: $2 per 1k posts |
| LoopOverPosts | Split In Batches | Iterate over scraped posts | ScrapeLastPosts / MergeTweets | SearchExistingPost / (loop) | ## 4. Extract LinkedIn post and carousel content |
| SearchExistingPost | Airtable | Deduplicate / check status | LoopOverPosts | Switch | ## 4. Extract LinkedIn post and carousel content |
| Switch | Switch | Route by existence/status | SearchExistingPost | IfIsCarousel / MergePost / LoopOverPosts | ## 4. Extract LinkedIn post and carousel content |
| IfIsCarousel | IF | Detect document/carousel | Switch | GetCarouselFile / MergeContent | ## 4. Extract LinkedIn post and carousel content |
| GetCarouselFile | HTTP Request | Download carousel PDF | IfIsCarousel | ExtractContentFromFile | ## 4. Extract LinkedIn post and carousel content |
| ExtractContentFromFile | OpenAI (LangChain) | OCR/layout extraction to JSON | GetCarouselFile | ParseJSON | ## 4. Extract LinkedIn post and carousel content |
| ParseJSON | Code | Parse JSON from model output | ExtractContentFromFile | MergeContent | ## 4. Extract LinkedIn post and carousel content |
| MergeContent | Merge | Combine carousel/no-carousel path | ParseJSON / IfIsCarousel | SavePost | ## 4. Extract LinkedIn post and carousel content |
| SavePost | Airtable | Create LK Post record (Scrapped) | MergeContent | MergePost | ## 4. Save LK posts in Airtable |
| MergePost | Merge | Normalize post source | SavePost / Switch | ComputeTweetDate | ## 5. Convert LK post into tweets and save them in Airtable… |
| ComputeTweetDate | Code | Compute default publication date | MergePost | ConvertPostIntoTweets | ## 5. Convert LK post into tweets and save them in Airtable… |
| Claude-4.5-Sonnet | OpenRouter Chat Model | LLM provider for conversion |  | ConvertPostIntoTweets (ai_languageModel) | ## 5. Convert LK post into tweets and save them in Airtable… |
| Structured Output Parser | Structured Output Parser | Enforce JSON output schema |  | ConvertPostIntoTweets (ai_outputParser) | ## 5. Convert LK post into tweets and save them in Airtable… |
| ConvertPostIntoTweets | LangChain Chain LLM | Generate thread + standalone JSON | ComputeTweetDate | SplitThreadVariations / CreateStandaloneTweet | ## 5. Convert LK post into tweets and save them in Airtable… |
| SplitThreadVariations | Split Out | Expand thread tweets array | ConvertPostIntoTweets | CreateThreadTweet | ## 5. Convert LK post into tweets and save them in Airtable… |
| CreateThreadTweet | Airtable | Create thread tweet drafts | SplitThreadVariations | MergeTweets | ## 5. Convert LK post into tweets and save them in Airtable… |
| CreateStandaloneTweet | Airtable | Create standalone tweet draft | ConvertPostIntoTweets | UpdatePostStatus | ## 5. Convert LK post into tweets and save them in Airtable… |
| UpdatePostStatus | Airtable | Set LK post status Converted | CreateStandaloneTweet | MergeTweets | ## 5. Convert LK post into tweets and save them in Airtable… |
| MergeTweets | Merge | Join status update + thread saves | UpdatePostStatus / CreateThreadTweet | LoopOverPosts | ## 5. Convert LK post into tweets and save them in Airtable… |
| Webhook_OnPostTweet | Webhook | Manual “tweet now” entrypoint |  | GetTweet | ## 1. (A) Manual trigger : when clicked on "TWEET NOW" button in Airtable |
| GetTweet | Airtable | Load one tweet record by id | Webhook_OnPostTweet | SwitchStatusAndVariation | ## 2. (A) Get tweet |
| Daily_AtNoon | Schedule Trigger | Daily publishing entrypoint |  | SearchApprovedTweets | ##  1. (B) Daily trigger |
| SearchApprovedTweets | Airtable | Find due approved tweets | Daily_AtNoon | SwitchStatusAndVariation | ## 2. (B) Get tweets to be posted… |
| SwitchStatusAndVariation | Switch | Route standalone vs thread (Approved only) | GetTweet / SearchApprovedTweets | PostStandaloneTweet / InitTemporaryTweetId | ## 3. Check tweet Variation (standalone/thread) and Status (must be Approved) |
| PostStandaloneTweet | Twitter/X | Publish a single tweet | SwitchStatusAndVariation | UpdateStandaloneTweetStatus | ## 4. Post a standalone tweet |
| UpdateStandaloneTweetStatus | Airtable | Mark standalone tweet Tweeted | PostStandaloneTweet |  | ## 4. Post a standalone tweet |
| InitTemporaryTweetId | Data Table | Reset tweet_id for thread posting | SwitchStatusAndVariation | SearchAllThreadTweets | ## 4. Post a thread of tweets |
| SearchAllThreadTweets | Airtable | Load all tweets in thread | InitTemporaryTweetId | LoopOverTweets | ## 4. Post a thread of tweets |
| LoopOverTweets | Split In Batches | Iterate thread tweets | SearchAllThreadTweets / UpdateThreadTweetStatus | GetLastTweetId / (loop) | ## 4. Post a thread of tweets |
| GetLastTweetId | Data Table | Read last posted tweet id | LoopOverTweets | IfEmpty | ## 4. Post a thread of tweets |
| IfEmpty | IF | Decide first tweet vs reply | GetLastTweetId | PostFirstThreadTweet / PostReplyTweet | ## 4. Post a thread of tweets |
| PostFirstThreadTweet | Twitter/X | Post first tweet of thread | IfEmpty | Merge | ## 4. Post a thread of tweets |
| PostReplyTweet | Twitter/X | Post reply in thread | IfEmpty | Merge | ## 4. Post a thread of tweets |
| Merge | Merge | Unify first/reply tweet responses | PostFirstThreadTweet / PostReplyTweet | UpdateTemporaryTweetId | ## 4. Post a thread of tweets |
| UpdateTemporaryTweetId | Data Table | Store new last tweet id | Merge | UpdateThreadTweetStatus | ## 4. Post a thread of tweets |
| UpdateThreadTweetStatus | Airtable | Mark each thread tweet Tweeted | UpdateTemporaryTweetId | LoopOverTweets | ## 4. Post a thread of tweets |
| Sticky Note1 | Sticky Note | Section label |  |  | ## 4. Save LK posts in Airtable |
| Sticky Note2 | Sticky Note | Section label |  |  | ## 5. Convert LK post into tweets… |
| Sticky Note3 | Sticky Note | Section label |  |  | ## 4. Extract LinkedIn post and carousel content |
| Sticky Note4 | Sticky Note | Section label |  |  | ## 1. (A) Manual trigger… |
| Sticky Note5 | Sticky Note | Section label |  |  | ## 4. Post a standalone tweet |
| Sticky Note6 | Sticky Note | Section label |  |  | ## 4. Post a thread of tweets |
| Sticky Note7 | Sticky Note | Section label |  |  | ## 2. Get config |
| Sticky Note8 | Sticky Note | Section label |  |  | ## 2. (B) Get tweets to be posted… |
| Sticky Note9 | Sticky Note | Section label |  |  | ##  1. (B) Daily trigger |
| Sticky Note10 | Sticky Note | Section label |  |  | ## 2. (A) Get tweet |
| Sticky Note12 | Sticky Note | Section label |  |  | ##  1. Weekly Trigger… |
| Sticky Note13 | Sticky Note | Section label |  |  | ## 3. Check tweet Variation… |
| Sticky Note17 | Sticky Note | Apify info |  |  | ## 3. Scrape LinkedIn posts… https://console.apify.com/actors/A3cAPGpwBEG8RJwse/information/latest/readme |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Airtable base and tables**
   1. Create/duplicate a base with 3 tables:
      - `Config`
      - `LK Posts`
      - `X Tweets`  
      Suggested template link (from note):  
      https://airtable.com/appiS2JpMdnxAS5o4/shrXh9iXTnL9fX63L/tbl7B3JblukFTGrAo/viwgajzRGnydUDjnw
   2. Ensure fields exist (names must match exactly):
      - **Config**: `Name`, `Type`, `Integer Value`, `String Value`
      - **LK Posts**: `LK Post ID`, `Status` (Scrapped/Converted/Tweeted), `Date`, `Content`, `Carousel Content`, `Post URL`, `Post Img URL`, `Post Doc URL`
      - **X Tweets**: `Status` (Pending/Rejected/Approved/Tweeted), `Variation` (Standalone/Thread), `Tweet Nb`, `Content`, `Publication Date`, `LK Post ID`
   3. In `Config`, add rows such as:
      - `profile_url` (String)
      - `posted_limit` (String, e.g. `week`)
      - `max_posts` (Integer)
      - `schedule_tweets_days_after_lk` (Integer)

2. **Create credentials in n8n**
   - **Airtable Personal Access Token** credential (for all Airtable nodes).
   - **Apify API** credential.
   - **OpenAI API** credential (for carousel OCR).
   - **OpenRouter API** credential (for Claude).
   - **X (Twitter) OAuth2** credential (for posting tweets).

3. **Weekly scraping pipeline nodes**
   1. Add **Schedule Trigger** named `Weekly_OnSunday`; set to weekly on Sunday at desired time/timezone.
   2. Add **Airtable node** `GetConfig` (Operation: Search) pointing to your base + `Config` table.
   3. Add **Code node** `FormatConfig`:
      - Read all items from `GetConfig`
      - Build `config[name]` using `Type` to pick `Integer Value` or `String Value`
      - Return the config object.
   4. Add **Apify node** `ScrapeLastPosts`:
      - Operation: *Run actor and get dataset*
      - Actor: `A3cAPGpwBEG8RJwse`
      - Body includes `maxPosts`, `postedLimit`, `targetUrls` using expressions from `FormatConfig`.
   5. Add **Split In Batches** `LoopOverPosts` to iterate posts.

4. **Post deduplication**
   1. Add **Airtable Search** `SearchExistingPost` on table `LK Posts`
      - Filter formula: `={LK Post ID}='{{ $json.id }}'`
      - Enable “Always Output Data”.
   2. Add **Switch** `Switch` with outputs:
      - `no_post_found` when `SearchExistingPost.item.json.id` is empty
      - `not_converted_yet` when `SearchExistingPost.Status == 'Scrapped'`
      - `already_converted_or_posted` otherwise
   3. Connect:
      - `already_converted_or_posted` → back to `LoopOverPosts` (continue loop)

5. **Optional carousel extraction**
   1. Add **IF** `IfIsCarousel` checking `LoopOverPosts.item.json.document.transcribedDocumentUrl` not empty.
   2. True path:
      - Add **HTTP Request** `GetCarouselFile` to download that URL (configure binary download if needed).
      - Add **OpenAI (LangChain) node** `ExtractContentFromFile` with model `chatgpt-4o-latest` and “file” input from `$binary`.
      - Add **Code** `ParseJSON` to parse the model output into `{ slides: ... }`.
   3. Add **Merge** `MergeContent` to merge carousel/no-carousel paths.

6. **Save LK post**
   1. Add **Airtable Create** `SavePost` to `LK Posts`, mapping fields from `LoopOverPosts` and `MergeContent` (carousel slides JSON string).
   2. Add **Merge** `MergePost`:
      - Input A: from `SavePost`
      - Input B: from `Switch` → `not_converted_yet` (existing record path)

7. **Compute publication date**
   1. Add **Code node** `ComputeTweetDate` using Luxon:
      - `tweet_post_date = lkDate + schedule_tweets_days_after_lk`
   2. Connect `MergePost` → `ComputeTweetDate`.

8. **LLM conversion (Claude via OpenRouter)**
   1. Add **OpenRouter Chat Model** node `Claude-4.5-Sonnet` and select `anthropic/claude-sonnet-4.5`.
   2. Add **Structured Output Parser** node with a schema example matching:
      - `thread_variation.tweets[]`
      - `standalone_variation`
      - `original_post_summary`
      - `recommended_format`
   3. Add **LangChain Chain LLM** node `ConvertPostIntoTweets`:
      - Paste the prompt (role/constraints/format).
      - Reference post content from `MergePost` with dual-shape fallback (`json.Content` vs `json.fields.Content`).
      - Attach `Claude-4.5-Sonnet` as language model input and the Structured Output Parser as output parser.
      - Enable retry on fail.

9. **Persist tweets and update LK post status**
   1. Add **Airtable Create** `CreateStandaloneTweet` (X Tweets) with:
      - Status `Pending`, Variation `Standalone`, Tweet Nb `0`, Publication Date from `ComputeTweetDate`.
   2. Add **Split Out** `SplitThreadVariations` splitting `output.thread_variation.tweets`.
   3. Add **Airtable Create** `CreateThreadTweet` for each split tweet with Variation `Thread`, Tweet Nb from `tweet_number`.
   4. Add **Airtable Update** `UpdatePostStatus` to set LK Posts status to `Converted` matched by `LK Post ID`.
   5. Add **Merge** `MergeTweets` and connect it back to `LoopOverPosts` to continue processing.

10. **Daily publishing pipeline**
   1. Add **Schedule Trigger** `Daily_AtNoon` set to daily at 12:30.
   2. Add **Airtable Search** `SearchApprovedTweets` (X Tweets) with formula:
      - `Status='Approved'`
      - `Publication Date` before `NOW()`
      - `Variation='Standalone' OR Tweet Nb=1`
   3. Add **Switch** `SwitchStatusAndVariation`:
      - Route Approved+Standalone to standalone poster
      - Route Approved+Thread to thread poster

11. **Manual publishing webhook**
   1. Add **Webhook** `Webhook_OnPostTweet` path `post-tweet`.
   2. In Airtable, create a button formula (as provided in note):
      - `"https://your-n8n-server-url/webhook/post-tweet?id=" & RECORD_ID()`
   3. Add **Airtable Get** `GetTweet` using `id = {{$json.query.id}}`.
   4. Connect `GetTweet` → `SwitchStatusAndVariation` (reuse same posting logic).

12. **Standalone posting**
   1. Add **Twitter/X** node `PostStandaloneTweet` (Create Tweet) with `text = {{$json.Content}}`.
   2. Add **Airtable Update** `UpdateStandaloneTweetStatus` to set Status=Tweeted for that Airtable record.  
      *Recommended fix while rebuilding:* update using the Airtable record `id` from the tweet item itself (not from `query.id`), or merge original Airtable item back after posting.

13. **Thread posting**
   1. Add **Data Table** nodes using an n8n Data Table `temporary_variables`:
      - `InitTemporaryTweetId` upsert `{name:'tweet_id', value:''}`
      - `GetLastTweetId` get where `name='tweet_id'`
      - `UpdateTemporaryTweetId` upsert `{name:'tweet_id', value: <last posted tweet id>}`
   2. Add **Airtable Search** `SearchAllThreadTweets` to fetch all tweets of the thread sorted by `Tweet Nb`.
   3. Add **Split In Batches** `LoopOverTweets` to iterate.
   4. Add **IF** `IfEmpty` checking if stored `tweet_id` is empty:
      - If empty → post first tweet (`PostFirstThreadTweet`)
      - Else → post reply (`PostReplyTweet` with inReplyToStatusId from data table)
   5. Merge both posting branches (`Merge`) then update stored tweet_id and mark each Airtable tweet record as Tweeted (`UpdateThreadTweetStatus`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Airtable base/template suggestion and required tables/fields | https://airtable.com/appiS2JpMdnxAS5o4/shrXh9iXTnL9fX63L/tbl7B3JblukFTGrAo/viwgajzRGnydUDjnw |
| Airtable “TWEET NOW” button URL formula | `"https://your-n8n-server-url/webhook/post-tweet?id=" & RECORD_ID()` |
| Apify actor documentation for LinkedIn scraping | https://console.apify.com/actors/A3cAPGpwBEG8RJwse/information/latest/readme |
| Apify cost note | “Price: $2 per 1k posts” |
| Operational behavior summary | Weekly scrape+convert → Airtable Pending; Daily post Approved due; Manual webhook to post immediately |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.