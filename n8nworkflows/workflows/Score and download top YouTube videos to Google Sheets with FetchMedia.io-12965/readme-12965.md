Score and download top YouTube videos to Google Sheets with FetchMedia.io

https://n8nworkflows.xyz/workflows/score-and-download-top-youtube-videos-to-google-sheets-with-fetchmedia-io-12965


# Score and download top YouTube videos to Google Sheets with FetchMedia.io

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Download and organize high-quality YouTube videos*  
**Title (user-provided):** *Score and download top YouTube videos to Google Sheets with FetchMedia.io*

### Purpose
This workflow discovers educational YouTube videos (short-form, high-engagement, recent), scores them for relevance, writes the top results to a Google Sheet, and optionally submits them to FetchMedia.io for asynchronous download. It then polls FetchMedia until completion (or retry limit) and updates the corresponding Google Sheet row with the download URL or failure info.

### Target use cases
- Curating a shortlist of high-quality YouTube resources for learning topics (here: AI agents, prompt engineering, AI tools).
- Building a “resource pipeline” that continuously finds, ranks, and optionally downloads videos.
- Maintaining a structured Google Sheets database (title, channel, metrics, URL, download status).

### Logical blocks
1. **1.1 Manual start & query generation** – define a list of search queries and iterate them.
2. **1.2 YouTube search & itemization** – run YouTube Data API search per query and split results into individual items.
3. **1.3 Relevance filtering & controlled batching** – filter titles via regex rules and process items in batches.
4. **1.4 Metadata enrichment & duration normalization** – fetch detailed video metadata and convert ISO-8601 duration to seconds.
5. **1.5 Quality filtering & normalization** – apply engagement/date/duration thresholds and shape data for downstream scoring.
6. **1.6 Deduplication, scoring, sorting, limiting** – remove duplicates, compute custom relevance score, sort, keep top N.
7. **1.7 Persist to Google Sheets** – append top results to a spreadsheet.
8. **1.8 FetchMedia async download & polling loop** – submit URLs to FetchMedia, poll status with a retry cap, update sheet row with results or failure.

---

## 2. Block-by-Block Analysis

### 2.1 Manual start & query generation

**Overview:** Initializes execution manually and sets a predefined array of search queries that will be processed one by one.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set Query
- Split search queries

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`manualTrigger`) to start the workflow on demand.
- **Configuration choices:** No parameters; uses n8n manual execution.
- **Inputs / outputs:** Entry node → outputs to **Set Query**.
- **Edge cases:** None (manual start only).

#### Node: Set Query
- **Type / role:** Set node (`set`) to create the query list.
- **Configuration choices:** Creates a single field:
  - `query` (type: array) with multiple predefined search strings (AI agents, prompt engineering, etc.).
- **Key expressions / variables:** Static array expression.
- **Connections:** Receives from trigger → outputs to **Split search queries**.
- **Edge cases:** If `query` is empty, downstream search will produce no results.

#### Node: Split search queries
- **Type / role:** Split Out (`splitOut`) to emit one item per query string.
- **Configuration choices:** `fieldToSplitOut = query`.
- **Connections:** From **Set Query** → to **Search YouTube**.
- **Edge cases:** If `query` is not an array, split fails or yields unexpected items.

---

### 2.2 YouTube search & itemization

**Overview:** Executes YouTube Data API v3 Search for each query and splits returned items into individual video candidates.

**Nodes involved:**
- Search YouTube
- Split YouTube Results

#### Node: Search YouTube
- **Type / role:** HTTP Request (`httpRequest`) calling YouTube Data API v3 `search`.
- **Configuration choices (interpreted):**
  - **Endpoint:** `GET https://www.googleapis.com/youtube/v3/search`
  - **Auth:** Predefined credential type `youTubeOAuth2Api` (OAuth2).
  - **Query params:**
    - `part=id,snippet`
    - `q={{ $json.query }}`
    - `type=video`
    - `maxResults=50`
    - `order=relevance`
    - `videoCategoryId=27` (Education)
    - `videoDuration=short` (YouTube’s “short” category; not necessarily Shorts)
- **Inputs / outputs:** Input item contains a single query string → output is YouTube API response with `items[]`.
- **Edge cases / failures:**
  - OAuth misconfiguration, missing YouTube Data API v3 enablement, quota errors (403), rate limits (429).
  - Some results may lack expected fields depending on API behavior.
- **Version notes:** HTTP Request v4.2; OAuth configured via node credentials.

#### Node: Split YouTube Results
- **Type / role:** Split Out (`splitOut`) to emit one item per search result.
- **Configuration choices:** `fieldToSplitOut = items`.
- **Connections:** From **Search YouTube** → to **Filter for Relevance**.
- **Edge cases:** If YouTube returns no `items`, produces no output.

---

### 2.3 Relevance filtering & controlled batching

**Overview:** Filters out spammy/monetization-oriented titles and keeps educational content. Then processes videos in batches to control API calls.

**Nodes involved:**
- Filter for Relevance
- Loop over YouTube Results

#### Node: Filter for Relevance
- **Type / role:** Filter (`filter`) to keep only items whose titles match educational patterns and exclude spam.
- **Configuration choices:**
  - Applies multiple regex checks against: `{{ $json.snippet.title.toLowerCase() }}`
  - **Must match** (regex): educational intent keywords such as `tutorial|course|guide|...|step.*by.*step|...`
  - **Must NOT match** monetization/spam phrases: `money|rich|profit|...`
  - **Must NOT match** automation/get-rich quick patterns: `autopilot|passive.*income|...`
  - **Must NOT match** clickbait patterns: `insane|crazy|ultimate.*secret|hack|...`
- **Connections:** From **Split YouTube Results** → to **Loop over YouTube Results**.
- **Edge cases:**
  - Missing `snippet.title` will cause expression evaluation errors.
  - Overly strict regex can filter out valid educational videos.
- **Version notes:** Filter v2.2 with strict type validation in options.

#### Node: Loop over YouTube Results
- **Type / role:** Split In Batches (`splitInBatches`) to process results in controlled chunks.
- **Configuration choices:**
  - `batchSize = 5`
  - `executeOnce = false` (normal batch looping)
  - Done branch intentionally unused (per sticky note).
- **Connections:**
  - Input from **Filter for Relevance**
  - Outputs to **Remove Duplicate Videos** and **Get Video Metadata** (two outgoing connections).
- **Important behavior note:** This node is also used as a loop-back target after metadata cleanup, which effectively continues batch processing while preserving item context.
- **Edge cases:**
  - If loop-back wiring is incorrect, it can create infinite loops or stop early.
  - Running two branches from the same batch can lead to surprising ordering; careful reliance on cross-node item references is required downstream.

---

### 2.4 Metadata enrichment & duration normalization

**Overview:** Retrieves full video metadata (including statistics and duration) and converts the ISO-8601 duration into numeric seconds so duration filtering is reliable.

**Nodes involved:**
- Get Video Metadata
- Add Duration Seconds

#### Node: Get Video Metadata
- **Type / role:** HTTP Request (`httpRequest`) calling YouTube Data API v3 `videos`.
- **Configuration choices:**
  - **Endpoint:** `GET https://www.googleapis.com/youtube/v3/videos`
  - **Auth:** `youTubeOAuth2Api`
  - **Query params:**
    - `part=snippet,statistics,contentDetails`
    - `id={{ $json.id.videoId }}`
- **Connections:** From **Loop over YouTube Results** → to **Add Duration Seconds**.
- **Edge cases / failures:**
  - If `id.videoId` is missing, request fails.
  - API can return empty `items[]` (deleted/private video), causing downstream code to default to `PT0S`.
  - Quota/rate limiting.
  
#### Node: Add Duration Seconds
- **Type / role:** Code node (`code`) to flatten the returned video object and compute `DurationSeconds`.
- **Configuration choices (logic):**
  - Reads `const video = $json.items?.[0]` from YouTube `videos` response.
  - Parses `video.contentDetails.duration` with regex `^PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?$`.
  - Returns a flattened object: `...video` plus `DurationSeconds`.
  - Defaults to `PT0S` if missing.
- **Connections:** From **Get Video Metadata** → to **Filter for Quality**.
- **Edge cases:**
  - If `items[0]` is undefined, it returns `{...undefined}` behavior (in JS spread, `...undefined` is ignored) but many expected fields may be missing later.
  - Duration formats outside the regex (rare) result in `m=null` and all 0 seconds.

---

### 2.5 Quality filtering & normalization

**Overview:** Keeps only recent, high-engagement, very short videos, then reshapes fields into a stable schema for scoring and storage.

**Nodes involved:**
- Filter for Quality
- Remove Unnecessary Fields

#### Node: Filter for Quality
- **Type / role:** Filter (`filter`) applying numeric/date thresholds.
- **Configuration choices (conditions):**
  - `statistics.viewCount > 10000`
  - `statistics.likeCount > 100`
  - `snippet.publishedAt after 2023-12-01T00:00:00`
  - `DurationSeconds < 61` (i.e., ≤ 60 seconds)
- **Connections:** From **Add Duration Seconds** → to **Remove Unnecessary Fields**.
- **Edge cases:**
  - Some videos can have likes hidden; `likeCount` may be missing → expression may evaluate as undefined and fail comparison depending on “loose validation”.
  - Published date parsing issues if field missing or malformed.
- **Version notes:** Filter v2.2 with “looseTypeValidation” enabled at the node level, reducing hard failures.

#### Node: Remove Unnecessary Fields
- **Type / role:** Set node (`set`) to project only needed fields and rename them.
- **Configuration choices (created fields):**
  - `Title = snippet.title`
  - `Channel = snippet.channelTitle`
  - `Published At = snippet.publishedAt`
  - `Views = statistics.viewCount`
  - `Likes = statistics.likeCount`
  - `Description = snippet.description`
  - `URL = https://www.youtube.com/watch?v={{ $json.id }}`
  - `Duration = DurationSeconds`
- **Connections:** From **Filter for Quality** → back to **Loop over YouTube Results** (loop continuation).
- **Important caveat:** The URL is built with `{{ $json.id }}`. After **Add Duration Seconds**, the flattened object’s `id` is usually the *videoId string* (from `items[0].id`), so this becomes correct. If the structure differs, URL may be malformed.
- **Edge cases:**
  - If `snippet` or `statistics` are missing (e.g., empty items), fields become undefined and later scoring may fail.

---

### 2.6 Deduplication, scoring, sorting, limiting

**Overview:** Removes duplicates (because multiple queries can return the same video), computes a custom relevance score, sorts descending, and keeps only the top N.

**Nodes involved:**
- Remove Duplicate Videos
- Generate Relevance Score
- Sort by Relevance (Our score)
- Take only the amount of top videos

#### Node: Remove Duplicate Videos
- **Type / role:** Remove Duplicates (`removeDuplicates`).
- **Configuration choices:**
  - Compare: `selectedFields`
  - Field to compare: `id.videoId`
- **Connections:** From **Loop over YouTube Results** → to **Generate Relevance Score**.
- **Critical edge case:** At this point, items flowing from **Loop over YouTube Results** may be either:
  - Search results (which have `id.videoId`), or
  - Flattened metadata objects (where `id` is a string, not `{ videoId }`).
  
  If the data is the flattened metadata path, `id.videoId` does not exist and deduplication may not work as intended (or treat all as unique/undefined). This workflow appears to rely on batch-loop behavior where the “search item structure” is still present in this path.
  
#### Node: Generate Relevance Score
- **Type / role:** Set node (`set`) computing ranking fields.
- **Configuration choices:**
  - Initializes fields for FetchMedia tracking: `DownloadURL`, `fetch_id`, `fetch_status`, `fetch_error` as empty.
  - Computes `video_id` using a cross-node reference:
    - `video_id = {{ $('Get Video Metadata').item.json.items[0].id }}`
  - Computes `RelevanceScore` with a weighted heuristic:
    - Adds points for educational keywords (tutorial/course/guide/step by step/etc.)
    - Adds points for AI-specific terms (prompt engineering, ai agent, ai tools, prompting)
    - Adds engagement bonuses for high view/like thresholds
    - Subtracts points for monetization indicators (“money”, “rich”, “earn”)
  - Re-maps fields into stable names (Title, Channel, Published at, Duration, Views, Likes, Description, URL).
- **Connections:** From **Remove Duplicate Videos** → to **Sort by Relevance (Our score)**.
- **Edge cases:**
  - Heavy use of `toLowerCase()` assumes `Title` is a string; missing Title can cause runtime errors.
  - `parseInt($json.Views)` and `parseInt($json.Likes)` assume numeric strings.
  - The cross-node lookup `$('Get Video Metadata').item...` can misalign if item indices diverge due to batching/branching; safer would be to carry `videoId` through the same item instead of referencing another node’s item.
  
#### Node: Sort by Relevance (Our score)
- **Type / role:** Sort (`sort`) to rank videos.
- **Configuration choices:** Sort by `RelevanceScore` descending.
- **Connections:** From **Generate Relevance Score** → to **Take only the amount of top videos**.
- **Edge cases:** Missing/undefined scores sort unpredictably (often last).

#### Node: Take only the amount of top videos
- **Type / role:** Limit (`limit`) to keep only top results.
- **Configuration choices:** `maxItems = 5`.
- **Connections:** From **Sort by Relevance** → to **Send to Google Sheets**.
- **Edge cases:** If fewer than 5 items, passes all.

---

### 2.7 Persist to Google Sheets

**Overview:** Appends the selected top videos into a Google Sheet with columns for scoring and download tracking.

**Nodes involved:**
- Send to Google Sheets

#### Node: Send to Google Sheets
- **Type / role:** Google Sheets node (`googleSheets`) append operation.
- **Configuration choices:**
  - Operation: **Append**
  - Document: “Eli_YouTube AI Resources” (ID provided in node)
  - Sheet: “YouTube AI Resources” (gid=0)
  - Mapping mode: Auto-map input data to columns.
  - Columns include: RelevanceScore, Title, Channel, Published at, Duration, Views, Likes, Description, URL, video_id, DownloadURL, fetch_id, fetch_status, fetch_error.
- **Connections:** From **Take only the amount of top videos** → to **Start Fetch (FetchMedia)**.
- **Edge cases / failures:**
  - OAuth issues, missing sheet permissions.
  - Column header mismatch: auto-mapping requires sheet headers to match input keys (case and spacing matter).
  - Appending duplicates: this workflow appends new rows every run; it does not check if the URL already exists.

---

### 2.8 FetchMedia async download & polling loop

**Overview:** Submits each selected YouTube URL to FetchMedia.io, then polls every 15 seconds until `success` or until attempt limit is reached (30 retries). Updates the same Google Sheet row with `download_url` or failure status.

**Nodes involved:**
- Start Fetch (FetchMedia) (HTTP Request)
- Init Polling State (Set)
- Poll loop (Wait → GET status → check → loop)
- Check Fetch Status (HTTP Request)
- Check if download is complete
- Increase attempt counter
- Check max attempts
- Search for the relevant video1
- Update the status and the download URL
- Search for the relevant video
- Update the status of the download

#### Node: Start Fetch (FetchMedia) (HTTP Request)
- **Type / role:** HTTP Request (`httpRequest`) to start an async fetch job.
- **Configuration choices:**
  - Method: `POST https://api.fetchmedia.io/v1/fetch`
  - Body: `url={{ $json.URL }}`
  - Header: `X-API-KEY={{ $env.FETCHMEDIA_API_KEY }}`
- **Connections:** From **Send to Google Sheets** → to **Init Polling State (Set)**.
- **Edge cases / failures:**
  - Missing env var `FETCHMEDIA_API_KEY` → 401/403 from FetchMedia.
  - FetchMedia API downtime/timeouts.
  - Invalid YouTube URL formatting upstream → job may fail.
- **Requirement:** Workflow expects `FETCHMEDIA_API_KEY` defined in n8n environment variables.

#### Node: Init Polling State (Set)
- **Type / role:** Set node initializes polling fields for each item.
- **Configuration choices:**
  - `fetch_id = {{ $json.fetch_id }}` (returned from FetchMedia start)
  - `attempt = 0`
  - `video_id = {{ $('Take only the amount of top videos').item.json.video_id }}`
  - `fetch_status = pending`
- **Connections:** → to **Poll loop**.
- **Edge cases:**
  - If FetchMedia does not return `fetch_id`, polling URL becomes invalid.
  - Cross-node reference to `Take only...` can misalign items if concurrency differs; ideally use `$json.video_id` carried forward.

#### Node: Poll loop (Wait → GET status → check → loop)
- **Type / role:** Wait node (`wait`) used as a timed polling delay.
- **Configuration choices:** Wait `amount = 15` (seconds).
- **Connections:** Wait → **Check Fetch Status**.
- **Edge cases:** Long-running polling increases execution time; may hit n8n execution timeout limits depending on hosting plan.

#### Node: Check Fetch Status (HTTP Request)
- **Type / role:** HTTP Request (`httpRequest`) to check job status.
- **Configuration choices:**
  - `GET https://api.fetchmedia.io/v1/fetch/{{ $('Init Polling State (Set)').item.json.fetch_id }}`
  - Headers: `X-API-KEY={{ $env.FETCHMEDIA_API_KEY }}`, plus content-type/accept.
- **Connections:** → **Check if download is complete**.
- **Edge cases:**
  - If fetch_id mismatched, returns 404 or error.
  - API rate limits if many videos and 15s polling.

#### Node: Check if download is complete
- **Type / role:** IF node (`if`) checks completion.
- **Configuration choices:** Condition `{{ $json.status }} == "success"`.
- **Connections:**
  - **True (success):** → **Search for the relevant video1**
  - **False (not success):** → **Increase attempt counter**
- **Edge cases:**
  - If API returns `failed` status, this branch still goes to retry loop until max attempts is hit (no early stop on `failed`).
  - If `status` missing, treated as not equal success → retries.

#### Node: Search for the relevant video1
- **Type / role:** Google Sheets lookup (`googleSheets`) to find the row to update.
- **Configuration choices:**
  - Filter: lookup `URL` equals `{{ $json.url }}`
- **Connections:** → **Update the status and the download URL**
- **Edge case / bug risk:** The FetchMedia status payload uses `url`, but the value must match the sheet’s `URL` exactly. If FetchMedia normalizes URLs or returns a different format, lookup may fail.

#### Node: Update the status and the download URL
- **Type / role:** Google Sheets update (`googleSheets`) to write results back.
- **Configuration choices:**
  - Matching column: `URL`
  - Writes:
    - `URL = {{ $json.URL }}` (note: uses `$json.URL`, but the previous node likely outputs sheet rows, not the FetchMedia payload)
    - `fetch_id = {{ $('Check if download is complete').item.json.id }}`
    - `DownloadURL = {{ $('Check if download is complete').item.json.download_url }}`
    - `fetch_error = {{ $('Check if download is complete').item.json.error }}`
    - `fetch_status = {{ $('Check if download is complete').item.json.status }}`
- **Connections:** End of “success update” path.
- **Edge cases / potential mismatch:**
  - This update relies on cross-node references to FetchMedia response while operating on the Google Sheets row item. This is valid in n8n but can break if multiple items are processed and indices diverge.
  - The mapping sets `URL` from `$json.URL` which might not exist on the sheet row output; safer would be to keep URL from the lookup result or use `{{ $json.URL ?? $json.url }}` depending on shape.

#### Node: Increase attempt counter
- **Type / role:** Code node increments attempt.
- **Configuration:** `attempt: ($json.attempt ?? 0) + 1`
- **Connections:** → **Check max attempts**
- **Edge cases:** If attempt becomes `NaN` due to non-number input, the IF check may fail.

#### Node: Check max attempts
- **Type / role:** IF node to cap retries.
- **Configuration choices:** Continue polling if `attempt < 31` (i.e., max 30 retries).
- **Connections:**
  - **True:** → back to **Poll loop**
  - **False:** → **Search for the relevant video** (failure handling)
- **Edge cases:** Retry cadence = 15 seconds * 30 ≈ 7.5 minutes per video (worst case), multiplied by number of items.

#### Node: Search for the relevant video
- **Type / role:** Google Sheets lookup for failure update.
- **Configuration:** Filter `URL = {{ $json.url }}`
- **Connections:** → **Update the status of the download**
- **Edge cases:** Same URL normalization risk as the “success” lookup.

#### Node: Update the status of the download
- **Type / role:** Google Sheets update to record failure.
- **Configuration choices:**
  - Matching column: `URL`
  - Writes `DownloadURL = "Issue with the download :-( "`
  - (Also includes `video_id` mapping to `{{ $json.url }}` which appears inconsistent with column semantics.)
- **Edge cases / bug risk:**
  - The configured value sets `video_id` to a URL (`$json.url`) rather than the actual YouTube ID; consider correcting to preserve schema.
  - Does not set `fetch_status="failed"` explicitly; could be improved for clarity.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set Query |  |
| Set Query | Set | Define array of search queries | When clicking ‘Execute workflow’ | Split search queries | ### Search YouTube + split results |
| Split search queries | Split Out | Emit one item per query | Set Query | Search YouTube | ### Search YouTube + split results |
| Search YouTube | HTTP Request | Call YouTube Search API (v3) | Split search queries | Split YouTube Results | ### YouTube Discovery & Initial Filtering |
| Split YouTube Results | Split Out | Emit one item per search result | Search YouTube | Filter for Relevance | ### YouTube Discovery & Initial Filtering |
| Filter for Relevance | Filter | Title keyword include/exclude filtering | Split YouTube Results | Loop over YouTube Results | ### Filter for relevance (title keywords / exclude spam) |
| Loop over YouTube Results | Split In Batches | Batch processing and loop control | Filter for Relevance; Remove Unnecessary Fields | Remove Duplicate Videos; Get Video Metadata | ## Batch Processing Note |
| Get Video Metadata | HTTP Request | Fetch full metadata (stats, duration) | Loop over YouTube Results | Add Duration Seconds | ### Fetch video metadata + convert duration to seconds |
| Add Duration Seconds | Code | Parse ISO-8601 duration to seconds; flatten payload | Get Video Metadata | Filter for Quality | ### Duration & Quality Validation |
| Filter for Quality | Filter | Apply thresholds (views/likes/date/duration) | Add Duration Seconds | Remove Unnecessary Fields | ### Quality filter (views/likes/date/duration) |
| Remove Unnecessary Fields | Set | Normalize fields for scoring/storage | Filter for Quality | Loop over YouTube Results | ### Duration & Quality Validation |
| Remove Duplicate Videos | Remove Duplicates | Deduplicate video candidates | Loop over YouTube Results | Generate Relevance Score | ### Deduplication & Relevance Scoring |
| Generate Relevance Score | Set | Compute custom score and tracking fields | Remove Duplicate Videos | Sort by Relevance (Our score) | ### Score + sort + limit top results |
| Sort by Relevance (Our score) | Sort | Rank by score descending | Generate Relevance Score | Take only the amount of top videos | ### Score + sort + limit top results |
| Take only the amount of top videos | Limit | Keep top N videos | Sort by Relevance (Our score) | Send to Google Sheets | ### Score + sort + limit top results |
| Send to Google Sheets | Google Sheets | Append rows to sheet | Take only the amount of top videos | Start Fetch (FetchMedia) (HTTP Request) | ### Write to Sheets + download + poll status + update row |
| Start Fetch (FetchMedia) (HTTP Request) | HTTP Request | Submit URL to FetchMedia async fetch | Send to Google Sheets | Init Polling State (Set) | ## Asynchronous Download & Status Polling (FetchMedia)<br>https://docs.fetchmedia.io/api-introduction |
| Init Polling State (Set) | Set | Initialize fetch_id + attempt counter | Start Fetch (FetchMedia) (HTTP Request) | Poll loop (Wait → GET status → check → loop) | ## Asynchronous Download & Status Polling (FetchMedia)<br>https://docs.fetchmedia.io/api-introduction |
| Poll loop (Wait → GET status → check → loop) | Wait | Delay between polling calls | Init Polling State (Set); Check max attempts | Check Fetch Status (HTTP Request) | ## Asynchronous Download & Status Polling (FetchMedia)<br>https://docs.fetchmedia.io/api-introduction |
| Check Fetch Status (HTTP Request) | HTTP Request | Get FetchMedia job status | Poll loop (Wait → GET status → check → loop) | Check if download is complete | ## Asynchronous Download & Status Polling (FetchMedia)<br>https://docs.fetchmedia.io/api-introduction |
| Check if download is complete | IF | Branch on status == success | Check Fetch Status (HTTP Request) | Search for the relevant video1; Increase attempt counter | ## Asynchronous Download & Status Polling (FetchMedia)<br>https://docs.fetchmedia.io/api-introduction |
| Search for the relevant video1 | Google Sheets | Lookup row for success update | Check if download is complete | Update the status and the download URL | ### Write to Sheets + download + poll status + update row |
| Update the status and the download URL | Google Sheets | Update row with download_url and status | Search for the relevant video1 | — | ### Write to Sheets + download + poll status + update row |
| Increase attempt counter | Code | Increment retry attempt | Check if download is complete | Check max attempts | ### If the FetchMedia service is unresponsive, we limit the number of retry attempts to 30.<br>After reaching this threshold, the process will stop and the corresponding row in the Google Sheet will be updated with a “Failed” status. |
| Check max attempts | IF | Stop after 30 retries | Increase attempt counter | Poll loop (Wait → GET status → check → loop); Search for the relevant video | ### If the FetchMedia service is unresponsive, we limit the number of retry attempts to 30.<br>After reaching this threshold, the process will stop and the corresponding row in the Google Sheet will be updated with a “Failed” status. |
| Search for the relevant video | Google Sheets | Lookup row for failure update | Check max attempts | Update the status of the download | ### Write to Sheets + download + poll status + update row |
| Update the status of the download | Google Sheets | Update row with failure message | Search for the relevant video | — | ### Write to Sheets + download + poll status + update row |
| Sticky Note | Sticky Note | Comment | — | — | ### YouTube Discovery & Initial Filtering |
| Sticky Note1 | Sticky Note | Comment | — | — | ### Duration & Quality Validation |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Batch Processing Note |
| Sticky Note3 | Sticky Note | Comment | — | — | ### Deduplication & Relevance Scoring |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Asynchronous Download & Status Polling (FetchMedia)<br>https://docs.fetchmedia.io/api-introduction |
| Sticky Note5 | Sticky Note | Comment | — | — | ## How it works (full block text in Notes section below) |
| Sticky Note6 | Sticky Note | Comment | — | — | ### Search YouTube + split results |
| Sticky Note7 | Sticky Note | Comment | — | — | ### Filter for relevance (title keywords / exclude spam) |
| Sticky Note8 | Sticky Note | Comment | — | — | ### Fetch video metadata + convert duration to seconds |
| Sticky Note9 | Sticky Note | Comment | — | — | ### Quality filter (views/likes/date/duration) |
| Sticky Note10 | Sticky Note | Comment | — | — | ### Score + sort + limit top results |
| Sticky Note11 | Sticky Note | Comment | — | — | ### Write to Sheets + download + poll status + update row |
| Sticky Note12 | Sticky Note | Comment | — | — | ### If the FetchMedia service is unresponsive, we limit the number of retry attempts to 30.<br>After reaching this threshold, the process will stop and the corresponding row in the Google Sheet will be updated with a “Failed” status. |

> Note: Sticky note nodes are included above to satisfy “do not skip any nodes”. They do not connect to the execution graph.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (set *Execution Order* to **v1** if you want identical behavior).
2. **Add node: Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`.

3. **Add node: Set**
   - Name: `Set Query`
   - Add field `query` (type **Array**) containing your search strings (e.g., the 6 shown in the workflow).
   - Connect: Manual Trigger → Set Query.

4. **Add node: Split Out**
   - Name: `Split search queries`
   - Field to split out: `query`
   - Connect: Set Query → Split search queries.

5. **Add node: HTTP Request**
   - Name: `Search YouTube`
   - Method: **GET**
   - URL: `https://www.googleapis.com/youtube/v3/search`
   - Authentication: **Predefined credential type**
   - Credential type: **YouTube OAuth2 API**
     - In Google Cloud: enable **YouTube Data API v3**
     - In n8n: create YouTube OAuth2 credentials and select them here
   - Send query parameters:
     - `part` = `id,snippet`
     - `q` = `{{$json.query}}`
     - `type` = `video`
     - `maxResults` = `50`
     - `order` = `relevance`
     - `videoCategoryId` = `27`
     - `videoDuration` = `short`
   - Connect: Split search queries → Search YouTube.

6. **Add node: Split Out**
   - Name: `Split YouTube Results`
   - Field to split out: `items`
   - Connect: Search YouTube → Split YouTube Results.

7. **Add node: Filter**
   - Name: `Filter for Relevance`
   - Create conditions (AND):
     - `snippet.title.toLowerCase()` **matches regex**: `tutorial|course|guide|training|learn|education|explained|masterclass|complete.*guide|step.*by.*step|beginners.*guide|how.*to.*build|fundamentals`
     - and **does not match regex** monetization: `money|rich|profit|earn|cash|income|\$|💰|sell|agency|business|client|freelance|side.*hustle|make.*money`
     - and **does not match regex** automation: `autopilot|passive.*income|[\d]+k.*subs|faceless.*videos|youtube.*automation.*money|100%.*automated.*money`
     - and **does not match regex** clickbait: `insane|crazy|ultimate.*secret|hack|trick|must.*do|steal.*these|best.*way.*to.*make|ways.*to.*make|you.*need.*to|[\d]+.*ways|[\d]+.*insane`
   - Connect: Split YouTube Results → Filter for Relevance.

8. **Add node: Split In Batches**
   - Name: `Loop over YouTube Results`
   - Batch size: `5`
   - Connect: Filter for Relevance → Loop over YouTube Results.

9. **Add node: HTTP Request**
   - Name: `Get Video Metadata`
   - Method: **GET**
   - URL: `https://www.googleapis.com/youtube/v3/videos`
   - Auth: same **YouTube OAuth2** credential
   - Query params:
     - `part` = `snippet,statistics,contentDetails`
     - `id` = `{{$json.id.videoId}}`
   - Connect: Loop over YouTube Results → Get Video Metadata.

10. **Add node: Code**
    - Name: `Add Duration Seconds`
    - Mode: “Run once for each item”
    - Paste logic to parse ISO duration and return flattened object with `DurationSeconds` (as in workflow).
    - Connect: Get Video Metadata → Add Duration Seconds.

11. **Add node: Filter**
    - Name: `Filter for Quality`
    - Conditions (AND):
      - `statistics.viewCount` > `10000`
      - `statistics.likeCount` > `100`
      - `snippet.publishedAt` after `2023-12-01T00:00:00`
      - `DurationSeconds` < `61`
    - Connect: Add Duration Seconds → Filter for Quality.

12. **Add node: Set**
    - Name: `Remove Unnecessary Fields`
    - Map fields:
      - Title, Channel, Published At, Views, Likes, Description
      - URL = `https://www.youtube.com/watch?v={{$json.id}}`
      - Duration = `{{$json.DurationSeconds}}`
    - Connect: Filter for Quality → Remove Unnecessary Fields.
    - Connect: Remove Unnecessary Fields → Loop over YouTube Results (to continue batch loop as designed).

13. **Add node: Remove Duplicates**
    - Name: `Remove Duplicate Videos`
    - Compare: “Selected fields”
    - Field to compare: `id.videoId`
    - Connect: Loop over YouTube Results → Remove Duplicate Videos.

14. **Add node: Set**
    - Name: `Generate Relevance Score`
    - Add fields:
      - `video_id` (cross-node reference to metadata result, or preferably carry it in-item)
      - `DownloadURL` empty
      - `RelevanceScore` expression (weighted scoring)
      - Keep normalized fields (Title, Channel, Published at, Duration, Views, Likes, Description, URL)
      - `fetch_id`, `fetch_status`, `fetch_error` empty
    - Connect: Remove Duplicate Videos → Generate Relevance Score.

15. **Add node: Sort**
    - Name: `Sort by Relevance (Our score)`
    - Sort field: `RelevanceScore` descending
    - Connect: Generate Relevance Score → Sort.

16. **Add node: Limit**
    - Name: `Take only the amount of top videos`
    - Max items: `5`
    - Connect: Sort → Limit.

17. **Add node: Google Sheets**
    - Name: `Send to Google Sheets`
    - Operation: **Append**
    - Set Google Sheets OAuth2 credentials (Google account with access).
    - Select Spreadsheet and Sheet tab.
    - Use auto-map and ensure the sheet has headers matching your keys.
    - Connect: Limit → Send to Google Sheets.

18. **(Optional download pipeline) Add node: HTTP Request**
    - Name: `Start Fetch (FetchMedia) (HTTP Request)`
    - Method: **POST**
    - URL: `https://api.fetchmedia.io/v1/fetch`
    - Body param: `url={{$json.URL}}`
    - Header: `X-API-KEY={{$env.FETCHMEDIA_API_KEY}}`
    - Ensure environment variable `FETCHMEDIA_API_KEY` is configured in your n8n runtime.
    - Connect: Send to Google Sheets → Start Fetch.

19. **Add node: Set**
    - Name: `Init Polling State (Set)`
    - Set:
      - `fetch_id={{$json.fetch_id}}`
      - `attempt=0`
      - `fetch_status=pending`
      - (Optionally carry URL/video_id for easier updates later)
    - Connect: Start Fetch → Init Polling State.

20. **Add node: Wait**
    - Name: `Poll loop (Wait → GET status → check → loop)`
    - Wait: 15 seconds
    - Connect: Init Polling State → Wait.

21. **Add node: HTTP Request**
    - Name: `Check Fetch Status (HTTP Request)`
    - Method: **GET**
    - URL: `https://api.fetchmedia.io/v1/fetch/{{ $('Init Polling State (Set)').item.json.fetch_id }}`
    - Header: `X-API-KEY={{$env.FETCHMEDIA_API_KEY}}`
    - Connect: Wait → Check Fetch Status.

22. **Add node: IF**
    - Name: `Check if download is complete`
    - Condition: `{{$json.status}} equals "success"`
    - Connect: Check Fetch Status → IF.

23. **Success branch: update Google Sheets**
    - Add **Google Sheets lookup** node `Search for the relevant video1`
      - Filter: `URL` equals `{{$json.url}}`
    - Add **Google Sheets update** node `Update the status and the download URL`
      - Match column: `URL`
      - Set `DownloadURL`, `fetch_id`, `fetch_status`, `fetch_error` from the FetchMedia payload.
    - Connect: IF (true) → lookup → update.

24. **Not-success branch: retry with cap**
    - Add **Code** node `Increase attempt counter` to increment `attempt`.
    - Add **IF** node `Check max attempts` with condition `attempt < 31`.
    - Connect: IF (false) → Increase attempt counter → Check max attempts.
    - Connect `Check max attempts`:
      - True → back to **Wait** node (poll again)
      - False → failure handling nodes below.

25. **Failure handling: update Google Sheets**
    - Add **Google Sheets lookup** node `Search for the relevant video`
      - Filter: `URL` equals `{{$json.url}}`
    - Add **Google Sheets update** node `Update the status of the download`
      - Match column: `URL`
      - Set `DownloadURL` to a failure message and optionally set `fetch_status="failed"`.
    - Connect: Check max attempts (false) → lookup → update.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow expects FETCHMEDIA_API_KEY to be set as an environment variable.” | FetchMedia integration requirement |
| FetchMedia API introduction | https://docs.fetchmedia.io/api-introduction |
| Sticky note: “The Done branch of this loop is intentionally unused. All processing occurs inside the batch loop to preserve item context and enable controlled retries.” | Batch loop design note |
| Sticky note (How it works + setup steps): searches YouTube with predefined queries, filters, enriches with metadata/duration seconds, scores/sorts/limits, appends to Google Sheets, optionally downloads with FetchMedia and updates rows by identifier. | Embedded project notes within workflow |

