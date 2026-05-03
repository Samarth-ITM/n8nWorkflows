Build a Reddit no-API weekly digest with ScrapeOps and Google Sheets

https://n8nworkflows.xyz/workflows/build-a-reddit-no-api-weekly-digest-with-scrapeops-and-google-sheets-13992


# Build a Reddit no-API weekly digest with ScrapeOps and Google Sheets

# 1. Workflow Overview

This workflow creates a weekly Reddit industry digest without using the Reddit API. It scrapes public subreddit listing pages through ScrapeOps, extracts post metadata, enriches posts with Reddit JSON post details, deduplicates against a Google Sheet, stores only new posts, then compiles a weekly digest and optionally emails it.

Typical use cases:
- Weekly monitoring of technical communities such as `selfhosted`, `devops`, `programming`, and `webdev`
- Building a content pipeline for newsletters, internal trend reports, or research tracking
- Persisting scraped community content into Google Sheets for later analysis

## 1.1 Trigger & Runtime Configuration

The workflow starts on a weekly schedule. It calculates the current week range, resets workflow-level static memory, and emits one item per subreddit with shared configuration such as timeframe, per-subreddit post limit, and Google Sheet ID.

## 1.2 Subreddit Listing Scraping

Each subreddit is processed one at a time via batching. For each subreddit, the workflow requests the “Top of Week” page from `old.reddit.com` through ScrapeOps Proxy and inserts a randomized 1–3 second delay.

## 1.3 Listing Parsing

The returned HTML is parsed in a Code node. The parser extracts metadata including post title, canonical Reddit URL, author, flair, score, comment count, timestamp, and a generated SHA-1 content hash.

## 1.4 Post Enrichment

For every parsed listing item, the workflow fetches the corresponding Reddit `.json` endpoint through ScrapeOps, extracts `selftext` and inferred post type, then merges that data back into the listing metadata and normalizes the final post fields.

## 1.5 Deduplication & Persistence

In parallel with the subreddit configuration stage, the workflow reads existing rows from the `posts` tab in Google Sheets. New scraped posts are compared against existing sheet content using both `content_hash` and normalized `post_url`. Only unseen posts are marked as new and appended to the `posts` sheet.

## 1.6 Weekly Digest Generation & Delivery

After batch processing completes, the workflow builds a digest from the in-memory collection of newly discovered posts. It derives lightweight topics from repeated words, selects top posts by score and comment count, writes a summary row into the `weekly_digest` sheet, and optionally emails the digest.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Trigger & Configuration

### Overview
This block launches the workflow weekly and prepares one execution item per target subreddit. It also initializes workflow static data used later for deduplication and digest generation.

### Nodes Involved
- Weekly Schedule Trigger
- Configure Subreddits & Week Range

### Node Details

#### Weekly Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point
- **Configuration choices:** Uses a basic interval rule. In this exported JSON the rule is minimal and represents a schedule-based trigger intended to run weekly.
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none
  - Output: `Configure Subreddits & Week Range`
- **Version-specific requirements:** Type version 1
- **Edge cases / failure types:**
  - Misconfigured schedule may cause it to run too often or not at all
  - Timezone interpretation depends on instance settings
- **Sub-workflow reference:** None

#### Configure Subreddits & Week Range
- **Type and role:** `n8n-nodes-base.code`; emits runtime configuration records
- **Configuration choices:**
  - Resets static global arrays:
    - `global.seen = []`
    - `global.newPosts = []`
  - Calculates:
    - `run_id` from current ISO timestamp
    - `run_date` as `YYYY-MM-DD`
    - `week_range` from Monday to Sunday in UTC
  - Defines subreddit list:
    - `selfhosted`
    - `devops`
    - `programming`
    - `webdev`
  - Sets fixed parameters per emitted item:
    - `sort = "top"`
    - `time_range = "week"`
    - `limit = 20`
    - `sheet_id = "1rKuVREV4pedie7uAbuEcLvghNrAEeAjIPUBE6cQPleI"`
- **Key expressions or variables used:**
  - Workflow static data via `$getWorkflowStaticData('global')`
  - UTC week calculations
- **Input and output connections:**
  - Input: `Weekly Schedule Trigger`
  - Outputs:
    - ` Split Subreddits Into Batches`
    - `Read Existing Posts from Sheet`
- **Version-specific requirements:** Code node type version 2
- **Edge cases / failure types:**
  - If static data is unavailable in a given runtime, fallback behavior relies on `globalThis`
  - Hardcoded subreddit list requires manual editing for changes
  - Hardcoded sheet ID may drift from the IDs configured in Google Sheets nodes
- **Sub-workflow reference:** None

---

## 2.2 Block: Scrape Subreddit Listings

### Overview
This block processes subreddits sequentially, fetches each subreddit’s weekly top page through ScrapeOps, and deliberately slows the request cadence to reduce scraping pressure.

### Nodes Involved
-  Split Subreddits Into Batches
- ScrapeOps: Fetch Subreddit Listing
-  Polite Delay (1–3s)

### Node Details

####  Split Subreddits Into Batches
- **Type and role:** `n8n-nodes-base.splitInBatches`; controls iteration over subreddit items
- **Configuration choices:**
  - `batchSize = 1`
  - Processes one subreddit at a time
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input:
    - `Configure Subreddits & Week Range`
    - loop-back from `Append New Posts to Sheet`
  - Outputs:
    - `ScrapeOps: Fetch Subreddit Listing`
    - `Build Weekly Digest`
- **Version-specific requirements:** Type version 2
- **Edge cases / failure types:**
  - Because `Build Weekly Digest` is connected to the second output, the digest runs when batching completes; if no new posts were collected, the digest may still execute with empty data
  - Loop behavior depends on n8n batch semantics; incorrect downstream termination can cause partial processing
- **Sub-workflow reference:** None

#### ScrapeOps: Fetch Subreddit Listing
- **Type and role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`; performs proxied HTTP fetch of subreddit HTML
- **Configuration choices:**
  - URL expression:
    - `https://old.reddit.com/r/{{$json.subreddit}}/top/?t=week`
  - Uses ScrapeOps account credential
  - No advanced options explicitly set
- **Key expressions or variables used:**
  - `$json.subreddit`
- **Input and output connections:**
  - Input: ` Split Subreddits Into Batches`
  - Output: ` Polite Delay (1–3s)`
- **Version-specific requirements:** ScrapeOps node type version 1; requires installed ScrapeOps n8n node package and valid API credentials
- **Edge cases / failure types:**
  - Invalid or missing ScrapeOps credential
  - Reddit returning blocked, challenge, or alternate HTML
  - Network timeout or proxy errors
  - If subreddit does not exist, parsing stage may return zero posts
- **Sub-workflow reference:** None

####  Polite Delay (1–3s)
- **Type and role:** `n8n-nodes-base.wait`; rate-control pause
- **Configuration choices:**
  - Wait duration in seconds
  - Randomized expression: `Math.floor(Math.random()*3)+1`
- **Key expressions or variables used:**
  - Dynamic wait duration expression
- **Input and output connections:**
  - Input: `ScrapeOps: Fetch Subreddit Listing`
  - Output: `Parse Listing HTML → Post Metadata`
- **Version-specific requirements:** Type version 1
- **Edge cases / failure types:**
  - Wait node resumes execution asynchronously; environment must support wait/resume properly
  - Very large runs can accumulate runtime overhead
- **Sub-workflow reference:** None

---

## 2.3 Block: Parse Post Metadata

### Overview
This block converts scraped subreddit listing HTML into structured post objects. It also generates fallback metadata, detects subreddit names, and computes stable hashes for deduplication.

### Nodes Involved
- Parse Listing HTML → Post Metadata

### Node Details

#### Parse Listing HTML → Post Metadata
- **Type and role:** `n8n-nodes-base.code`; HTML parser and record builder
- **Configuration choices:**
  - Reads HTML from `$json.data`, `$json.body`, or raw `$json`
  - Uses `limit` from input, defaulting to 20
  - Calculates fallback values:
    - `run_id`
    - `run_date`
    - `week_range`
  - Parses old Reddit listing blocks by splitting on `<div class="thing`
  - Extracts:
    - `post_id`
    - `post_url`
    - `post_title`
    - `post_text` from listing `data-selftext-html` if present
    - `author`
    - `created_utc`
    - `score`
    - `num_comments`
    - `flair`
    - `subreddit`
  - Normalizes URLs to `www.reddit.com`
  - Builds `content_hash` as SHA-1 of `subreddit + title + url`
  - Sets `is_new = true` initially
  - `alwaysOutputData = true`
- **Key expressions or variables used:**
  - `$json.data`
  - `$json.body`
  - `$json.limit`
  - `require('crypto')`
- **Input and output connections:**
  - Input: ` Polite Delay (1–3s)`
  - Outputs:
    - ` ScrapeOps: Fetch Post Details (JSON)`
    - `Merge Post Metadata + Text` (input 0)
- **Version-specific requirements:** Code node type version 2; runtime must permit `require('crypto')`
- **Edge cases / failure types:**
  - HTML structure changes on Reddit can break regex parsing
  - Encoded characters may not be fully normalized
  - If no posts are detected, downstream merge may receive empty data
  - If `old.reddit.com` changes attributes like `data-permalink` or `data-fullname`, extraction can fail silently
- **Sub-workflow reference:** None

---

## 2.4 Block: Post Enrichment & Finalization

### Overview
This block fetches each post’s JSON representation, extracts full text where available, merges enrichment data with listing data, and produces the final normalized post record.

### Nodes Involved
-  ScrapeOps: Fetch Post Details (JSON)
- Extract Selftext & Post Type
- Merge Post Metadata + Text
- Finalize & Normalize Post Fields

### Node Details

####  ScrapeOps: Fetch Post Details (JSON)
- **Type and role:** `@scrapeops/n8n-nodes-scrapeops.ScrapeOps`; fetches per-post Reddit JSON endpoint
- **Configuration choices:**
  - URL expression:
    - `($json.post_url || '').replace(/\?.*$/, '').replace(/\/$/, '') + '.json?raw_json=1'`
  - `returnType = "json"`
  - Uses ScrapeOps account credential
- **Key expressions or variables used:**
  - `$json.post_url`
- **Input and output connections:**
  - Input: `Parse Listing HTML → Post Metadata`
  - Output: `Extract Selftext & Post Type`
- **Version-specific requirements:** ScrapeOps node type version 1
- **Edge cases / failure types:**
  - Post URL may be malformed or blank
  - Some posts may return HTML instead of JSON due to blocking or redirects
  - Deleted or removed posts can yield incomplete data
- **Sub-workflow reference:** None

#### Extract Selftext & Post Type
- **Type and role:** `n8n-nodes-base.code`; robust JSON extractor for Reddit post details
- **Configuration choices:**
  - Searches for response text in `body`, `data`, `response`, or longest string field
  - Decodes common HTML entities
  - Detects and rejects HTML responses
  - Parses either array or object JSON payload
  - Extracts:
    - `post_text_extracted`
    - `post_type`
    - `post_title`
    - `post_id`
    - `post_url`
    - `subreddit`
    - `score`
    - `num_comments`
    - `author`
    - `created_utc`
  - Returns `extracted_ok: false` with diagnostic fields on failure
- **Key expressions or variables used:**
  - `$json`
  - nested Reddit JSON path `data.children[0].data`
- **Input and output connections:**
  - Input: ` ScrapeOps: Fetch Post Details (JSON)`
  - Output: `Merge Post Metadata + Text` (input 1)
- **Version-specific requirements:** Code node type version 2
- **Edge cases / failure types:**
  - Response body absent
  - JSON parse errors
  - HTML challenge pages
  - Alternate Reddit JSON structures not matching expected path
  - `selftext` legitimately empty for link, image, and many media posts
- **Sub-workflow reference:** None

#### Merge Post Metadata + Text
- **Type and role:** `n8n-nodes-base.merge`; combines listing metadata with enriched post data
- **Configuration choices:**
  - `mode = combine`
  - `combinationMode = mergeByPosition`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs:
    - Input 0: `Parse Listing HTML → Post Metadata`
    - Input 1: `Extract Selftext & Post Type`
  - Output: `Finalize & Normalize Post Fields`
- **Version-specific requirements:** Merge node type version 2
- **Edge cases / failure types:**
  - Merge-by-position assumes both branches emit items in exactly the same order and count
  - If the JSON-fetch branch drops or adds items, records can become misaligned
- **Sub-workflow reference:** None

#### Finalize & Normalize Post Fields
- **Type and role:** `n8n-nodes-base.code`; final field cleanup
- **Configuration choices:**
  - If `post_text_extracted` is non-empty, it overwrites `post_text`
  - Otherwise keeps existing `post_text`
  - Deletes `post_text_extracted`
- **Key expressions or variables used:**
  - `item.json.post_text_extracted`
- **Input and output connections:**
  - Input: `Merge Post Metadata + Text`
  - Output: ` Merge Scraped + Existing Posts`
- **Version-specific requirements:** Code node type version 2
- **Edge cases / failure types:**
  - If merge misalignment occurred earlier, the wrong text can be assigned to a record
- **Sub-workflow reference:** None

---

## 2.5 Block: Read Existing Posts, Deduplicate, and Save

### Overview
This block loads existing post history from Google Sheets, compares scraped posts against historical content, flags only unseen posts as new, and appends them to the `posts` worksheet.

### Nodes Involved
- Read Existing Posts from Sheet
-  Merge Scraped + Existing Posts
- Deduplicate New Posts
- Append New Posts to Sheet

### Node Details

#### Read Existing Posts from Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; loads existing saved posts
- **Configuration choices:**
  - Reads from spreadsheet ID `1rKuVREV4pedie7uAbuEcLvghNrAEeAjIPUBE6cQPleI`
  - Targets `gid=0`, cached as sheet name `posts`
  - `alwaysOutputData = true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Configure Subreddits & Week Range`
  - Output: ` Merge Scraped + Existing Posts` (input 1)
- **Version-specific requirements:** Google Sheets node type version 3; requires OAuth2 credentials
- **Edge cases / failure types:**
  - OAuth token expiration or missing scopes
  - Spreadsheet or sheet tab renamed/deleted
  - Large sheets may increase runtime
- **Sub-workflow reference:** None

####  Merge Scraped + Existing Posts
- **Type and role:** `n8n-nodes-base.merge`; synchronization point between scraped items and sheet-read branch
- **Configuration choices:**
  - `mode = combine`
  - `combinationMode = mergeByPosition`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs:
    - Input 0: `Finalize & Normalize Post Fields`
    - Input 1: `Read Existing Posts from Sheet`
  - Output: `Deduplicate New Posts`
- **Version-specific requirements:** Merge node type version 2
- **Edge cases / failure types:**
  - The deduplication code does not rely on merged row pairing; it separately calls `$items('Read Existing Posts from Sheet')`, so this merge mainly acts as a gating/join node
  - Merge-by-position here is not semantically ideal because counts will differ greatly between scraped posts and historical sheet rows
- **Sub-workflow reference:** None

#### Deduplicate New Posts
- **Type and role:** `n8n-nodes-base.code`; deduplicates against sheet data and workflow static cache
- **Configuration choices:**
  - Loads workflow static global store
  - Ensures `global.seen` and `global.newPosts` arrays exist
  - Normalizes URLs by replacing `old.reddit.com` with `www.reddit.com`
  - Reads all rows from `Read Existing Posts from Sheet` using `$items(...)`
  - Adds both `content_hash` and `post_url` to a lookup set
  - For each incoming scraped item:
    - marks `is_new = true` if unseen
    - pushes new items into `global.newPosts`
    - otherwise sets `is_new = false`
  - Returns all items regardless of newness
  - `alwaysOutputData = true`
- **Key expressions or variables used:**
  - `$getWorkflowStaticData('global')`
  - `$items('Read Existing Posts from Sheet')`
- **Input and output connections:**
  - Input: ` Merge Scraped + Existing Posts`
  - Output: `Append New Posts to Sheet`
- **Version-specific requirements:** Code node type version 2
- **Edge cases / failure types:**
  - Because all items are returned, the next append node will write duplicates too unless separately filtered; this is a major behavioral issue
  - Static global cache is reset at the start of each run, so cross-run memory comes mainly from Google Sheets, not static memory
  - Column fallback references like `r[15]`, `r.Q`, `r[6]`, `r.G` assume possible alternate formats and may not always map correctly
- **Sub-workflow reference:** None

#### Append New Posts to Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; appends scraped post rows to the `posts` tab
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet ID: `1rKuVREV4pedie7uAbuEcLvghNrAEeAjIPUBE6cQPleI`
  - Sheet name: `posts`
  - Explicit column mapping for:
    - `run_id`
    - `run_date`
    - `subreddit`
    - `sort`
    - `time_range`
    - `post_id`
    - `post_url`
    - `post_title`
    - `post_text`
    - `author`
    - `created_utc`
    - `score`
    - `num_comments`
    - `flair`
    - `extracted_at`
    - `content_hash`
    - `is_new`
- **Key expressions or variables used:**
  - Per-column `{{$json.field}}` expressions
- **Input and output connections:**
  - Input: `Deduplicate New Posts`
  - Output: ` Split Subreddits Into Batches` (loop-back)
- **Version-specific requirements:** Google Sheets node type version 4.5; requires OAuth2 credentials
- **Edge cases / failure types:**
  - As currently wired, this node appends all items passed from `Deduplicate New Posts`, including those with `is_new = false`
  - Sheet schema mismatch can cause blank cells or append errors
  - Numeric fields are not force-converted; Google Sheets may infer unexpected types
- **Sub-workflow reference:** None

---

## 2.6 Block: Build Weekly Digest and Send Email

### Overview
This block compiles the run’s newly discovered posts into a lightweight digest, stores the digest in Google Sheets, and optionally sends it by email.

### Nodes Involved
- Build Weekly Digest
- Append Weekly Digest to Sheet
- Send Weekly Digest Email

### Node Details

#### Build Weekly Digest
- **Type and role:** `n8n-nodes-base.code`; creates summary metrics, topic clusters, and a text digest
- **Configuration choices:**
  - Reads `global.newPosts`
  - Computes:
    - `total_posts`
    - combined `subreddits`
    - frequency-based topic words excluding common stopwords
    - up to 8 topic clusters, padded to at least 5 when possible
    - top 10 posts sorted by score then comments
  - Produces:
    - `top_topics_json`
    - `top_posts_json`
    - `weekly_brief_text`
    - `created_at`
    - `run_id`
    - `week_range`
- **Key expressions or variables used:**
  - `$getWorkflowStaticData('global')`
  - word tokenization using `/[^a-z0-9+#]+/`
- **Input and output connections:**
  - Input: ` Split Subreddits Into Batches` (completion path)
  - Output: `Append Weekly Digest to Sheet`
- **Version-specific requirements:** Code node type version 2
- **Edge cases / failure types:**
  - If `global.newPosts` is empty, it still generates a digest row with `total_posts = 0`
  - Topic extraction is simplistic and may produce weak topic labels
  - Stopword list is English-only and minimal
- **Sub-workflow reference:** None

#### Append Weekly Digest to Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; stores weekly digest output
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet ID: `1rKuVREV4pedie7uAbuEcLvghNrAEeAjIPUBE6cQPleI`
  - Sheet name: `weekly_digest`
  - Column mapping:
    - `run_id`
    - `created_at`
    - `subreddits`
    - `week_range`
    - `total_posts`
    - `top_posts_json`
    - `top_topics_json`
    - `weekly_brief_text`
- **Key expressions or variables used:**
  - Per-column `{{$json.field}}` expressions
- **Input and output connections:**
  - Input: `Build Weekly Digest`
  - Output: `Send Weekly Digest Email`
- **Version-specific requirements:** Google Sheets node type version 4.5
- **Edge cases / failure types:**
  - Missing `weekly_digest` tab causes failure
  - Large JSON strings may make sheet cells difficult to inspect
- **Sub-workflow reference:** None

#### Send Weekly Digest Email
- **Type and role:** `n8n-nodes-base.emailSend`; sends the digest via email
- **Configuration choices:**
  - Subject expression:
    - `Weekly Developer Tools Digest (Reddit) – {{$json.week_range}}`
    - The exported JSON shows mojibake in the dash character, which should be corrected manually
  - Email body: `{{$json.weekly_brief_text}}`
  - To: `user@example.com`
  - From: `you@example.com`
  - `executeOnce = true`
- **Key expressions or variables used:**
  - `$json.weekly_brief_text`
  - `$json.week_range`
- **Input and output connections:**
  - Input: `Append Weekly Digest to Sheet`
  - Output: none
- **Version-specific requirements:** Email Send node type version 2; requires SMTP or email transport configuration depending on n8n setup
- **Edge cases / failure types:**
  - Placeholder email addresses must be replaced
  - Missing email transport credentials causes failure
  - `executeOnce` means only one email is sent even if upstream emits multiple items
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Schedule Trigger | Schedule Trigger | Starts the workflow on a schedule |  | Configure Subreddits & Week Range | ## 1. Trigger & Configuration<br>Fires weekly and sets runtime config — subreddit list, week range, batch size, and Google Sheet IDs. |
| Configure Subreddits & Week Range | Code | Builds per-subreddit runtime items and resets static state | Weekly Schedule Trigger |  Split Subreddits Into Batches; Read Existing Posts from Sheet | ## 1. Trigger & Configuration<br>Fires weekly and sets runtime config — subreddit list, week range, batch size, and Google Sheet IDs. |
|  Split Subreddits Into Batches | Split In Batches | Iterates through subreddits one at a time | Configure Subreddits & Week Range; Append New Posts to Sheet | ScrapeOps: Fetch Subreddit Listing; Build Weekly Digest | ## 2. Scrape Subreddit Listings<br>Batch through each subreddit and scrape the "Top of Week" page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with a polite delay between requests. |
| ScrapeOps: Fetch Subreddit Listing | ScrapeOps | Fetches subreddit top-of-week HTML via proxy |  Split Subreddits Into Batches |  Polite Delay (1–3s) | ## 2. Scrape Subreddit Listings<br>Batch through each subreddit and scrape the "Top of Week" page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with a polite delay between requests. |
|  Polite Delay (1–3s) | Wait | Adds random delay between requests | ScrapeOps: Fetch Subreddit Listing | Parse Listing HTML → Post Metadata | ## 2. Scrape Subreddit Listings<br>Batch through each subreddit and scrape the "Top of Week" page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with a polite delay between requests. |
| Parse Listing HTML → Post Metadata | Code | Parses old Reddit listing HTML into structured posts |  Polite Delay (1–3s) |  ScrapeOps: Fetch Post Details (JSON); Merge Post Metadata + Text | ## 3. Parse Post Metadata<br>Extract title, URL, score, comment count, author, and timestamps from listing HTML into structured JSON. |
|  ScrapeOps: Fetch Post Details (JSON) | ScrapeOps | Fetches per-post Reddit JSON | Parse Listing HTML → Post Metadata | Extract Selftext & Post Type | ## 4. Enrich & Finalize Posts<br>Fetch each post as JSON to extract `selftext`, merge with listing metadata, and normalize all fields into the final record. |
| Extract Selftext & Post Type | Code | Extracts selftext and post characteristics from Reddit JSON |  ScrapeOps: Fetch Post Details (JSON) | Merge Post Metadata + Text | ## 4. Enrich & Finalize Posts<br>Fetch each post as JSON to extract `selftext`, merge with listing metadata, and normalize all fields into the final record. |
| Merge Post Metadata + Text | Merge | Merges listing metadata with post JSON extraction | Parse Listing HTML → Post Metadata; Extract Selftext & Post Type | Finalize & Normalize Post Fields | ## 4. Enrich & Finalize Posts<br>Fetch each post as JSON to extract `selftext`, merge with listing metadata, and normalize all fields into the final record. |
| Finalize & Normalize Post Fields | Code | Chooses best post text and cleans fields | Merge Post Metadata + Text |  Merge Scraped + Existing Posts | ## 4. Enrich & Finalize Posts<br>Fetch each post as JSON to extract `selftext`, merge with listing metadata, and normalize all fields into the final record. |
| Read Existing Posts from Sheet | Google Sheets | Loads existing saved posts for deduplication | Configure Subreddits & Week Range |  Merge Scraped + Existing Posts | ## 5. Deduplicate & Save<br>Compare against existing Sheet rows by hash and URL, then append only new posts to the `posts` tab. |
|  Merge Scraped + Existing Posts | Merge | Synchronizes scraped branch and sheet-read branch before deduplication | Finalize & Normalize Post Fields; Read Existing Posts from Sheet | Deduplicate New Posts | ## 5. Deduplicate & Save<br>Compare against existing Sheet rows by hash and URL, then append only new posts to the `posts` tab. |
| Deduplicate New Posts | Code | Flags duplicates using hash and URL and stores new posts in static memory |  Merge Scraped + Existing Posts | Append New Posts to Sheet | ## 5. Deduplicate & Save<br>Compare against existing Sheet rows by hash and URL, then append only new posts to the `posts` tab. |
| Append New Posts to Sheet | Google Sheets | Appends post rows to the `posts` sheet and loops batch execution | Deduplicate New Posts |  Split Subreddits Into Batches | ## 5. Deduplicate & Save<br>Compare against existing Sheet rows by hash and URL, then append only new posts to the `posts` tab. |
| Build Weekly Digest | Code | Builds digest summary from newly found posts |  Split Subreddits Into Batches | Append Weekly Digest to Sheet | ## 6. Weekly Digest & Email<br>Generate topic clusters and top post summaries, write to `weekly_digest` tab, and optionally send by email. |
| Append Weekly Digest to Sheet | Google Sheets | Stores weekly digest in sheet | Build Weekly Digest | Send Weekly Digest Email | ## 6. Weekly Digest & Email<br>Generate topic clusters and top post summaries, write to `weekly_digest` tab, and optionally send by email. |
| Send Weekly Digest Email | Email Send | Emails the final digest text | Append Weekly Digest to Sheet |  | ## 6. Weekly Digest & Email<br>Generate topic clusters and top post summaries, write to `weekly_digest` tab, and optionally send by email. |
| Overview (Sticky) | Sticky Note | Workspace documentation |  |  | # 📰 Reddit Industry Digest (Weekly) → Google Sheets<br>This workflow builds a weekly industry digest by collecting top posts from selected subreddits — no Reddit API needed. It scrapes public Reddit pages via **ScrapeOps Proxy**, enriches each post with full text using Reddit's JSON endpoint, deduplicates against your Google Sheet, and generates a weekly summary that can optionally be emailed.<br>### How it works<br>1. ⏰ **Weekly Schedule Trigger** fires automatically once a week.<br>2. ⚙️ **Configure Subreddits & Week Range** sets the subreddit list, week range, and Sheet IDs.<br>3. 📦 **Split Subreddits Into Batches** processes each subreddit one at a time.<br>4. 🌐 **ScrapeOps: Fetch Subreddit Listing** scrapes the top-of-week page from `old.reddit.com`.<br>5. ⏳ **Polite Delay** adds a 1–3s pause between requests.<br>6. 🔍 **Parse Listing HTML** extracts title, URL, score, comments, author, and timestamps.<br>7. 📡 **ScrapeOps: Fetch Post Details** retrieves each post as JSON to extract `selftext`.<br>8. 🔀 **Merge & Normalize** combines listing data with post body text into a final record.<br>9. 🧹 **Deduplicate New Posts** filters posts already in the Sheet by hash and URL.<br>10. 💾 **Append New Posts** saves only new posts to the `posts` tab.<br>11. 📊 **Build Weekly Digest** generates topic clusters and top post summaries.<br>12. 📧 **Send Digest Email** optionally emails the weekly summary.<br>### Setup steps<br>- Register for a free ScrapeOps API key: https://scrapeops.io/app/register/n8n<br>- Add ScrapeOps credentials in n8n. Docs: https://scrapeops.io/docs/n8n/overview/<br>- Duplicate [this sheet](https://docs.google.com/spreadsheets/d/1rKuVREV4pedie7uAbuEcLvghNrAEeAjIPUBE6cQPleI/edit?usp=sharing) to copy Columns and Spreadsheet ID.<br>- Connect Google Sheets and set your Spreadsheet ID in the Sheet nodes.<br>- Update your subreddit list in **Configure Subreddits & Week Range**.<br>- Optional: enable **Send Digest Email** and configure credentials.<br>### Customization<br>- Add or remove subreddits in the configure node.<br>- Change timeframe from `week` to `month` in the fetch URL.<br>- Add a Slack node to post the digest to a channel. |
| Section: Trigger & Inputs | Sticky Note | Visual section label |  |  | ## 1. Trigger & Configuration<br>Fires weekly and sets runtime config — subreddit list, week range, batch size, and Google Sheet IDs. |
| Section: Scrape Listings | Sticky Note | Visual section label |  |  | ## 2. Scrape Subreddit Listings<br>Batch through each subreddit and scrape the "Top of Week" page via [ScrapeOps Proxy](https://scrapeops.io/docs/n8n/proxy-api/) with a polite delay between requests. |
| Section: Post Enrichment | Sticky Note | Visual section label |  |  | ## 3. Parse Post Metadata<br>Extract title, URL, score, comment count, author, and timestamps from listing HTML into structured JSON. |
| Section: Post Enrichment1 | Sticky Note | Visual section label |  |  | ## 4. Enrich & Finalize Posts<br>Fetch each post as JSON to extract `selftext`, merge with listing metadata, and normalize all fields into the final record. |
| Section: Post Enrichment2 | Sticky Note | Visual section label |  |  | ## 5. Deduplicate & Save<br>Compare against existing Sheet rows by hash and URL, then append only new posts to the `posts` tab. |
| Section: Post Enrichment3 | Sticky Note | Visual section label |  |  | ## 6. Weekly Digest & Email<br>Generate topic clusters and top post summaries, write to `weekly_digest` tab, and optionally send by email. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Reddit Industry Digest with ScrapeOps and Google Sheets`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run weekly.
   - Choose the desired weekday and time in your n8n instance timezone.

3. **Add a Code node named `Configure Subreddits & Week Range`**
   - Connect it after the trigger.
   - Paste logic that:
     - resets workflow static global arrays `seen` and `newPosts`
     - computes:
       - `run_id`
       - `run_date`
       - Monday-to-Sunday `week_range`
     - defines a subreddit list such as:
       - `selfhosted`
       - `devops`
       - `programming`
       - `webdev`
     - emits one item per subreddit with:
       - `subreddit`
       - `sort = top`
       - `time_range = week`
       - `limit = 20`
       - `sheet_id = your spreadsheet ID`

4. **Add a Google Sheets credential**
   - Use OAuth2 for Google Sheets.
   - Ensure access to the destination spreadsheet.

5. **Prepare the spreadsheet**
   - Create or duplicate a spreadsheet with two tabs:
     - `posts`
     - `weekly_digest`
   - The `posts` tab should contain columns:
     - `run_id`
     - `run_date`
     - `subreddit`
     - `sort`
     - `time_range`
     - `post_id`
     - `post_url`
     - `post_title`
     - `post_text`
     - `author`
     - `created_utc`
     - `score`
     - `num_comments`
     - `flair`
     - `extracted_at`
     - `content_hash`
     - `is_new`
   - The `weekly_digest` tab should contain columns:
     - `run_id`
     - `week_range`
     - `subreddits`
     - `total_posts`
     - `top_topics_json`
     - `weekly_brief_text`
     - `top_posts_json`
     - `created_at`

6. **Add a Google Sheets node named `Read Existing Posts from Sheet`**
   - Connect it from `Configure Subreddits & Week Range`.
   - Configure it to read from your spreadsheet.
   - Select the `posts` tab.
   - Enable it to output data even if empty, if available in your node version.

7. **Add a `Split In Batches` node**
   - Name it ` Split Subreddits Into Batches`.
   - Connect it from `Configure Subreddits & Week Range`.
   - Set `Batch Size` to `1`.

8. **Install and configure ScrapeOps**
   - Install the ScrapeOps n8n node package if it is not already installed.
   - Create ScrapeOps credentials with your API key.
   - Reference:
     - https://scrapeops.io/app/register/n8n
     - https://scrapeops.io/docs/n8n/overview/

9. **Add a ScrapeOps node named `ScrapeOps: Fetch Subreddit Listing`**
   - Connect it from ` Split Subreddits Into Batches`.
   - Set URL to:
     - `https://old.reddit.com/r/{{$json.subreddit}}/top/?t=week`
   - Use the ScrapeOps credential.
   - Keep response as HTML/text.

10. **Add a Wait node named ` Polite Delay (1–3s)`**
    - Connect it after the listing fetch.
    - Set unit to `seconds`.
    - Set amount expression to:
      - `{{ Math.floor(Math.random()*3)+1 }}`

11. **Add a Code node named `Parse Listing HTML → Post Metadata`**
    - Connect it after the wait node.
    - Implement logic that:
      - reads listing HTML from `data` or `body`
      - parses each Reddit post block from `old.reddit.com`
      - extracts title, author, permalink, score, comments, flair, and timestamp
      - normalizes Reddit URLs to `https://www.reddit.com/...`
      - computes `content_hash` using SHA-1
      - emits one item per post
      - honors a `limit` from input, default `20`
    - Enable `Always Output Data`.

12. **Add a ScrapeOps node named ` ScrapeOps: Fetch Post Details (JSON)`**
    - Connect it from `Parse Listing HTML → Post Metadata`.
    - Set URL expression to:
      - `{{ ($json.post_url || '').replace(/\?.*$/, '').replace(/\/$/, '') + '.json?raw_json=1' }}`
    - Set return type to `json`.
    - Use the same ScrapeOps credential.

13. **Add a Code node named `Extract Selftext & Post Type`**
    - Connect it after the post-details node.
    - Implement logic that:
      - looks for the raw response in `body`, `data`, `response`, or the longest string field
      - decodes HTML entities
      - rejects HTML responses
      - parses JSON
      - extracts post data from `data.children[0].data`
      - emits fields including:
        - `post_text_extracted`
        - `post_type`
        - `post_title`
        - `post_id`
        - `post_url`
        - `subreddit`
        - `score`
        - `num_comments`
        - `author`
        - `created_utc`
      - returns diagnostic data on parse failure

14. **Add a Merge node named `Merge Post Metadata + Text`**
    - Connect input 0 from `Parse Listing HTML → Post Metadata`
    - Connect input 1 from `Extract Selftext & Post Type`
    - Set:
      - Mode: `Combine`
      - Combination mode: `Merge By Position`

15. **Add a Code node named `Finalize & Normalize Post Fields`**
    - Connect it after the merge.
    - Configure it to:
      - overwrite `post_text` with `post_text_extracted` when non-empty
      - otherwise keep the existing `post_text`
      - remove `post_text_extracted`

16. **Add a Merge node named ` Merge Scraped + Existing Posts`**
    - Connect input 0 from `Finalize & Normalize Post Fields`
    - Connect input 1 from `Read Existing Posts from Sheet`
    - Set:
      - Mode: `Combine`
      - Combination mode: `Merge By Position`
   - Note: this node mainly acts as a synchronization point.

17. **Add a Code node named `Deduplicate New Posts`**
    - Connect it after the merge.
    - Implement logic that:
      - loads workflow static global data
      - reads all rows from `Read Existing Posts from Sheet` with `$items(...)`
      - builds a set of existing `content_hash` and normalized `post_url`
      - checks each scraped item against that set
      - sets `is_new` true or false
      - pushes only new posts into `global.newPosts`
      - returns items for downstream use
    - Enable `Always Output Data`.

18. **Important correction: filter before appending**
    - The provided workflow claims to append only new posts, but as wired it returns all items to the append node.
    - To reproduce the intended behavior safely, add an `IF` node or a Code filter after `Deduplicate New Posts`:
      - condition: `{{$json.is_new}}` is true
    - Send only the true branch to the append node.
    - If reproducing the JSON exactly, omit this filter; if reproducing the intended logic, include it.

19. **Add a Google Sheets node named `Append New Posts to Sheet`**
    - Connect it from:
      - ideally the filtered `true` branch from step 18
      - or directly from `Deduplicate New Posts` if you want to mirror the provided wiring
    - Configure:
      - Operation: `Append`
      - Spreadsheet: your spreadsheet
      - Sheet: `posts`
      - Map the columns explicitly to the fields listed in step 5

20. **Loop batch execution**
    - Connect `Append New Posts to Sheet` back to ` Split Subreddits Into Batches`.
    - This continues processing the next subreddit.

21. **Add a Code node named `Build Weekly Digest`**
    - Connect it to the second output of ` Split Subreddits Into Batches`, which runs when batching completes.
    - Implement logic that:
      - reads `global.newPosts`
      - counts total new posts
      - creates a subreddit summary
      - tokenizes title + post text
      - excludes common stopwords
      - derives top keywords and simple topic clusters
      - sorts top posts by score, then comment count
      - creates:
        - `top_topics_json`
        - `top_posts_json`
        - `weekly_brief_text`
        - `created_at`
        - `run_id`
        - `week_range`

22. **Add a Google Sheets node named `Append Weekly Digest to Sheet`**
    - Connect it after `Build Weekly Digest`.
    - Configure:
      - Operation: `Append`
      - Sheet: `weekly_digest`
      - Explicitly map:
        - `run_id`
        - `created_at`
        - `subreddits`
        - `week_range`
        - `total_posts`
        - `top_posts_json`
        - `top_topics_json`
        - `weekly_brief_text`

23. **Add an Email Send node named `Send Weekly Digest Email`**
    - Connect it after `Append Weekly Digest to Sheet`.
    - Configure:
      - To: your recipient address
      - From: a valid sender address
      - Subject: `Weekly Developer Tools Digest (Reddit) – {{$json.week_range}}`
      - Text body: `{{$json.weekly_brief_text}}`
    - Enable `Execute Once`.

24. **Configure email credentials**
    - Depending on your n8n environment, configure SMTP or the supported email transport.
    - Replace placeholder addresses.

25. **Test with manual execution**
    - Run the workflow manually.
    - Verify:
      - subreddit pages are fetched
      - posts are parsed
      - per-post JSON is readable
      - `posts` tab receives rows
      - `weekly_digest` tab receives one digest row
      - email sends correctly if enabled

26. **Validate edge conditions**
    - Test with:
      - a nonexistent subreddit
      - an empty `posts` tab
      - a repeated run on the same week
      - one or more link/image posts with empty `selftext`

27. **Recommended hardening improvements**
    - Add a filter before `Append New Posts to Sheet` so only `is_new = true` rows are appended
    - Replace merge-by-position with a safer key-based join where practical
    - Add error handling for blocked HTML, bad JSON, and credential failures
    - Move hardcoded subreddit list and spreadsheet ID into environment variables or workflow variables

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Register for a free ScrapeOps API key | https://scrapeops.io/app/register/n8n |
| ScrapeOps n8n documentation | https://scrapeops.io/docs/n8n/overview/ |
| ScrapeOps Proxy API documentation | https://scrapeops.io/docs/n8n/proxy-api/ |
| Duplicate the sample Google Sheet template | https://docs.google.com/spreadsheets/d/1rKuVREV4pedie7uAbuEcLvghNrAEeAjIPUBE6cQPleI/edit?usp=sharing |
| Customization note: add or remove subreddits in the configuration Code node | Workflow setup note |
| Customization note: change timeframe from `week` to `month` in the listing fetch URL | Workflow setup note |
| Customization note: add a Slack node to send the digest to a channel | Workflow setup note |

## Additional implementation observations
- The workflow has a single entry point: `Weekly Schedule Trigger`.
- There are no sub-workflows or workflow-execution nodes in this workflow.
- The current implementation does **not fully enforce** “append only new posts” because `Deduplicate New Posts` returns all items and `Append New Posts to Sheet` receives them directly.
- The digest is based only on posts collected during the current run and stored in `global.newPosts`, not on all posts in the spreadsheet.
- The workflow depends on `old.reddit.com` HTML structure; if Reddit changes markup, the parser will need updates.